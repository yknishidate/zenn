---
title: "基本セットアップ"
---

この章では、Vulkanの基本的なセットアップを行います。

# 必要なメンバー変数を追加

まずは必要なメンバー変数を追加します。

```cpp
class Application {
    // ...

private:
    vk::UniqueInstance instance;
    vk::UniqueDebugUtilsMessengerEXT debugMessenger;
    vk::UniqueSurfaceKHR surface;

    vk::PhysicalDevice physicalDevice;
    vk::UniqueDevice device;

    vk::Queue queue;
    uint32_t queueFamilyIndex{};

    vk::UniqueCommandPool commandPool;
    vk::UniqueCommandBuffer commandBuffer;
};
```

この記事では、`vk::UniqueHandle<T>`を使ってメンバー変数を保持します。これは、自動的にメモリを解放してくれるハンドルです。`vk::PhysicalDevice`や`vk::Queue`など、一部のオブジェクトは元々解放が不要なので、`vk::UniqueHandle`を使う必要はありません。

# インスタンスを作成

まずはインスタンスを作成します。`initVulkan()`のなかで`vkutils::createInstance()`を呼び出します。この際、後述する検証レイヤーを有効化するために、`VK_LAYER_KHRONOS_validation`を有効化しておきます。また、レイトレーシングを利用するために、`Vulkan 1.2`を指定します。

```diff cpp
void initVulkan() {
    std::vector<const char*> layers = {
        "VK_LAYER_KHRONOS_validation",
    };

    instance = vkutils::createInstance(VK_API_VERSION_1_2, layers);
}
```

# デバッグメッセンジャーを作成

検証レイヤーによるデバッグメッセージを出力するために、デバッグメッセンジャーを作成します。

```diff cpp
void initVulkan() {
    // ...
+   debugMessenger = vkutils::createDebugMessenger(*instance);
}
```

# サーフェスを作成

サーフェスはウィンドウシステムとVulkanを接続するためのものです。この記事ではGLFWを使っているため、GLFWのウィンドウからサーフェスを作成します。

```diff cpp
void initVulkan() {
    // ...
+   surface = vkutils::createSurface(*instance, window);
}
```

# 物理デバイスを選択

物理デバイスは、VulkanをサポートしているGPUのことです。この記事では、レイトレーシングを利用するために、必要な拡張機能をサポートしているデバイスを選択します。

```diff cpp
void initVulkan() {
    // ...
+   std::vector<const char*> deviceExtensions = {
+       // For swapchain
+       VK_KHR_SWAPCHAIN_EXTENSION_NAME,
+       // For ray tracing
+       VK_KHR_PIPELINE_LIBRARY_EXTENSION_NAME,
+       VK_KHR_RAY_TRACING_PIPELINE_EXTENSION_NAME,
+       VK_KHR_ACCELERATION_STRUCTURE_EXTENSION_NAME,
+       VK_KHR_DEFERRED_HOST_OPERATIONS_EXTENSION_NAME,
+       VK_KHR_BUFFER_DEVICE_ADDRESS_EXTENSION_NAME,
+   };

+   physicalDevice = vkutils::pickPhysicalDevice(*instance, 
+                                                *surface, 
+                                                deviceExtensions);
}
```

# 論理デバイスを作成

論理デバイスは、物理デバイスを抽象化したものです。Vulkanのほとんどのオブジェクトは論理デバイスを介して作成されます。また、GPUにコマンドを送るために必要なキューも論理デバイスから取得します。キューには種類があるのですが、この記事では汎用的なキューを取得しておきます。

```diff cpp
void initVulkan() {
    // ...
+   queueFamilyIndex = vkutils::findGeneralQueueFamily(physicalDevice, *surface);
+   device = vkutils::createLogicalDevice(physicalDevice,
+                                         queueFamilyIndex,
+                                         deviceExtensions);
+   queue = device->getQueue(queueFamilyIndex, 0);
}
```

# コマンドプールとコマンドバッファを作成

コマンドバッファはGPUに送るコマンドを格納するためのものです。コマンドバッファを作成するためには、コマンドプールも作成する必要があります。

```diff cpp
void initVulkan() {
    // ...
+   commandPool = vkutils::createCommandPool(*device, queueFamilyIndex);
+   commandBuffer = vkutils::createCommandBuffer(*device, *commandPool);
}
```

これで基本的なセットアップが完了しました。

[ここまでのC++コード(01_skip_base_setup.cpp)](https://github.com/nishidate-yuki/vulkan_raytracing_from_scratch/blob/master/code/01_skip_base_setup.cpp)
