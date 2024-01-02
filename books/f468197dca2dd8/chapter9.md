この章ではトップレベルアクセラレーション構造を作成していきます。ボトムレベルの方とほとんど同じ流れです。既にいくつか便利な関数が作成されているので、比較的さくっと作成できるはずです。

大まかな流れはこのようになります。

1. インスタンスを作成
2. インスタンスのバッファを作成
3. ジオメトリにインスタンスをセット
4. トップレベルアクセラレーション構造のバッファを作成
5. トップレベルアクセラレーション構造を作成
6. スクラッチバッファを作成
7. トップレベルアクセラレーション構造をビルド


# インスタンスを作成

まずはメンバ変数にトップレベルアクセラレーション構造を追加します。
```cpp
AccelerationStructure tlas;
```

トップレベルアクセラレーション構造を作成する関数も追加し、`initVulkan()`で呼び出します。

```cpp
void initVulkan()
{
    ...
    createTopLevelAS();
}

void createTopLevelAS()
{
}
```

ここから関数の中身を記述します。

まずはトップレベルアクセラレーション構造に格納するインスタンスを作成しましょう。本来は複数のインスタンスを格納できますが、今回は1つだけにします。

インスタンスは主に

- ボトムレベルアクセラレーション構造への参照
- 変換行列

を持ちます。

```cpp
VkTransformMatrixKHR transformMatrix = {
    1.0f, 0.0f, 0.0f, 0.0f,
    0.0f, 1.0f, 0.0f, 0.0f,
    0.0f, 0.0f, 1.0f, 0.0f };
    
vk::AccelerationStructureInstanceKHR accelerationStructureInstance{};
accelerationStructureInstance
    .setTransform(transformMatrix)
    .setInstanceCustomIndex(0)
    .setMask(0xFF)
    .setInstanceShaderBindingTableRecordOffset(0)
    .setFlags(vk::GeometryInstanceFlagBitsKHR::eTriangleFacingCullDisable)
    .setAccelerationStructureReference(blas.buffer.deviceAddress);
```

# インスタンスのバッファを作成

次にバッファを作成します。このあたりはボトムレベルでいうバーテックスバッファの確保にあたります。

```cpp
Buffer instancesBuffer = createBuffer(
    sizeof(vk::AccelerationStructureInstanceKHR),
    vk::BufferUsageFlagBits::eAccelerationStructureBuildInputReadOnlyKHR
    | vk::BufferUsageFlagBits::eShaderDeviceAddress,
    vk::MemoryPropertyFlagBits::eHostVisible 
    | vk::MemoryPropertyFlagBits::eHostCoherent,
    &accelerationStructureInstance);
```

# ジオメトリにインスタンスをセット

ジオメトリの設定を行います。ちなみにジオメトリという単語は抽象的に使われていて、インスタンス、トライアングル、バウンディングボックスの総称です。

```cpp
vk::AccelerationStructureGeometryInstancesDataKHR instancesData{};
instancesData
    .setArrayOfPointers(false)
    .setData(instancesBuffer.deviceAddress);

vk::AccelerationStructureGeometryKHR geometry{};
geometry
    .setGeometryType(vk::GeometryTypeKHR::eInstances)
    .setGeometry({ instancesData })
    .setFlags(vk::GeometryFlagBitsKHR::eOpaque);
```

# トップレベルアクセラレーション構造のバッファを作成

ビルドに必要なサイズを取得します。`type`が`TopLevel`に変わっています。
```cpp
vk::AccelerationStructureBuildGeometryInfoKHR buildGeometryInfo{};
buildGeometryInfo
    .setType(vk::AccelerationStructureTypeKHR::eTopLevel)
    .setFlags(vk::BuildAccelerationStructureFlagBitsKHR::ePreferFastTrace)
    .setGeometries(geometry);

const uint32_t primitiveCount = 1;
auto buildSizesInfo = device->getAccelerationStructureBuildSizesKHR(
    vk::AccelerationStructureBuildTypeKHR::eDevice, buildGeometryInfo, primitiveCount);
```

アクセラレーション構造を保持するバッファを作成します。既に関数があるので呼び出すだけでOKです。
```cpp
tlas.buffer = createAccelerationStructureBuffer(buildSizesInfo);
```

# トップレベルアクセラレーション構造を作成

トップレベルアクセラレーション構造を作成します。
```cpp
tlas.handle = device->createAccelerationStructureKHRUnique(
    vk::AccelerationStructureCreateInfoKHR{}
    .setBuffer(tlas.buffer.handle.get())
    .setSize(buildSizesInfo.accelerationStructureSize)
    .setType(vk::AccelerationStructureTypeKHR::eTopLevel)
);
```

これで準備完了です。ビルドに入りましょう。

# スクラッチバッファを作成

前回と同様にスクラッチバッファを用意します。

```cpp
Buffer scratchBuffer = createScratchBuffer(buildSizesInfo.buildScratchSize);
```

# トップレベルアクセラレーション構造をビルド

ビルドに必要な情報を作成します。ボトムレベルをトップレベルに書き換えた程度で、前の章とほぼ同じです。
```cpp
vk::AccelerationStructureBuildGeometryInfoKHR accelerationBuildGeometryInfo{};
accelerationBuildGeometryInfo
    .setType(vk::AccelerationStructureTypeKHR::eTopLevel)
    .setFlags(vk::BuildAccelerationStructureFlagBitsKHR::ePreferFastTrace)
    .setDstAccelerationStructure(tlas.handle.get())
    .setGeometries(geometry)
    .setScratchData(scratchBuffer.deviceAddress);
    
vk::AccelerationStructureBuildRangeInfoKHR accelerationStructureBuildRangeInfo{};
accelerationStructureBuildRangeInfo
    .setPrimitiveCount(1)
    .setPrimitiveOffset(0)
    .setFirstVertex(0)
    .setTransformOffset(0);
```

ビルドコマンドでビルドを実行します。こちらも同様です。

```cpp
auto commandBuffer = vkutils::createCommandBuffer(device.get(), commandPool.get(), true);
commandBuffer->buildAccelerationStructuresKHR(accelerationBuildGeometryInfo, &accelerationStructureBuildRangeInfo);
vkutils::submitCommandBuffer(device.get(), commandBuffer.get(), graphicsQueue);
```

最後にアドレスを取得しておきます。

```cpp
tlas.buffer.deviceAddress = device->getAccelerationStructureAddressKHR({ tlas.handle.get() });
```

以上でボトムレベルとトップレベルのアクセラレーション構造が作成完了しました。次の章ではレイトレーシングパイプラインを作成していきます。

[ここまでのC++コード(06_create_top_level_as.cpp)](https://github.com/nishidate-yuki/vulkan_raytracing_from_scratch/blob/master/code/06_create_top_level_as.cpp)
