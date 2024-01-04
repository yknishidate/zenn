---
title: "バッファの作成"
---

GPUでデータを扱うためにはバッファが必要になります。例えば、頂点バッファやインデックスバッファ、アクセラレーション構造のバッファなどです。この章では、これらで利用するバッファ構造体を作成していきます。

# Buffer構造体を作成

Buffer構造体を追加します。パフォーマンスの観点からは推奨されませんが、コードを簡単にするために、メモリはバッファごとに確保することにします。また、バッファのアクセスに利用するデバイスアドレスもメンバ変数として持たせます。

```cpp
struct Buffer {
    vk::UniqueBuffer buffer;
    vk::UniqueDeviceMemory memory;
    vk::DeviceAddress address{};
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
        vk::BufferCreateInfo createInfo{};
        createInfo.setSize(size);
        createInfo.setUsage(usage);
        buffer = device.createBufferUnique(createInfo);
    }
};
```

次にメモリを確保します。デバイスアドレスが必要なバッファでは`vk::MemoryAllocateFlagBits::eDeviceAddress`を立てます。

```cpp
void init(...) {
    // ...

    // Allocate memory
    vk::MemoryAllocateFlagsInfo allocateFlags{};
    if (usage & vk::BufferUsageFlagBits::eShaderDeviceAddress) {
        allocateFlags.flags = vk::MemoryAllocateFlagBits::eDeviceAddress;
    }

    vk::MemoryRequirements memoryReq =
        device.getBufferMemoryRequirements(*buffer);
    uint32_t memoryType = vkutils::getMemoryType(physicalDevice,  //
                                                 memoryReq, memoryProperty);
    vk::MemoryAllocateInfo allocateInfo{};
    allocateInfo.setAllocationSize(memoryReq.size);
    allocateInfo.setMemoryTypeIndex(memoryType);
    allocateInfo.setPNext(&allocateFlags);
    memory = device.allocateMemoryUnique(allocateInfo);
}
```

バッファハンドルとメモリをバインドします。

```cpp
void init(...) {
    // ...

    // Bind buffer to memory
    device.bindBufferMemory(*buffer, *memory, 0);
}
```

格納したいデータのポインタが渡されている場合は、メモリにコピーします。
    
```cpp
void init(...) {
    // ...

    // Copy data
    if (data) {
        void* mappedPtr = device.mapMemory(*memory, 0, size);
        memcpy(mappedPtr, data, size);
        device.unmapMemory(*memory);
    }
}
```

最後に、デバイスアドレスが必要な場合は取得しておきます。

```cpp
void init(...) {
    // ...

    // Get address
    if (usage & vk::BufferUsageFlagBits::eShaderDeviceAddress) {
        vk::BufferDeviceAddressInfoKHR addressInfo{};
        addressInfo.setBuffer(*buffer);
        address = device.getBufferAddressKHR(&addressInfo);
    }
}
```

以上でバッファの初期化は完了です。

[ここまでのC++コード(03_create_buffer.cpp)](https://github.com/nishidate-yuki/vulkan_raytracing_from_scratch/blob/master/code/03_create_buffer.cpp)
