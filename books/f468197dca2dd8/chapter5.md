---
title: "ストレージイメージの作成"
---

Vulkan Ray Tracingではレイトレーシングで計算したピクセル値を1枚のイメージに格納します。

![](https://storage.googleapis.com/zenn-user-upload/pawnsgd81eatekod2irc116k8tvx)

ラスタライズではスワップチェインのイメージにレンダリングするのに対して、レイトレーシングではストレージイメージに**レンダリングした後にコピーする**形になります。

余談ですがOpenGLのComputeShaderでレイトレーシングする場合は、テクスチャに結果を書き込んだ後、カメラの視界を覆うように設置したポリゴンに張り付けて描画するという手法が用いられます。それと比べると無駄な作業が少ないことが分かるかと思います。

大まかな流れは次のようになります。

1. 構造体を定義
2. イメージを作成
3. メモリを確保
4. ビューを取得
5. イメージレイアウトを変更

# StorageImageの構造体

まずはStorageImageの構造体を作成します。
```cpp
struct StorageImage
{
    vk::UniqueDeviceMemory memory;
    vk::UniqueImage image;
    vk::UniqueImageView view;
    vk::Format format;
    uint32_t width;
    uint32_t height;
};
```

メンバ変数に追加します。
```cpp
private:
    ...

    StorageImage storageImage;
```

# StorageImageの作成

`createStorageImage()`を作成し、`initVulkan()`の中で実行します。ここからは作成したメンバ関数の中身を記述していきます。

```cpp
void initVulkan()
{
    ...

    createStorageImage();
}

void createStorageImage()
{
    
}
```

まずはイメージのサイズを指定します。

```cpp
void createStorageImage()
{
    storageImage.width = WIDTH;
    storageImage.height = HEIGHT;
}
```

次にイメージハンドルを作成します。先述の通り、コピー元として使用したいので`ImageUsageFlagBits::eTransferSrc`を指定しています。
```cpp
storageImage.image = device->createImageUnique(
    vk::ImageCreateInfo{}
    .setImageType(vk::ImageType::e2D)
    .setFormat(vk::Format::eB8G8R8A8Unorm)
    .setExtent({ storageImage.width , storageImage.height, 1 })
    .setMipLevels(1)
    .setArrayLayers(1)
    .setUsage(vk::ImageUsageFlagBits::eTransferSrc | vk::ImageUsageFlagBits::eStorage)
);
```
初めて`vulkan.hpp`を使うと記法に違和感を覚えるかもしれません。`createImageUnique()`は`vk::ImageCreateInfo`構造体を受け取る関数です。引数の中でこの構造体を初期化して、さらにそのフィールドの設定も行っています。コンストラクタもセッターも構造体のインスタンスを返すため、このように続けて記述することが可能です。（メソッドチェーンと呼ばれる手法です）

ハンドルを作成できたらメモリ要件を取得し、それを用いてメモリ確保を行います。メモリタイプには`MemoryPropertyFlagBits::eDeviceLocal`を指定しています。これはホストからアクセスできず、GPUからは最も高速にアクセスできることを示します。多くの場合は単純にVRAM上となります。

```cpp
auto memoryRequirements = device->getImageMemoryRequirements(storageImage.image.get());
storageImage.memory = device->allocateMemoryUnique(
    vk::MemoryAllocateInfo{}
    .setAllocationSize(memoryRequirements.size)
    .setMemoryTypeIndex(vkutils::getMemoryType(
        memoryRequirements, vk::MemoryPropertyFlagBits::eDeviceLocal))
);
```

確保できたメモリは先ほど作成したハンドルにバインドします。
```cpp
device->bindImageMemory(storageImage.image.get(), storageImage.memory.get(), 0);
```

このような

- ハンドルの作成
- メモリ確保
- ハンドルとメモリをバインド

という一連の流れは今後もよく登場しますので、覚えておくと楽だと思います。

# イメージのビューの取得

次は作成したイメージに対するビューを取得します。パイプラインからイメージにアクセスするには、どのようにアクセスし、どう解釈すればいいのかをイメージビューを用いて伝える必要があります。

```cpp
storageImage.view = device->createImageViewUnique(
    vk::ImageViewCreateInfo{}
    .setImage(storageImage.image.get())
    .setViewType(vk::ImageViewType::e2D)
    .setFormat(vk::Format::eB8G8R8A8Unorm)
    .setSubresourceRange({ vk::ImageAspectFlagBits::eColor, 0, 1, 0, 1 })
);
```

# イメージレイアウトを設定

イメージはレイアウトを持っています。これはピクセル値がメモリ上でどのようにレイアウトされているかを表すものです。イメージに対して何か操作をするとき、そのイメージレイアウトが操作に対して適切かどうかを確認する必要があります。

たとえば

- 表示用のレイアウト
- フラグメントシェーダから書き込む用のレイアウト
- コピー元用のレイアウト
- コピー先用のレイアウト

などがあります。

ここではイメージレイアウトを`General`に設定しておきます。コピー操作の際にはこれをコピー元用のレイアウトに変更することになります。

レイアウトの変更はデバイス上で行われるので、コマンドバッファを作成してコマンドを積み、提出するという手順が必要になります。

```cpp
auto commandBuffer = vkutils::createCommandBuffer(device.get(), commandPool.get(), true);

vkutils::setImageLayout(commandBuffer.get(), storageImage.image.get(),
    vk::ImageLayout::eUndefined, vk::ImageLayout::eGeneral,
    { vk::ImageAspectFlagBits::eColor, 0, 1, 0, 1 });

vkutils::submitCommandBuffer(device.get(), commandBuffer.get(), graphicsQueue);
```

これでストレージイメージが作成できました。レイトレーシングで計算した画像はここに保存されることになります。

使い方をイメージしやすくするために、すこし先取りしてシェーダコードを見てみましょう。

```glsl:example.rgen
imageStore(image, ivec2(gl_LaunchIDEXT.xy), vec4(hitValue, 0.0));
```

このようにシェーダでは`imageStore()`関数で`image`に書き込みが行われています。この`image`が今作成したストレージイメージになります。

次の章ではアクセラレーション構造を作成していきます。

[ここまでのC++コード(02_create_storage_image.cpp)](https://github.com/nishidate-yuki/vulkan_raytracing_from_scratch/blob/master/code/02_create_storage_image.cpp)
