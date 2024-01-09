---
title: "シェーダバインディングテーブルの作成"
---

この章ではシェーダバインディングテーブルを作成していきます。

# シェーダバインディングテーブルについて

TODO: 説明追加

```cpp
// ...
Buffer sbt{};
vk::StridedDeviceAddressRegionKHR raygenRegion{};
vk::StridedDeviceAddressRegionKHR missRegion{};
vk::StridedDeviceAddressRegionKHR hitRegion{};
```

```cpp
void initVulkan() {
    // ...
    createShaderBindingTable();
}

void createShaderBindingTable() {
}
```

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

    // Create SBT
    vk::DeviceSize sbtSize =
        raygenRegion.size + missRegion.size + hitRegion.size;
    sbt.init(physicalDevice, *device, sbtSize,
                vk::BufferUsageFlagBits::eShaderBindingTableKHR |
                    vk::BufferUsageFlagBits::eTransferSrc |
                    vk::BufferUsageFlagBits::eShaderDeviceAddress,
                vk::MemoryPropertyFlagBits::eHostVisible |
                    vk::MemoryPropertyFlagBits::eHostCoherent);

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

    // Copy handles
    uint32_t handleIndex = 0;
    uint8_t* sbtHead =
        static_cast<uint8_t*>(device->mapMemory(*sbt.memory, 0, sbtSize));

    uint8_t* dstPtr = sbtHead;
    auto copyHandle = [&](uint32_t index) {
        std::memcpy(dstPtr, handleStorage.data() + handleSize * index,
                    handleSize);
    };

    // Raygen
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

[ここまでのC++コード(09_create_sbt.hpp)](https://github.com/nishidate-yuki/vulkan_raytracing_from_scratch/blob/master/code/09_create_sbt.hpp)

[シェーダファイル](https://github.com/nishidate-yuki/vulkan_raytracing_from_scratch/tree/master/shaders)
