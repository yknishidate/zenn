
レイトレーシングではバーテックスバッファをそのまま描画に使用するわけではなく、アクセラレーション構造を使います。

# アクセラレーション構造について

アクセラレーション構造は以下の2つに分かれています。

- ボトムレベルアクセラレーション構造
  実際に頂点データなどを保持する構造
- トップレベルアクセラレーション構造
  ボトムレベルアクセラレーション構造への参照と、それに対する変換行列などを保持する構造

アクセラレーション構造は基本的にはBVH(Bounding Volume Hierarchy)をイメージすればいいようです。これを用いることで高速な衝突判定が行えるようになります。

![](https://nvpro-samples.github.io/vk_raytracing_tutorial_KHR/Images/AccelerationStructure.svg)
出典:[NVIDIA Vulkan Ray Tracing Tutorial](https://nvpro-samples.github.io/vk_raytracing_tutorial_KHR/)

ちなみに、上図のように異なるインスタンスから同じボトムレベルアクセラレーション構造を参照すれば、簡単にインスタンシングを実現できます。

---

3章から5章ではボトムレベルアクセラレーション構造を作成していきます。

- 3章: バーテックスバッファとインデックスバッファの作成
- 4章: ボトムレベルアクセラレーション構造の準備
- 5章: ボトムレベルアクセラレーション構造のビルド

まずは準備段階として、バーテックスバッファとインデックスバッファを作成します。大まかな流れはこのようになります。

1. 構造体を定義
2. 三角形のデータを用意
3. バッファを作成

# Vertex構造体とBuffer構造体

Vertex構造体とBuffer構造体をクラスの上に定義します。
```cpp
struct Vertex
{
    float pos[3];
};

struct Buffer
{
    vk::UniqueBuffer handle;
    vk::UniqueDeviceMemory deviceMemory;
    uint64_t deviceAddress;
};
```

次に`createBottomLevelAS()`を作成して、`initVulkan()`から呼び出します。

```cpp
void initVulkan()
{
    ...
    createBottomLevelAS();
}
```

```cpp
void createBottomLevelAS()
{
}
```

# データの用意

`createBottomLevelAS()`の先頭では、まず三角形のデータを定義します。
```cpp
std::vector<Vertex> vertices = {
    {{1.0f, 1.0f, 0.0f}},
    {{-1.0f, 1.0f, 0.0f}},
    {{0.0f, -1.0f, 0.0f}} };
std::vector<uint32_t> indices = { 0, 1, 2 };
```

定義したデータからバッファサイズを計算しておき、バッファの使い方とメモリプロパティも設定します。

```cpp
auto vertexBufferSize = vertices.size() * sizeof(Vertex);
auto indexBufferSize = indices.size() * sizeof(uint32_t);
vk::BufferUsageFlags bufferUsage{ 
    vk::BufferUsageFlagBits::eAccelerationStructureBuildInputReadOnlyKHR
    | vk::BufferUsageFlagBits::eShaderDeviceAddress
    | vk::BufferUsageFlagBits::eStorageBuffer };
vk::MemoryPropertyFlags memoryProperty{ 
    vk::MemoryPropertyFlagBits::eHostVisible
    | vk::MemoryPropertyFlagBits::eHostCoherent };
```

アクセラレーション構造のビルドに使用するため`bufferUsage`には`BufferUsageFlagBits::eAccelerationStructureBuildInputReadOnlyKHR`を指定しています。また、`memoryProperty`にはホストから見えるメモリを指定しています。これは三角形データを簡単にコピーできるようにするためです。


# バッファの作成

これらを元にバッファを作成していきます。新たな関数`createBuffer()`を追加して呼び出します。今度は`createBuffer()`に移って中身を書いていきましょう。

```cpp
void createBottomLevelAS()
{
    ...
    Buffer vertexBuffer = createBuffer(vertexBufferSize, bufferUsage, memoryProperty, vertices.data());
    Buffer indexBuffer = createBuffer(indexBufferSize, bufferUsage, memoryProperty, indices.data());
}

Buffer createBuffer(vk::DeviceSize size, vk::BufferUsageFlags usage, 
    vk::MemoryPropertyFlags memoryPropertiy, void* data = nullptr)
{

}
```

`createBuffer()`の流れは

- ハンドル作成
- メモリ確保
- ハンドルとメモリをバインド
- データをメモリにコピー
- デバイスアドレスを取得

となっています。

まずはハンドルを作成します。
```cpp
Buffer buffer{};
buffer.handle = device->createBufferUnique(
    vk::BufferCreateInfo{}
    .setSize(size)
    .setUsage(usage)
    .setQueueFamilyIndexCount(0)
);
```

メモリを確保します。
```cpp
auto memoryRequirements = device->getBufferMemoryRequirements(buffer.handle.get());
vk::MemoryAllocateFlagsInfo memoryFlagsInfo{};

if (usage & vk::BufferUsageFlagBits::eShaderDeviceAddress) {
    memoryFlagsInfo.flags = vk::MemoryAllocateFlagBits::eDeviceAddress;
}

buffer.deviceMemory = device->allocateMemoryUnique(
    vk::MemoryAllocateInfo{}
    .setAllocationSize(memoryRequirements.size)
    .setMemoryTypeIndex(vkutils::getMemoryType(memoryRequirements, memoryPropertiy))
    .setPNext(&memoryFlagsInfo)
);
```

ハンドルとメモリをバインドします。
```cpp
device->bindBufferMemory(buffer.handle.get(), buffer.deviceMemory.get(), 0);
```

データをメモリにコピーします。よく見るとただホスト側のメモリに`memcpy()`しているだけだということが分かります。ホストから見えるメモリを指定すると、マップするだけでこのように簡単にコピーが可能になります。
```cpp
if (data) {
    void* dataPtr = device->mapMemory(buffer.deviceMemory.get(), 0, size);
    memcpy(dataPtr, data, static_cast<size_t>(size));
    device->unmapMemory(buffer.deviceMemory.get());
}
```

最後にバッファのデバイスアドレスを取得したらバッファをリターンします。ただし、ここでさらにもう一つの関数`getBufferDeviceAddress()`を作成しています。

```cpp
Buffer createBuffer(...)
{
    ...

    vk::BufferDeviceAddressInfoKHR bufferDeviceAddressInfo{ buffer.handle.get() };
    buffer.deviceAddress = getBufferDeviceAddress(buffer.handle.get());

    return buffer;
}

uint64_t getBufferDeviceAddress(vk::Buffer buffer)
{
    vk::BufferDeviceAddressInfoKHR bufferDeviceAI{ buffer };
    return device->getBufferAddressKHR(&bufferDeviceAI);
}
```

これでバーテックスバッファとインデックスバッファの作成ができました。次の章ではボトムレベルアクセラレーション構造を作成します。

[ここまでのC++コード(03_create_vertex_buffer.cpp)](https://github.com/nishidate-yuki/vulkan_raytracing_from_scratch/blob/master/code/03_create_vertex_buffer.cpp)
