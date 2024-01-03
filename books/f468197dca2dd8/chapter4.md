---
title: "基本セットアップ"
---

この章ではレイトレーシングに関係のない、Vulkanの基本的なセットアップを一気に行います。ほとんどヘルパーに任せしてしまいます。

# 拡張機能を追加

セットアップを始める前に、まずはレイトレーシングに必要な拡張機能を有効化します。
`initVulkan()`のなかで拡張機能のリストを作成します。作成したリスト
は `vkutils::addDeviceExtensions()` でヘルパーに渡しておきます。

```cpp
void initVulkan()
{
    std::vector<const char*> deviceExtensions = {
        VK_KHR_RAY_TRACING_PIPELINE_EXTENSION_NAME,
        VK_KHR_ACCELERATION_STRUCTURE_EXTENSION_NAME
    };
    vkutils::addDeviceExtensions(deviceExtensions);
}
```

# 検証レイヤーを有効化する

さらに同じ関数内で検証レイヤーを有効化してデバッグメッセージが出るようにします。

```cpp
vkutils::enableDebugMessage();
```

# インスタンス・デバッグメッセンジャー

Vulkanのインスタンスとデバッグメッセンジャーをメンバ変数に追加します。

この記事では基本的に`UniqueHandle`としてメンバ変数を保持します。こうすることで明示的なメモリ解放を行わなくて済みます。ただし、オブジェクトによっては正しい順番で解放しないとエラーとなることがあるので注意が必要です。

```cpp

private:
    ...

    vk::UniqueInstance instance;
    vk::UniqueDebugUtilsMessengerEXT debugUtilsMessenger;
```

`initVulkan()`でこれらを作成します。
```cpp
instance = vkutils::createInstance();
debugUtilsMessenger = vkutils::createDebugMessenger(instance.get());
```

# サーフェス・デバイス・キュー

同様にサーフェスとデバイス、キューを追加して作成します。

```cpp
vk::UniqueSurfaceKHR surface;
vk::UniqueDevice device;
vk::Queue graphicsQueue;
```

```cpp
surface = vkutils::createSurface(instance.get(), window);
device = vkutils::createLogicalDevice(instance.get(), surface.get());
graphicsQueue = vkutils::getGraphicsQueue(device.get());
```

# スワップチェイン・イメージ

こちらも同様に、メンバ変数を追加して作成します。`vk::Image`は`UniqueHandle`にする必要はありません。

```cpp
vk::UniqueSwapchainKHR swapChain;
std::vector<vk::Image> swapChainImages;
```

```cpp
swapChain = vkutils::createSwapChain(device.get(), surface.get());
swapChainImages = vkutils::getSwapChainImages(device.get(), swapChain.get());
```

# コマンドプール・コマンドバッファ

このコマンドバッファは描画時に提出するためのものです。

```cpp
vk::UniqueCommandPool commandPool;
std::vector<vk::UniqueCommandBuffer> drawCommandBuffers;
```

```cpp
commandPool = vkutils::createCommandPool(device.get());
drawCommandBuffers = vkutils::createDrawCommandBuffers(device.get(), commandPool.get());
```

これで基本的なセットアップが完了しました。この章は天下り的になってしまいましたが、次章からいよいよレイトレーシングに関連する作業を行っていきます。

[ここまでのC++コード(01_skip_base_setup.cpp)](https://github.com/nishidate-yuki/vulkan_raytracing_from_scratch/blob/master/code/01_skip_base_setup.cpp)