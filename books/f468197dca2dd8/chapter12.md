---
title: "コマンドバッファの構築"
---

この章では描画する際に使用するコマンドバッファを作成していきます。Vulkanではデバイスで実行したいコマンドはコマンドバッファに積み、サブミットすることで実行されます。ここで特徴的なのは、事前にコマンドバッファにコマンドを積んでおけばサブミットするだけで実行することが可能で、さらにそのコマンドバッファは何度でも使いまわすことができるということです。

この章では描画する際に必要な一連のコマンドを構築していきます。ここではコマンドを積むだけなので、実際にはなにも実行されないことを覚えておいてください。

大まかな流れはこのようになります。

1. コマンドバッファを開始する
2. シェーダのアドレス領域を計算する
3. レイトレーシングパイプラインをバインドする
4. ディスクリプタセットをバインドする
5. レイトレーシングを実行する
6. 出力イメージをスワップチェインのイメージにコピーする
7. スワップチェインのイメージを表示する
8. コマンドバッファを終了する

まずは新しい関数を追加します。

```cpp
void initVulkan()
{
    ...
    buildCommandBuffers();
}

void buildCommandBuffers()
{
    
}
```

コマンドバッファの操作に入る前に、後で使用するサブリソースレンジという構造体を作成しておきます。

```cpp
vk::ImageSubresourceRange subresourceRange{};
subresourceRange
    .setAspectMask(vk::ImageAspectFlagBits::eColor)
    .setBaseMipLevel(0)
    .setLevelCount(1)
    .setBaseArrayLayer(0)
    .setLayerCount(1);
```

# コマンドバッファを開始する

ここから、コマンドバッファを構築していきます。コマンドバッファのオブジェクト自体は、既にメンバ変数として用意されているはずです。

```cpp
std::vector<vk::UniqueCommandBuffer> drawCommandBuffers; // 既にあります
```

これは最初の方で`initVulkan()`の中でヘルパーを用いて取得してあります。

この全てのコマンドバッファに同じコマンドを積んでいくので、ここからは以下のような`for`の中に書いていきます。

```cpp
for (int32_t i = 0; i < drawCommandBuffers.size(); ++i) {
    // ここでコマンドを積む
}
```

まず最初はコマンドバッファの記録を開始します。

```cpp
drawCommandBuffers[i]->begin(
    vk::CommandBufferBeginInfo{}
);
```

# シェーダのアドレス領域を計算する

シェーダバインディングテーブルの情報を使って、シェーダがどこにあるのかを指定します。

```cpp
const uint32_t handleSizeAligned = vkutils::getHandleSizeAligned();

vk::StridedDeviceAddressRegionKHR raygenShaderSbtEntry{};
raygenShaderSbtEntry
    .setDeviceAddress(raygenShaderBindingTable.deviceAddress)
    .setStride(handleSizeAligned)
    .setSize(handleSizeAligned);

vk::StridedDeviceAddressRegionKHR missShaderSbtEntry{};
missShaderSbtEntry
    .setDeviceAddress(missShaderBindingTable.deviceAddress)
    .setStride(handleSizeAligned)
    .setSize(handleSizeAligned);

vk::StridedDeviceAddressRegionKHR hitShaderSbtEntry{};
hitShaderSbtEntry
    .setDeviceAddress(hitShaderBindingTable.deviceAddress)
    .setStride(handleSizeAligned)
    .setSize(handleSizeAligned);
```

# レイトレーシングパイプラインをバインドする

`bindPipeline()`でレイトレーシングパイプラインを指定します。

```cpp
drawCommandBuffers[i]->bindPipeline(vk::PipelineBindPoint::eRayTracingKHR, pipeline.get());
```

# ディスクリプタセットをバインドする

`bindDescriptorSets()`でディスクリプタセットを指定します。前の章でも触れましたが、このようにコマンドではディスクリプタセットをひとつの単位として操作します。

```cpp
drawCommandBuffers[i]->bindDescriptorSets(
    vk::PipelineBindPoint::eRayTracingKHR, // pipelineBindPoint
    pipelineLayout.get(),                  // layout
    0,                                     // firstSet
    descriptorSet.get(),                   // descriptorSets
    nullptr                                // dynamicOffsets
);
```

# レイトレーシングを実行する

いよいよレイトレーシングを実行するコマンドです。

```cpp
drawCommandBuffers[i]->traceRaysKHR(
    raygenShaderSbtEntry, // raygenShaderBindingTable
    missShaderSbtEntry,   // missShaderBindingTable
    hitShaderSbtEntry,    // hitShaderBindingTable
    {},                   // callableShaderBindingTable
    storageImage.width,   // width
    storageImage.height,  // height
    1                     // depth
);
```

# 出力イメージをスワップチェインのイメージにコピーする

レイトレーシングの結果はストレージイメージに保存されています。これを画面に表示するためにスワップチェインイメージにコピーします。

コピーコマンドを積む前に、ストレージイメージとスワップチェインイメージの両方のレイアウトを設定する必要があります。

まず、スワップチェインイメージをコピー先に設定します
```cpp
vkutils::setImageLayout(drawCommandBuffers[i].get(), 
    swapChainImages[i], vk::ImageLayout::eUndefined,
    vk::ImageLayout::eTransferDstOptimal, subresourceRange);
```

次にストレージイメージをコピー元に設定します。
```cpp
vkutils::setImageLayout(drawCommandBuffers[i].get(), 
    storageImage.image.get(), vk::ImageLayout::eUndefined, 
    vk::ImageLayout::eTransferSrcOptimal, subresourceRange);
```

これで準備完了です。コピーコマンドを積みます。
```cpp
vk::ImageCopy copyRegion{};
copyRegion
    .setSrcSubresource({ vk::ImageAspectFlagBits::eColor, 0, 0, 1 })
    .setSrcOffset({ 0, 0, 0 })
    .setDstSubresource({ vk::ImageAspectFlagBits::eColor, 0, 0, 1 })
    .setDstOffset({ 0, 0, 0 })
    .setExtent({ storageImage.width, storageImage.height, 1 });
drawCommandBuffers[i]->copyImage(
    storageImage.image.get(),             // srcImage
    vk::ImageLayout::eTransferSrcOptimal, // srcImageLayout
    swapChainImages[i],                   // dstImage
    vk::ImageLayout::eTransferDstOptimal, // dstImageLayout
    copyRegion                            // regions
);
```

コピーが完了したら、またレイアウトを設定しなおす必要があります。

スワップチェインイメージはこれから表示したいので、表示用に設定します。
```cpp
vkutils::setImageLayout(drawCommandBuffers[i].get(), 
    swapChainImages[i], vk::ImageLayout::eTransferDstOptimal, 
    vk::ImageLayout::ePresentSrcKHR, subresourceRange);
```

ストレージイメージは元の設定に戻しておきます。
```cpp
vkutils::setImageLayout(drawCommandBuffers[i].get(), 
    storageImage.image.get(), vk::ImageLayout::eTransferSrcOptimal, 
    vk::ImageLayout::eGeneral, subresourceRange);
```

これですべてのコマンドを積めました。最後にコマンドバッファの記録を終了します。

```cpp
drawCommandBuffers[i]->end();
```


これでコマンドバッファが完成しました。残るは描画です。次の章では最後の準備として同期用のオブジェクトを作成します。

[ここまでのC++コード(10_build_command_buffers.cpp)](https://github.com/nishidate-yuki/vulkan_raytracing_from_scratch/blob/master/code/10_build_command_buffers.cpp)