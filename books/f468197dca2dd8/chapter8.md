---
title: "BottomLevelASのビルド"
---

前章でBottomLevelASのビルド準備が終了したので、いよいよ実際にビルドしていきます。

大まかな流れはこのようになります。

1. スクラッチバッファを作成
2. アクセラレーション構造をビルドする

# スクラッチバッファを作成

`createBottomLevelAS()`の続きを記述します。ビルドには一時的にバッファが必要となります。このバッファをスクラッチバッファと呼びますが、まずはこれを用意してあげなければいけません。いつものように新しい関数を作ります。

```cpp
void createBottomLevelAS()
{
    ...

    Buffer scratchBuffer = createScratchBuffer(buildSizesInfo.buildScratchSize);
}

Buffer createScratchBuffer(vk::DeviceSize size)
{
}
```

ここからは`createScratchBuffer()`を埋めていきます。ハンドル作成、メモリ確保、バインドはいつもと同様です。

```cpp
Buffer scratchBuffer;

scratchBuffer.handle = device->createBufferUnique(
    vk::BufferCreateInfo{}
    .setSize(size)
    .setUsage(vk::BufferUsageFlagBits::eStorageBuffer | vk::BufferUsageFlagBits::eShaderDeviceAddress)
);

auto memoryRequirements = device->getBufferMemoryRequirements(scratchBuffer.handle.get());
vk::MemoryAllocateFlagsInfo memoryAllocateFlagsInfo{ vk::MemoryAllocateFlagBits::eDeviceAddress };

scratchBuffer.deviceMemory = device->allocateMemoryUnique(
    vk::MemoryAllocateInfo{}
    .setAllocationSize(memoryRequirements.size)
    .setMemoryTypeIndex(vkutils::getMemoryType(memoryRequirements, vk::MemoryPropertyFlagBits::eDeviceLocal))
    .setPNext(&memoryAllocateFlagsInfo)
);
device->bindBufferMemory(scratchBuffer.handle.get(), scratchBuffer.deviceMemory.get(), 0);
```

最後にデバイスアドレスを取得します。

```cpp
vk::BufferDeviceAddressInfoKHR bufferDeviceAddressInfo{ scratchBuffer.handle.get() };
scratchBuffer.deviceAddress = getBufferDeviceAddress(scratchBuffer.handle.get());

return scratchBuffer;
```

# アクセラレーション構造をビルドする

`createScratchBuffer()`が完成したので、再度`createBottomLevelAS()`に戻りましょう。

ビルドに必要な情報を作成します。
```cpp
vk::AccelerationStructureBuildGeometryInfoKHR accelerationBuildGeometryInfo{};
accelerationBuildGeometryInfo
    .setType(vk::AccelerationStructureTypeKHR::eBottomLevel)
    .setFlags(vk::BuildAccelerationStructureFlagBitsKHR::ePreferFastTrace)
    .setMode(vk::BuildAccelerationStructureModeKHR::eBuild)
    .setDstAccelerationStructure(blas.handle.get())
    .setGeometries(geometry)
    .setScratchData(scratchBuffer.deviceAddress);

vk::AccelerationStructureBuildRangeInfoKHR accelerationStructureBuildRangeInfo{};
accelerationStructureBuildRangeInfo
    .setPrimitiveCount(1)
    .setPrimitiveOffset(0)
    .setFirstVertex(0)
    .setTransformOffset(0);
```

これで実際にデバイス上でビルドすることができます。コマンドバッファを作成してビルドコマンドを積み、サブミットします。
```cpp
auto commandBuffer = vkutils::createCommandBuffer(device.get(), commandPool.get(), true);
commandBuffer->buildAccelerationStructuresKHR(accelerationBuildGeometryInfo, &accelerationStructureBuildRangeInfo);
vkutils::submitCommandBuffer(device.get(), commandBuffer.get(), graphicsQueue);
```

最後にハンドルを取得しておきます。
```cpp
blas.buffer.deviceAddress = device->getAccelerationStructureAddressKHR({ blas.handle.get() });
```

ようやくボトムレベルアクセラレーション構造がビルドできました。次の章ではトップレベルアクセラレーション構造をビルドします。

[ここまでのC++コード(05_build_bottom_level_as.cpp)](https://github.com/nishidate-yuki/vulkan_raytracing_from_scratch/blob/master/code/05_build_bottom_level_as.cpp)
