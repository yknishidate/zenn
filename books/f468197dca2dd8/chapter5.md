---
title: "スワップチェーンの作成"
---

この章では、スワップチェーンを作成します。

# 必要なメンバー変数を追加

スワップチェーンと、その中に含まれるイメージを保持するためのメンバー変数を追加します。

```cpp
private:
    // ...
    vk::SurfaceFormatKHR surfaceFormat;
    vk::UniqueSwapchainKHR swapchain;
    std::vector<vk::Image> swapchainImages;
    std::vector<vk::UniqueImageView> swapchainImageViews;
```

# スワップチェーンの作成

サーフェスから利用可能なフォーマットを取得したのち、スワップチェーンを作成するために`vkutils::createSwapchain()`を呼び出します。

通常、ラスタライズパイプラインで描画する場合は、スワップチェーンのイメージは`vk::ImageUsageFlagBits::eColorAttachment`を指定します。しかし、レイトレーシングパイプラインでは、シェーダの中でストレージイメージに対して色を書きこむため、`vk::ImageUsageFlagBits::eStorage`を指定することに注意してください。

スワップチェーンを作成したら、その中に含まれるイメージを取得します。

```cpp
void initVulkan() {
    // ...
    surfaceFormat = vkutils::chooseSurfaceFormat(physicalDevice, *surface);
    swapchain = vkutils::createSwapchain(
        physicalDevice, *device, *surface, queueFamilyIndex,
        vk::ImageUsageFlagBits::eStorage, surfaceFormat,
        WIDTH, HEIGHT);
    swapchainImages = device->getSwapchainImagesKHR(*swapchain);
}
```

# イメージビューの作成

イメージビューは、イメージを読み書きするためのオブジェクトです。イメージビューを作成するために、`createSwapchainImageViews()`という関数を作成します。

```cpp
void initVulkan() {
    // ...
    createSwapchainImageViews();
}

void createSwapchainImageViews() {
    // ここにコードを追加
}
```

まずは、スワップチェーンの画像ごとにイメージビューを作成します。

```cpp
void createSwapchainImageViews() {
    for (auto image : swapchainImages) {
        vk::ImageViewCreateInfo createInfo{};
        createInfo.setImage(image);
        createInfo.setViewType(vk::ImageViewType::e2D);
        createInfo.setFormat(surfaceFormat.format);
        createInfo.setComponents(
            {vk::ComponentSwizzle::eR, vk::ComponentSwizzle::eG,
                vk::ComponentSwizzle::eB, vk::ComponentSwizzle::eA});
        createInfo.setSubresourceRange(
            {vk::ImageAspectFlagBits::eColor, 0, 1, 0, 1});
        swapchainImageViews.push_back(
            device->createImageViewUnique(createInfo));
    }
}
```

イメージビューはレイアウトという状態を持ち、その状態によってイメージをどのように扱うかが決まります。ここでは表示用としておくため、`vk::ImageLayout::ePresentSrcKHR`に変更します。

```cpp
void createSwapchainImageViews() {
    // ...

    vkutils::oneTimeSubmit(*device, *commandPool, queue,
        [&](vk::CommandBuffer commandBuffer) {
            for (auto image : swapchainImages) {
                vkutils::setImageLayout(commandBuffer, image,
                                        vk::ImageLayout::eUndefined,
                                        vk::ImageLayout::ePresentSrcKHR);
            }
        });
}
```

以上でスワップチェーンの作成が完了しました。

[ここまでのC++コード(02_create_swapchain.hpp)](https://github.com/nishidate-yuki/vulkan_raytracing_from_scratch/blob/master/code/02_create_swapchain.hpp)
