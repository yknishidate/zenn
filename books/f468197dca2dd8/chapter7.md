---
title: "BottomLevelASの作成"
---

# アクセラレーション構造(AS)について

ASはポリゴンとレイの交差判定を高速化するための階層的なデータ構造です。ASの具体的な実装はVulkanの仕様に含まれていませんが、基本的にはBounding Volume Hierarchy(BVH)が使われます。

VulkanのASは以下の2つに分かれています。

- ボトムレベルアクセラレーション構造(BLAS)：
  ひとつのメッシュを表す構造。ポリゴンを構成する頂点やインデックスを保持する。
- トップレベルアクセラレーション構造(TLAS)：
  ひとつのシーンを表す構造。BLASへの参照と変換行列の組をインスタンスとして、その集合を保持する。

![](https://nvpro-samples.github.io/vk_raytracing_tutorial_KHR/Images/AccelerationStructure.svg)
*出典:[NVIDIA Vulkan Ray Tracing Tutorial](https://nvpro-samples.github.io/vk_raytracing_tutorial_KHR/)*

# AS構造体の作成

この章では、BLASを作成していきます。BLASとTLASを共通化するためのAS構造体を作成し、BLASをメンバー変数に追加します。ついでに`Vertex`構造体も追加しておきます。

```cpp
struct Vertex {
    float pos[3];
};

struct AccelStruct {
    vk::UniqueAccelerationStructureKHR accel;
    Buffer buffer;
};

class Application {
    // ...
    AccelStruct bottomAccel{};
};
```

# 三角ポリゴンの準備

`createBottomLevelAS()` という関数を作成します。

```cpp
void initVulkan() {
    // ...
    createBottomLevelAS();
}

void createBottomLevelAS() {
}
```

この記事では、メッシュは単純な三角形としているので、3つの頂点と3つのインデックスを用意します。

```cpp
void createBottomLevelAS() {
    std::cout << "Create BLAS\n";

    std::vector<Vertex> vertices = {
        {{1.0f, 1.0f, 0.0f}},
        {{-1.0f, 1.0f, 0.0f}},
        {{0.0f, -1.0f, 0.0f}},
    };
    std::vector<uint32_t> indices = {0, 1, 2};
}
```

このデータを一度バッファに格納します。ASのビルドに使うには`vk::BufferUsageFlagBits::eAccelerationStructureBuildInputReadOnlyKHR`を指定します。また、アドレスを取得するために`vk::BufferUsageFlagBits::eShaderDeviceAddress`も指定します。

```cpp
void createBottomLevelAS() {
    // ...

    // Create vertex buffer and index buffer
    vk::BufferUsageFlags bufferUsage{
        vk::BufferUsageFlagBits::eAccelerationStructureBuildInputReadOnlyKHR |
        vk::BufferUsageFlagBits::eShaderDeviceAddress};
    vk::MemoryPropertyFlags memoryProperty{
        vk::MemoryPropertyFlagBits::eHostVisible |
        vk::MemoryPropertyFlagBits::eHostCoherent};
    Buffer vertexBuffer;
    Buffer indexBuffer;
    vertexBuffer.init(physicalDevice, *device,
                      vertices.size() * sizeof(Vertex),
                      bufferUsage, memoryProperty, vertices.data());
    indexBuffer.init(physicalDevice, *device,
                      indices.size() * sizeof(uint32_t),
                      bufferUsage, memoryProperty, indices.data());
}
```

作成したバッファをもとに、ASのジオメトリを作成します。Vulkan Ray Tracingでは、Triangle、AABBs、Instancesの3タイプのジオメトリがサポートされていますが、今回は`vk::GeometryTypeKHR::eTriangles`を使用します。また、透過は考慮しないので、フラグは`vk::GeometryFlagBitsKHR::eOpaque`としておきます。

```cpp
void createBottomLevelAS() {
    // ...

    // Create geometry
    vk::AccelerationStructureGeometryTrianglesDataKHR triangles{};
    triangles.setVertexFormat(vk::Format::eR32G32B32Sfloat);
    triangles.setVertexData(vertexBuffer.address);
    triangles.setVertexStride(sizeof(Vertex));
    triangles.setMaxVertex(static_cast<uint32_t>(vertices.size()));
    triangles.setIndexType(vk::IndexType::eUint32);
    triangles.setIndexData(indexBuffer.address);

    vk::AccelerationStructureGeometryKHR geometry{};
    geometry.setGeometryType(vk::GeometryTypeKHR::eTriangles);
    geometry.setGeometry({triangles});
    geometry.setFlags(vk::GeometryFlagBitsKHR::eOpaque);
}
```

# ASの作成

用意したジオメトリをもとにASを作成します。BLASとTLASでロジックを共通化するため、`AccelStruct`構造体に`AccelStruct::init()`という関数を作成し、`createBottomLevelAS()`の最後に呼び出します。BLASとして作成するため、`vk::AccelerationStructureTypeKHR::eBottomLevel`を指定します。

```cpp
struct AccelStruct {
    // ...
    void init(vk::PhysicalDevice physicalDevice, vk::Device device,
              vk::CommandPool commandPool, vk::Queue queue,
              vk::AccelerationStructureTypeKHR type,
              vk::AccelerationStructureGeometryKHR geometry,
              uint32_t primitiveCount){

    }
};

void createBottomLevelAS() {
    // ...

    // Create and build BLAS
    uint32_t primitiveCount = static_cast<uint32_t>(indices.size() / 3);
    bottomAccel.init(physicalDevice, *device, *commandPool, queue,
                     vk::AccelerationStructureTypeKHR::eBottomLevel,
                     geometry, primitiveCount);
}
```

`AccelStruct::init()`の中身を作成します。まず、ビルドの設定を用意します。

- `type`はBLASかTLASかを指定します。
- `mode`はビルドのモードを指定します。
  - ゼロからビルドする場合は`vk::BuildAccelerationStructureModeKHR::eBuild`を指定します。
  - 構築済みのASを更新する場合は`vk::BuildAccelerationStructureModeKHR::eUpdate`を指定します。例えばインスタンスが動くシーンにおいて、TLASを高速に更新したい場合に有効です。
- `flags`はビルドのオプションを指定します。一般的に、BVHはビルドの速度とトレースの速度のトレードオフがあります。
  - `vk::BuildAccelerationStructureFlagBitsKHR::ePreferFastTrace`を指定すると、トレースの速度を優先します。
  - `vk::BuildAccelerationStructureFlagBitsKHR::ePreferFastBuild`を指定すると、ビルドの速度を優先します。
  - また、ASの更新を行う場合は`vk::BuildAccelerationStructureFlagBitsKHR::eAllowUpdate`も指定する必要があります。

```cpp
void init(vk::PhysicalDevice physicalDevice,
          vk::Device device,
          vk::CommandPool commandPool,
          vk::Queue queue,
          vk::AccelerationStructureTypeKHR type,
          vk::AccelerationStructureGeometryKHR geometry,
          uint32_t primitiveCount) {
    // Get build info
    vk::AccelerationStructureBuildGeometryInfoKHR buildInfo{};
    buildInfo.setType(type);
    buildInfo.setMode(vk::BuildAccelerationStructureModeKHR::eBuild);
    buildInfo.setFlags(
        vk::BuildAccelerationStructureFlagBitsKHR::ePreferFastTrace);
    buildInfo.setGeometries(geometry);

    vk::AccelerationStructureBuildSizesInfoKHR buildSizes =
        device.getAccelerationStructureBuildSizesKHR(
            vk::AccelerationStructureBuildTypeKHR::eDevice, buildInfo,
            primitiveCount);
}
```

次に、ASを格納するためのバッファを作成します。ASはデバイスメモリに格納しないと著しくパフォーマンスが低下するため、`vk::MemoryPropertyFlagBits::eDeviceLocal`を指定します。

```cpp
void init(...) {
    // ...
    // Create buffer for AS
    buffer.init(physicalDevice, device,
                buildSizes.accelerationStructureSize,
                vk::BufferUsageFlagBits::eAccelerationStructureStorageKHR,
                vk::MemoryPropertyFlagBits::eDeviceLocal);
}
```

次に、ASのハンドルを作成します。作成するASのサイズとタイプを指定します。

```cpp
void init(...) {
    // ...
    // Create AS
    vk::AccelerationStructureCreateInfoKHR createInfo{};
    createInfo.setBuffer(*buffer.buffer);
    createInfo.setSize(buildSizes.accelerationStructureSize);
    createInfo.setType(type);
    accel = device.createAccelerationStructureKHRUnique(createInfo);
}
```

# ASのビルド

ASをビルドするためには、スクラッチバッファが必要です。スクラッチバッファはビルドに必要な一時的なメモリであり、必要なサイズは`vk::AccelerationStructureBuildSizesInfoKHR::buildScratchSize`に格納されています。多数のASをビルドする際には、スクラッチバッファは再利用するといいです。

```cpp
void init(...) {
    // ...
    // Create scratch buffer
    Buffer scratchBuffer;
    scratchBuffer.init(physicalDevice, device, buildSizes.buildScratchSize,
                       vk::BufferUsageFlagBits::eStorageBuffer |
                       vk::BufferUsageFlagBits::eShaderDeviceAddress,
                       vk::MemoryPropertyFlagBits::eDeviceLocal);

    buildInfo.setDstAccelerationStructure(*accel);
    buildInfo.setScratchData(scratchBuffer.address);
}
```

では、いよいよASをビルドします。`vkutils::oneTimeSubmit()`を使って、コマンドバッファを一時的に確保し、ビルドコマンドを実行します。`vk::AccelerationStructureBuildRangeInfoKHR`を使うと細かいビルドの範囲を指定することができますが、今回は単純に全体をビルドします。

```cpp
void init(...) {
    // ...
    // Build
    vk::AccelerationStructureBuildRangeInfoKHR buildRangeInfo{};
    buildRangeInfo.setPrimitiveCount(primitiveCount);
    buildRangeInfo.setPrimitiveOffset(0);
    buildRangeInfo.setFirstVertex(0);
    buildRangeInfo.setTransformOffset(0);

    vkutils::oneTimeSubmit(          //
        device, commandPool, queue,  //
        [&](vk::CommandBuffer commandBuffer) {
            commandBuffer.buildAccelerationStructuresKHR(buildInfo,
                                                         &buildRangeInfo);
        });
}
```

最後に、ASのアドレスを取得します。BLASのアドレスは、TLASをビルドする際に必要になります。

```cpp
void init(...) {
    // ...
    // Get address
    vk::AccelerationStructureDeviceAddressInfoKHR addressInfo{};
    addressInfo.setAccelerationStructure(*accel);
    buffer.address = device.getAccelerationStructureAddressKHR(addressInfo);
}
```

以上でBLASの作成が完了しました。次はTLASの作成に移ります。

[ここまでのC++コード(04_create_bottom_level_as.cpp)](https://github.com/nishidate-yuki/vulkan_raytracing_from_scratch/blob/master/code/04_create_bottom_level_as.cpp)
