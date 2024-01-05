---
title: "TopLevelASの作成"
---

この章ではTLASを作成していきます。BLASでは三角ポリゴンを入力としていましたが、TLASではBLASの参照と変換行列を入力とします。ビルドなどの流れは既にAS構造体に実装してあるので、TLASの作成は簡単です。

# TLASの作成

まずはメンバ変数にTLASを追加します。
```cpp
AccelStruct topAccel{};
```

TLASを作成する関数を追加し、`initVulkan()`で呼び出します。

```cpp
void initVulkan() {
    // ...
    createTopLevelAS();
}

void createTopLevelAS() {
}
```

関数の中身を記述していきます。まずはTLASに格納するインスタンスを作成しましょう。複数のインスタンスを格納できますが、今回は1つだけにします。

- `setTransform()`には変換行列を渡します。今回は単位行列を渡しています。
- `setInstanceCustomIndex()`にはインスタンスのカスタムインデックスを渡します。これを利用すると、シェーダで指定したインデックスを取得できます。例えば、インスタンスIDとは別に、参照するメッシュのIDを取得すると便利です。
- `setMask()`にはインスタンスの可視性マスクを渡します。これはレイとの衝突判定に使われます。
- `setInstanceShaderBindingTableRecordOffset()`にはシェーダバインディングテーブルのオフセットを渡します。これは複数のヒットシェーダを扱うときに、どのシェーダを呼び出すかを指定するために使います。
- `setFlags()`にはインスタンスのフラグを渡します。今回はカリングを無効にしています。
- `setAccelerationStructureReference()`には参照するBLASのアドレスを渡します。

```cpp
void createTopLevelAS() {
    std::cout << "Create TLAS\n";

    // Create instance
    vk::TransformMatrixKHR transform = std::array{
        std::array{1.0f, 0.0f, 0.0f, 0.0f},
        std::array{0.0f, 1.0f, 0.0f, 0.0f},
        std::array{0.0f, 0.0f, 1.0f, 0.0f},
    };

    vk::AccelerationStructureInstanceKHR accelInstance{};
    accelInstance.setTransform(transform);
    accelInstance.setInstanceCustomIndex(0);
    accelInstance.setMask(0xFF);
    accelInstance.setInstanceShaderBindingTableRecordOffset(0);
    accelInstance.setFlags(
        vk::GeometryInstanceFlagBitsKHR::eTriangleFacingCullDisable);
    accelInstance.setAccelerationStructureReference(
        bottomAccel.buffer.address);
}
```

次にバッファを作成します。

```cpp
void createTopLevelAS() {
    // ...

    Buffer instanceBuffer;
    instanceBuffer.init(
        physicalDevice, *device,
        sizeof(vk::AccelerationStructureInstanceKHR),
        vk::BufferUsageFlagBits::eAccelerationStructureBuildInputReadOnlyKHR |
        vk::BufferUsageFlagBits::eShaderDeviceAddress,
        vk::MemoryPropertyFlagBits::eHostVisible |
        vk::MemoryPropertyFlagBits::eHostCoherent,
        &accelInstance);
}
```

インスタンスをもとにジオメトリの設定を行います。

```cpp
void createTopLevelAS() {
    // ...

    // Create geometry
    vk::AccelerationStructureGeometryInstancesDataKHR instancesData{};
    instancesData.setArrayOfPointers(false);
    instancesData.setData(instanceBuffer.address);

    vk::AccelerationStructureGeometryKHR geometry{};
    geometry.setGeometryType(vk::GeometryTypeKHR::eInstances);
    geometry.setGeometry({instancesData});
    geometry.setFlags(vk::GeometryFlagBitsKHR::eOpaque);
}
```

最後にASを作成します。`init()`の引数には`vk::AccelerationStructureTypeKHR::eTopLevel`を渡します。

```cpp
void createTopLevelAS() {
    // ...

    // Create and build TLAS
    constexpr uint32_t primitiveCount = 1;
    topAccel.init(physicalDevice, *device, *commandPool, queue,
                  vk::AccelerationStructureTypeKHR::eTopLevel,
                  geometry, primitiveCount);
}
```

以上でBLASとTLASの作成できました。

[ここまでのC++コード(06_create_top_level_as.hpp)](https://github.com/nishidate-yuki/vulkan_raytracing_from_scratch/blob/master/code/06_create_top_level_as.hpp)
