---
title: "バッファの作成"
---

GPUでデータを扱うためにはバッファが必要になります。例えば、頂点バッファやインデックスバッファ、アクセラレーション構造のバッファなどです。この章では、これらで利用するバッファを作成していきます。

# Buffer構造体を作成

Buffer構造体を追加します。パフォーマンスの観点からは推奨されませんが、コードを簡単にするために、メモリはバッファごとに確保することにします。また、レイトレーシングではバッファのアクセスにデバイスアドレスを使うことが多いため、メンバ変数として持たせます。

```cpp
struct Buffer {
    vk::UniqueBuffer buffer;
    vk::UniqueDeviceMemory deviceMemory;
    uint64_t deviceAddress;
};
```

# 初期化関数を作成

バッファを作成するための関数を作成します。

`Buffer::init()`の流れは

- ハンドル作成
- メモリ確保
- ハンドルとメモリをバインド
- (必要があれば) データをメモリにコピー
- (必要があれば) デバイスアドレスを取得

となっています。

まずはハンドルを作成します。

```cpp
struct Buffer
{
    // ...
    
    void init(vk::PhysicalDevice physicalDevice,
              vk::Device device,
              vk::DeviceSize size,
              vk::BufferUsageFlags usage,
              vk::MemoryPropertyFlags memoryProperty,
              const void* data = nullptr) {
        // Create buffer
        vk::BufferCreateInfo bufferCreateInfo{};
        bufferCreateInfo.setSize(size);
        bufferCreateInfo.setUsage(usage);
        bufferCreateInfo.setQueueFamilyIndexCount(0);
        buffer = device.createBufferUnique(bufferCreateInfo);
    }
};
```

次にメモリを確保します。デバイスアドレスが必要なバッファでは`vk::MemoryAllocateFlagBits::eDeviceAddress`を有効化します。

```cpp
void init(...) {
    // ...

    // Allocate memory
    auto memoryRequirements = device.getBufferMemoryRequirements(*buffer);
    vk::MemoryAllocateFlagsInfo memoryFlagsInfo{};
    if (usage & vk::BufferUsageFlagBits::eShaderDeviceAddress) {
        memoryFlagsInfo.flags = vk::MemoryAllocateFlagBits::eDeviceAddress;
    }

    vk::MemoryAllocateInfo allocateInfo{};
    allocateInfo.setAllocationSize(memoryRequirements.size);
    allocateInfo.setMemoryTypeIndex(vkutils::getMemoryType(
        physicalDevice, memoryRequirements, memoryProperty));
    allocateInfo.setPNext(&memoryFlagsInfo);
    deviceMemory = device.allocateMemoryUnique(allocateInfo);
}
```

バッファハンドルとメモリをバインドします。

```cpp
void init(...) {
    // ...

    // Bind buffer to memory
    device.bindBufferMemory(*buffer, *deviceMemory, 0);
}
```

次に、格納したいデータのポインタが渡されていれば、バッファにコピーします。
    
```cpp
void init(...) {
    // ...

    // Copy data
    if (data) {
        void* dataPtr = device.mapMemory(*deviceMemory, 0, size);
        memcpy(dataPtr, data, size);
        device.unmapMemory(*deviceMemory);
    }
}
```

最後に、デバイスアドレスを取得しておきます。

```cpp
void init(...) {
    // ...

    // Get address
    if (usage & vk::BufferUsageFlagBits::eShaderDeviceAddress) {
        vk::BufferDeviceAddressInfoKHR bufferDeviceAI{*buffer};
        deviceAddress = device.getBufferAddressKHR(&bufferDeviceAI);
    }
}
```

以上でバッファの初期化は完了です。

[ここまでのC++コード(03_create_buffer.cpp)](https://github.com/nishidate-yuki/vulkan_raytracing_from_scratch/blob/master/code/03_create_buffer.cpp)
