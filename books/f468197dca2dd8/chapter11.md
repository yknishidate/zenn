この章ではシェーダバインディングテーブルを作成していきます。

# シェーダバインディングテーブルについて

レイトレーシングではレイを飛ばしたらミスするのか、オブジェクトAにヒットするのか、オブジェクトBにヒットするのか、といったことは飛ばしてみなければ分かりません。それはつまりどのシェーダを実行すればいいのかが事前に分からないということを意味します。

この状況でレンダリングするためには必要なことが3つあります。

1. すべてのシェーダがデバイス上でいつでも利用可能であること
2. どの状況でどのシェーダを実行するべきかが分かること
3. どのシェーダがどこに保存されているのかが分かること

この2番と3番を伝えるためのテーブルがシェーダバインディングテーブルです。

2番をさらに具体的に考えると以下のようになります。

- どのRaygenシェーダをエントリポイントとするか
- どのMissシェーダを実行するか
- どのインスタンスにどのヒットシェーダを割り当てるか

大規模なプロジェクトだとこの辺りが複雑になってきますが、今回の記事では各シェーダはそれぞれ1つずつなのでシンプルです。

---

それではコードに入っていきます。大まかな流れはこのようになります。

1. シェーダバインディングテーブルのサイズを計算する
2. シェーダグループの全ハンドルを取得する
3. シェーダバインディングテーブルのバッファを作成する

まず`createShaderBindingTable()`を作成して`initVulkan()`から呼び出します。

```cpp
void initVulkan()
{
    ...
    
    createShaderBindingTable();
}

void createShaderBindingTable()
{
    
}
```

# シェーダバインディングテーブルのサイズを計算する

シェーダグループのハンドルサイズは物理デバイスのプロパティから取得できます。今回はヘルパーを経由しますが、単純な関数です。グループ数と掛けて、シェーダバインディングテーブルのサイズを計算します。

```cpp
const uint32_t handleSize = vkutils::getShaderGroupHandleSize();
const uint32_t handleSizeAligned = vkutils::getHandleSizeAligned();
const uint32_t groupCount = static_cast<uint32_t>(shaderGroups.size());
const uint32_t sbtSize = groupCount * handleSizeAligned;
```

# シェーダグループの全ハンドルを取得する

シェーダグループのハンドルを取得します。前の章でレイトレーシングパイプラインを作成した際にシェーダがデバイスに送られているため、現在デバイス上のどこにシェーダがあるのかというハンドルを取得できるようになっています。

```cpp
std::vector<uint8_t> shaderHandleStorage(sbtSize);
auto result = device->getRayTracingShaderGroupHandlesKHR(pipeline.get(), 0, groupCount, static_cast<size_t>(sbtSize), shaderHandleStorage.data());
if (result != vk::Result::eSuccess) {
    throw std::runtime_error("failed to get ray tracing shader group handles.");
}
```

# シェーダバインディングテーブルのバッファを作成する

このデータを使ってバッファを作成します。まずは何度もやっているように`usage`と`memoryProperty`を設定します。`usage`には`ShaderBindingTableKHR`、`memoryProperty`はホストからコピーしたいので`HostVisible`を指定しています。

```cpp
const vk::BufferUsageFlags sbtBufferUsafgeFlags = 
    vk::BufferUsageFlagBits::eShaderBindingTableKHR
    | vk::BufferUsageFlagBits::eTransferSrc 
    | vk::BufferUsageFlagBits::eShaderDeviceAddress;

const vk::MemoryPropertyFlags sbtMemoryProperty =
    vk::MemoryPropertyFlagBits::eHostVisible 
    | vk::MemoryPropertyFlagBits::eHostCoherent;
```

バッファをメンバ変数に追加します。

```cpp
Buffer raygenShaderBindingTable;
Buffer missShaderBindingTable;
Buffer hitShaderBindingTable;
```

バッファを作成します。今回はすべてのグループがシェーダ1つを含むので、このようなコードになります。

```cpp
raygenShaderBindingTable = createBuffer(handleSize, sbtBufferUsafgeFlags, sbtMemoryProperty, shaderHandleStorage.data() + 0 * handleSizeAligned);
missShaderBindingTable = createBuffer(handleSize, sbtBufferUsafgeFlags, sbtMemoryProperty, shaderHandleStorage.data() + 1 * handleSizeAligned);
hitShaderBindingTable = createBuffer(handleSize, sbtBufferUsafgeFlags, sbtMemoryProperty, shaderHandleStorage.data() + 2 * handleSizeAligned);
```

以上でシェーダバインディングテーブルの作成が完了しました。実際はただのバッファとして扱えることが分かります。

次の章ではディスクリプタセットを作成します。

[これまでのC++コード(08_create_shader_binding_table.cpp)](https://github.com/nishidate-yuki/vulkan_raytracing_from_scratch/blob/master/code/08_create_shader_binding_table.cpp)

