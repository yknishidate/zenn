---
title: "三角形の描画"
---

いよいよ最後の章です。

# drawFrame()を追加

まずは、メインループに描画用の関数を追加します。

```cpp
void run() {
    // ...

    while (!glfwWindowShouldClose(window)) {
        glfwPollEvents();
        drawFrame();
    }

    // ...
}

void drawFrame() {
    static int frame = 0;
    std::cout << frame << '\n';
    frame++;
}
```

## スワップチェーンから画像を取得

描画の際には、スワップチェーンから描画対象となる画像を取得し、それに対する描画コマンドをサブミットします。画像の用意ができてから描画を行う必要があるため、同期用のセマフォを作成し、`vk::Device::acquireNextImageKHR()`で画像インデックスを取得します。

```cpp
void drawFrame() {
    // ...

    // Create semaphore
    vk::UniqueSemaphore imageAvailableSemaphore =
        device->createSemaphoreUnique({});

    // Acquire next image
    auto result = device->acquireNextImageKHR(
        *swapchain, std::numeric_limits<uint64_t>::max(),
        *imageAvailableSemaphore);
    if (result.result != vk::Result::eSuccess) {
        std::cerr << "Failed to acquire next image.\n";
        std::abort();
    }

    // Update descriptor sets using current image
    uint32_t imageIndex = result.value;
}
```

## ディスクリプタセットの更新

この記事では、スワップチェーンの画像に対してシェーダから直接書き込みます。そのため、各フレームでスワップチェーンの中の描画対象となる画像を交代しながらバインドします。そのため、各フレームでディスクリプタセットの内容を更新する必要があります。シェーダでは`binding=0`にTLASを、`binding=1`にストレージイメージをバインドしていたので、それに合わせて`updateDescriptorSet()`を作成します。一般的なバッファやイメージには専用の`setBufferInfo()`や`setImageInfo()`が用意されていますが、TLASは拡張機能を使っているため、`setPNext()`で渡す点に注意してください。

```cpp
void updateDescriptorSet(vk::ImageView imageView) {
    std::vector<vk::WriteDescriptorSet> writes(2);

    // [0]: For AS
    vk::WriteDescriptorSetAccelerationStructureKHR accelInfo{};
    accelInfo.setAccelerationStructures(*topAccel.accel);
    writes[0].setDstSet(*descSet);
    writes[0].setDstBinding(0);
    writes[0].setDescriptorCount(1);
    writes[0].setDescriptorType(
        vk::DescriptorType::eAccelerationStructureKHR);
    writes[0].setPNext(&accelInfo);

    // [0]: For storage image
    vk::DescriptorImageInfo imageInfo{};
    imageInfo.setImageView(imageView);
    imageInfo.setImageLayout(vk::ImageLayout::eGeneral);
    writes[1].setDstSet(*descSet);
    writes[1].setDstBinding(1);
    writes[1].setDescriptorType(vk::DescriptorType::eStorageImage);
    writes[1].setImageInfo(imageInfo);

    // Update
    device->updateDescriptorSets(writes, nullptr);
}

void drawFrame() {
    // ...

    updateDescriptorSet(*swapchainImageViews[imageIndex]);
}
```

## コマンドバッファを記録

GPUにサブミットするコマンドバッファを記録します。スワップチェーンの画像は元々表示用に設定したので、一度レイアウトを書きこみ可能な`vk::ImageLayout::eGeneral`に変更します。その後、レイトレーシングパイプラインとディスクリプタセットをバインドします。そして`vk::CommandBuffer::traceRaysKHR()`でレイトレーシングを画像のピクセルサイズに合わせて実行し、最後に画像を表示用のレイアウトに戻します。

```cpp
void recordCommandBuffer(vk::Image image) {
    // Begin
    commandBuffer->begin(vk::CommandBufferBeginInfo{});

    // Set image layout to general
    vkutils::setImageLayout(*commandBuffer, image,  //
                            vk::ImageLayout::ePresentSrcKHR,
                            vk::ImageLayout::eGeneral);

    // Bind pipeline
    commandBuffer->bindPipeline(vk::PipelineBindPoint::eRayTracingKHR,
                                *pipeline);

    // Bind desc set
    commandBuffer->bindDescriptorSets(
        vk::PipelineBindPoint::eRayTracingKHR,  // pipelineBindPoint
        *pipelineLayout,                        // layout
        0,                                      // firstSet
        *descSet,                               // descSets
        nullptr                                 // dynamicOffsets
    );

    // Trace rays
    commandBuffer->traceRaysKHR(  //
        raygenRegion,             // raygen
        missRegion,               // miss
        hitRegion,                // hit
        {},                       // callable
        WIDTH, HEIGHT, 1          // width, height, depth
    );

    // Set image layout to present src
    vkutils::setImageLayout(*commandBuffer, image,  //
                            vk::ImageLayout::eGeneral,
                            vk::ImageLayout::ePresentSrcKHR);

    // End
    commandBuffer->end();
}

void drawFrame() {
    // ...

    recordCommandBuffer(swapchainImages[imageIndex]);
}
```

## サブミットと待機

画像を待つためのセマフォを指定してコマンドバッファをサブミットし、GPUの処理が終わるまで待機します。もちろん実用的なアプリケーションでは、待機せずに次のフレームの描画を開始するのがいいでしょう。この辺りは[Vulkan Tutorial](https://vulkan-tutorial.com/Drawing_a_triangle/Drawing/Frames_in_flight)に詳しく書かれています。

```cpp
void drawFrame() {
    // ...

    // Submit command buffer
    vk::PipelineStageFlags waitStage{vk::PipelineStageFlagBits::eTopOfPipe};
    vk::SubmitInfo submitInfo{};
    submitInfo.setWaitDstStageMask(waitStage);
    submitInfo.setCommandBuffers(*commandBuffer);
    submitInfo.setWaitSemaphores(*imageAvailableSemaphore);
    queue.submit(submitInfo);

    // Wait
    queue.waitIdle();

    // Present
    vk::PresentInfoKHR presentInfo{};
    presentInfo.setSwapchains(*swapchain);
    presentInfo.setImageIndices(imageIndex);
    if (queue.presentKHR(presentInfo) != vk::Result::eSuccess) {
        std::cerr << "Failed to present.\n";
        std::abort();
    }
}
```

最後に描画した画像を表示します。

```cpp
void drawFrame() {
    // ...

    // Present
    vk::PresentInfoKHR presentInfo{};
    presentInfo.setSwapchains(*swapchain);
    presentInfo.setImageIndices(imageIndex);
    if (queue.presentKHR(presentInfo) != vk::Result::eSuccess) {
        std::cerr << "Failed to present.\n";
        std::abort();
    }
}
```

# 実行

実行してみると、三角形が表示されるはずです。

![](https://storage.googleapis.com/zenn-user-upload/rr5crszad0xyh2a33lxmbh21gd0u)


[最終的なC++コード(10_draw_triangle.hpp)](https://github.com/nishidate-yuki/vulkan_raytracing_from_scratch/blob/master/code/10_draw_triangle.hpp)

[シェーダファイル](https://github.com/nishidate-yuki/vulkan_raytracing_from_scratch/tree/master/shaders)

# まとめ

以上で、Vulkan Ray Tracingの基本的な入門は終了です。まだまだ発展的な内容はありますが、Vulkan Ray Tracingの基本的な概念は理解できたのではないでしょうか。もし記事にミスがあったり、説明が不足している部分があれば、Zennにコメントしていただくか、GitHubで編集提案していただくか、X(twitter.com/yknishidate)に連絡していただけると幸いです。また、分からないことがあれば、Xに連絡していただければ分かる範囲でお答えします。それでは、最後まで読んでいただきありがとうございました。

# おすすめ

Vulkan Ray Tracingについてさらに詳しく知りたい方は、てっくまさんの[Vulkan Raytracing Programming Vol.1](https://booth.pm/ja/items/3564766)をおすすめします。各関数のパラメータまで詳説しており、発展的内容も多く含まれています。
