---
title: "シェーダバインディングテーブルの作成"
---

この章ではシェーダバインディングテーブルを作成していきます。

# シェーダバインディングテーブルについて

レイトレーシングではパイプライン内でGPUが適切なシェーダを選択して実行する必要があるため、事前に全てのシェーダのアドレスを渡しておく必要があります。シェーダバインディングテーブル（SBT）は、シェーダのアドレスを格納するためのバッファで、CPUでいう関数ポインタのテーブルのような役割を果たします。

# 必要なメンバー変数を追加

シェーダはRaygen、Miss、Hitに分かれます。Raygenは必ず1つですが、MissとHitには複数のシェーダが存在する可能性があります。そのため、SBTには各タイプのシェーダがいくつ存在するかを指定する必要があるため、`address`、`stride`、`size`の3つ情報を持つ`vk::StridedDeviceAddressRegionKHR`という構造体を使います。

```cpp
// ...
Buffer sbt{};
vk::StridedDeviceAddressRegionKHR raygenRegion{};
vk::StridedDeviceAddressRegionKHR missRegion{};
vk::StridedDeviceAddressRegionKHR hitRegion{};
```

# アライメントからRegionのストライドとサイズを計算

まず`createShaderBindingTable()`を追加します。

```cpp
void initVulkan() {
    // ...
    createShaderBindingTable();
}

void createShaderBindingTable() {
}
```

SBTはアライメント要件が決まっています。それをもとに、`raygenRegion`、`missRegion`、`hitRegion`のストライドとサイズを計算します。これは仕様に合わせる定型コードなので、あまり深く考えなくても大丈夫です。詳細が知りたい方は[NVIDIA Vulkan Ray Tracing Tutorial](https://nvpro-samples.github.io/vk_raytracing_tutorial_KHR/)を参照してください。

![](https://nvpro-samples.github.io/vk_raytracing_tutorial_KHR/Images/sbt_0.png)
*SBTのアライメント（NVIDIA Vulkan Ray Tracing Tutorial[https://nvpro-samples.github.io/vk_raytracing_tutorial_KHR/]から引用）*

```cpp
void createShaderBindingTable() {
    // Get RT props
    vk::PhysicalDeviceRayTracingPipelinePropertiesKHR rtProperties =
        vkutils::getRayTracingProps(physicalDevice);
    uint32_t handleSize = rtProperties.shaderGroupHandleSize;
    uint32_t handleAlignment = rtProperties.shaderGroupHandleAlignment;
    uint32_t baseAlignment = rtProperties.shaderGroupBaseAlignment;
    uint32_t handleSizeAligned =
        vkutils::alignUp(handleSize, handleAlignment);

    // Set strides and sizes
    uint32_t raygenShaderCount = 1;  // raygen count must be 1
    uint32_t missShaderCount = 1;
    uint32_t hitShaderCount = 1;

    raygenRegion.setStride(
        vkutils::alignUp(handleSizeAligned, baseAlignment));
    raygenRegion.setSize(raygenRegion.stride);

    missRegion.setStride(handleSizeAligned);
    missRegion.setSize(vkutils::alignUp(missShaderCount * handleSizeAligned,
                                        baseAlignment));

    hitRegion.setStride(handleSizeAligned);
    hitRegion.setSize(vkutils::alignUp(hitShaderCount * handleSizeAligned,
                                        baseAlignment));
}
```

SBTのバッファを作成します。SBT用のバッファは`vk::BufferUsageFlagBits::eShaderBindingTableKHR`を立てて作成します。

```cpp
void createShaderBindingTable() {
    // ...

    // Create SBT
    vk::DeviceSize sbtSize =
        raygenRegion.size + missRegion.size + hitRegion.size;
    sbt.init(physicalDevice, *device, sbtSize,
                vk::BufferUsageFlagBits::eShaderBindingTableKHR |
                    vk::BufferUsageFlagBits::eTransferSrc |
                    vk::BufferUsageFlagBits::eShaderDeviceAddress,
                vk::MemoryPropertyFlagBits::eHostVisible |
                    vk::MemoryPropertyFlagBits::eHostCoherent);
}
```

SBTバッファにシェーダハンドルを詰めていきます。まず、必要な数のシェーダハンドルを`vk::Device::getRayTracingShaderGroupHandlesKHR()`で取得します。ここで取得されるシェーダハンドルを実際に見てみると、何かしらのデータが3つ詰まっていることが分かります。

```cpp
void createShaderBindingTable() {
    // ...

    // Get shader group handles
    uint32_t handleCount =
        raygenShaderCount + missShaderCount + hitShaderCount;
    uint32_t handleStorageSize = handleCount * handleSize;
    std::vector<uint8_t> handleStorage(handleStorageSize);
    auto result = device->getRayTracingShaderGroupHandlesKHR(
        *pipeline, 0, handleCount, handleStorageSize, handleStorage.data());
    if (result != vk::Result::eSuccess) {
        std::cerr << "Failed to get ray tracing shader group handles.\n";
        std::abort();
    }
}
```

```
09 00 00 00 00 00 7c 03 ff ff 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0b 00 00 00 00 00 bc 03 ff ff 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0c 00 00 00 00 00 dc 03 ff ff 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
```

取得したハンドルは密に詰まっているため、SBTバッファに全体をコピーするとアライメントが合いません。そのため、アライメント要件に合うようにハンドルをコピーしていきますが、ここも定型コードという感じ。

```cpp
void createShaderBindingTable() {
    // ...

    // Copy handles
    uint8_t* sbtHead =
        static_cast<uint8_t*>(device->mapMemory(*sbt.memory, 0, sbtSize));

    uint8_t* dstPtr = sbtHead;
    auto copyHandle = [&](uint32_t index) {
        std::memcpy(dstPtr, handleStorage.data() + handleSize * index,
                    handleSize);
    };

    // Raygen
    uint32_t handleIndex = 0;
    copyHandle(handleIndex++);

    // Miss
    dstPtr = sbtHead + raygenRegion.size;
    for (uint32_t c = 0; c < missShaderCount; c++) {
        copyHandle(handleIndex++);
        dstPtr += missRegion.stride;
    }

    // Hit
    dstPtr = sbtHead + raygenRegion.size + missRegion.size;
    for (uint32_t c = 0; c < hitShaderCount; c++) {
        copyHandle(handleIndex++);
        dstPtr += hitRegion.stride;
    }
}
```

最後にRegionのデバイスアドレスを設定します。

```cpp
void createShaderBindingTable() {
    // ...

    // Copy handles
    uint8_t* sbtHead =
        static_cast<uint8_t*>(device->mapMemory(*sbt.memory, 0, sbtSize));

    uint8_t* dstPtr = sbtHead;
    auto copyHandle = [&](uint32_t index) {
        std::memcpy(dstPtr, handleStorage.data() + handleSize * index,
                    handleSize);
    };

    // Raygen
    uint32_t handleIndex = 0;
    copyHandle(handleIndex++);

    // Miss
    dstPtr = sbtHead + raygenRegion.size;
    for (uint32_t c = 0; c < missShaderCount; c++) {
        copyHandle(handleIndex++);
        dstPtr += missRegion.stride;
    }

    // Hit
    dstPtr = sbtHead + raygenRegion.size + missRegion.size;
    for (uint32_t c = 0; c < hitShaderCount; c++) {
        copyHandle(handleIndex++);
        dstPtr += hitRegion.stride;
    }

    raygenRegion.setDeviceAddress(sbt.address);
    missRegion.setDeviceAddress(sbt.address + raygenRegion.size);
    hitRegion.setDeviceAddress(sbt.address + raygenRegion.size +
                                missRegion.size);
}
```

これでSBTが完成しました。

[ここまでのC++コード(09_create_sbt.hpp)](https://github.com/nishidate-yuki/vulkan_raytracing_from_scratch/blob/master/code/09_create_sbt.hpp)

[シェーダファイル](https://github.com/nishidate-yuki/vulkan_raytracing_from_scratch/tree/master/shaders)
