---
title: "ディスクリプタセットの作成"
---

この章ではディスクリプタセットを作成します。ディスクリプタ自体はレイトレ特有のものではないのですが、簡単に整理します。

# ディスクリプタについて

ディスクリプタは様々なデータをシェーダから使用できるようにするための概念です。

ディスクリプタという単語はパイプラインの作成時に既に出てきています。詳しく説明しませんでしたが、パイプラインの作成時に出てきた構造体は`DescriptorSetLayout`、`PipelineLayout`など、`Layout`という名前がついていました。`Layout`はその名の通り、データのレイアウトだけを指定するもので、データの中身については参照しないものでした。これだけではデータを使えません。

つまり、シェーダからユニフォーム変数を使うためには

- データ自体
- データがどのように並んでいるかを示すレイアウト

この2つが必要です。これらは次の図のように対応しています。

![](https://storage.googleapis.com/zenn-user-upload/7vdwkau1dusty7fpwfbw8emxmo53)

図を見れば明らかなように、パイプラインの作成時には右側のレイアウトだけを先に作成したということです。ということで、この章では左側を作成していきます。

大まかな流れはこのようになります。

1. ディスクリプタプールを作成
2. ディスクリプタセットを確保
3. トップレベルアクセラレーション構造のディスクリプタを作成
4. ストレージイメージのディスクリプタを作成
5. ディスクリプタセットを更新

# ディスクリプタプールを作成

まずはディスクリプタプールとディスクリプタセットをメンバ変数に追加します。

```cpp
vk::UniqueDescriptorPool descriptorPool;
vk::UniqueDescriptorSet descriptorSet;
```

いつものように新しい関数を作成して呼び出します。

```cpp
void initVulkan()
{
    ...

    createDescriptorSets();
}

void createDescriptorSets()
{
    
}
```

ディスクリプタプールはディスクリプタセットを割り当てるためのメモリ領域のようなものです。ですので、まずはどの程度の大きさが必要なのかを決める必要があります。

```cpp
std::vector<vk::DescriptorPoolSize> poolSizes = {
    {vk::DescriptorType::eAccelerationStructureKHR, 1},
    {vk::DescriptorType::eStorageImage, 1}
};
```

今回必要なのは1つのトップレベルアクセラレーション構造と1つのストレージイメージです。

次に実際にディスクリプタプールを作成します。`FreeDescriptorSet`を指定すると、このプールから作成したディスクリプタセットはプールが破棄される際に自動で解放されるようになります。

```cpp
descriptorPool = device->createDescriptorPoolUnique(
    vk::DescriptorPoolCreateInfo{}
    .setPoolSizes(poolSizes)
    .setMaxSets(1)
    .setFlags(vk::DescriptorPoolCreateFlagBits::eFreeDescriptorSet)
);
```

# ディスクリプタセットを確保

ディスクリプタセットは複数作ることができますが、今回は1つだけでOKです。`allocateDescriptorSetsUnique()`には既に作成したディスクリプタセットレイアウトを渡しています。このレイアウトと同じようにディスクリプタセットを作成してくれます。

```cpp
auto descriptorSets = device->allocateDescriptorSetsUnique(
    vk::DescriptorSetAllocateInfo{}
    .setDescriptorPool(descriptorPool.get())
    .setSetLayouts(descriptorSetLayout.get())
);
descriptorSet = std::move(descriptorSets.front());
```

この関数は作成するディスクリプタセットが1つだけでも配列の形で結果を返してきます。ですので、配列の先頭要素を`std::move`して取得します。

# トップレベルアクセラレーション構造のディスクリプタを作成

ディスクリプタを作成と言っていますが、実際に`vk::Descriptor`などという構造体があるわけではないので注意です。コマンドなどで操作する場合はディスクリプタセットが1つの単位となっています。

ここでは`vk::WriteDescriptorSet`という構造体を使います。
```cpp
vk::WriteDescriptorSetAccelerationStructureKHR descriptorAccelerationStructureInfo{};
descriptorAccelerationStructureInfo
    .setAccelerationStructures(tlas.handle.get());

vk::WriteDescriptorSet accelerationStructureWrite{};
accelerationStructureWrite
    .setDstSet(descriptorSet.get())
    .setDstBinding(0)
    .setDescriptorCount(1)
    .setDescriptorType(vk::DescriptorType::eAccelerationStructureKHR)
    .setPNext(&descriptorAccelerationStructureInfo);
```

アクセラレーション構造は拡張機能で提供される構造体なので、特別な`vk::WriteDescriptorSetAccelerationStructureKHR`を先に作成して`setPNext()`で`vk::WriteDescriptorSet`の後ろにくっつけています。

# ストレージイメージのディスクリプタを作成

レイトレーシングの結果を保存するストレージイメージについても`vk::WriteDescriptorSet`を作成します。

```cpp
vk::DescriptorImageInfo imageDescriptor{};
imageDescriptor
    .setImageView(storageImage.view.get())
    .setImageLayout(vk::ImageLayout::eGeneral);

vk::WriteDescriptorSet resultImageWrite{};
resultImageWrite
    .setDstSet(descriptorSet.get())
    .setDescriptorType(vk::DescriptorType::eStorageImage)
    .setDstBinding(1)
    .setImageInfo(imageDescriptor);
```

# ディスクリプタセットを更新

最後に作成した2つの構造体を渡して、ディスクリプタを更新します。

```cpp
device->updateDescriptorSets({ accelerationStructureWrite, resultImageWrite }, nullptr);
```

以上でディスクリプタセットが作成できました。

この記事もそろそろ終盤です。次の章ではコマンドバッファを構築していきます。

[ここまでのC++コード(09_create_descriptor_sets.cpp)](https://github.com/nishidate-yuki/vulkan_raytracing_from_scratch/blob/master/code/09_create_descriptor_sets.cpp)