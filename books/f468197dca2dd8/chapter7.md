
前章でバーテックスバッファを作成できたので、ここからボトムレベルアクセラレーション構造を作成していきます。

大まかな流れはこのようになります。

1. 構造体を定義
2. バッファをジオメトリにセット
3. アクセラレーション構造のバッファを作成
4. アクセラレーション構造を作成

# 構造体を定義

アクセラレーション構造を表す構造体を定義します。

```cpp
struct AccelerationStructure
{
    vk::UniqueAccelerationStructureKHR handle;
    Buffer buffer;
};
```

メンバ変数にも追加しましょう。
```cpp
private:
    ...
    AccelerationStructure blas;
```

# バッファをジオメトリにセット

`createBottomLevelAS()`に戻り、続きを記述していきます。

ジオメトリを作成します。ここではバーテックスバッファとインデックスバッファを組み合わせ、三角形データとしてジオメトリを作成しています。

```cpp
vk::AccelerationStructureGeometryTrianglesDataKHR triangleData{};
triangleData
    .setVertexFormat(vk::Format::eR32G32B32Sfloat)
    .setVertexData(vertexBuffer.deviceAddress)
    .setVertexStride(sizeof(Vertex))
    .setMaxVertex(vertices.size())
    .setIndexType(vk::IndexType::eUint32)
    .setIndexData(indexBuffer.deviceAddress);

vk::AccelerationStructureGeometryKHR geometry{};
geometry
    .setGeometryType(vk::GeometryTypeKHR::eTriangles)
    .setGeometry({ triangleData })
    .setFlags(vk::GeometryFlagBitsKHR::eOpaque);
```

# アクセラレーション構造のバッファを作成

アクセラレーション構造のビルドに必要なサイズを取得します。

```cpp
vk::AccelerationStructureBuildGeometryInfoKHR buildGeometryInfo{};
buildGeometryInfo
    .setType(vk::AccelerationStructureTypeKHR::eBottomLevel)
    .setFlags(vk::BuildAccelerationStructureFlagBitsKHR::ePreferFastTrace)
    .setGeometries(geometry);

const uint32_t primitiveCount = 1;
auto buildSizesInfo = device->getAccelerationStructureBuildSizesKHR(
    vk::AccelerationStructureBuildTypeKHR::eDevice, buildGeometryInfo, primitiveCount);
```

ASを保持するためのバッファを作成していきますが、これは新しい関数として分割します。

```cpp
void createBottomLevelAS()
{
    ...
    
    blas.buffer = createAccelerationStructureBuffer(buildSizesInfo);
}

Buffer createAccelerationStructureBuffer(vk::AccelerationStructureBuildSizesInfoKHR buildSizesInfo)
{

}
```

この関数もいつもの流れで

- ハンドル作成
- メモリ確保
- バインド

を行います。

まずはハンドル作成です。
```cpp
Buffer buffer{};
buffer.handle = device->createBufferUnique(
    vk::BufferCreateInfo{}
    .setSize(buildSizesInfo.accelerationStructureSize)
    .setUsage(vk::BufferUsageFlagBits::eAccelerationStructureStorageKHR
        | vk::BufferUsageFlagBits::eShaderDeviceAddress)
);
```

次にメモリを確保します。通常のバッファと異なり、`vk::MemoryAllocateFlagsInfo`を`vk::MemoryAllocateInfo`の`pNext`に設定していることに注意してください。

```cpp
auto memoryRequirements = device->getBufferMemoryRequirements(buffer.handle.get());
vk::MemoryAllocateFlagsInfo memoryAllocateFlagsInfo{ vk::MemoryAllocateFlagBits::eDeviceAddress };

buffer.deviceMemory = device->allocateMemoryUnique(
    vk::MemoryAllocateInfo{}
    .setAllocationSize(memoryRequirements.size)
    .setMemoryTypeIndex(vkutils::getMemoryType(
        memoryRequirements, vk::MemoryPropertyFlagBits::eHostVisible | vk::MemoryPropertyFlagBits::eHostCoherent))
    .setPNext(&memoryAllocateFlagsInfo)
);
```

バインドすれば終了です。
```cpp
device->bindBufferMemory(buffer.handle.get(), buffer.deviceMemory.get(), 0);

return buffer;
```

# アクセラレーション構造を作成

`createAccelerationStructureBuffer()`が完成したので、`createBottomLevelAS()`に戻りましょう。

アクセラレーション構造の作成を行います。

```cpp
blas.handle = device->createAccelerationStructureKHRUnique(
    vk::AccelerationStructureCreateInfoKHR{}
    .setBuffer(blas.buffer.handle.get())
    .setSize(buildSizesInfo.accelerationStructureSize)
    .setType(vk::AccelerationStructureTypeKHR::eBottomLevel)
);
```

これでボトムレベルアクセラレーション構造をビルドする準備が全て整いました。次の章ではいよいよビルドを行います。

[ここまでのC++コード(04_create_bottom_level_as.cpp)](https://github.com/nishidate-yuki/vulkan_raytracing_from_scratch/blob/master/code/04_create_bottom_level_as.cpp)

