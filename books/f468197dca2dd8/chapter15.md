
この記事最後の章です。この章では実際に描画ループを作っていきます。

先にメンバ変数にフレームを追加します。

```cpp
size_t currentFrame = 0;
```

次に描画用の関数`drawFrame()`を作成し、`mainLoop()`の中で呼び出します。

```cpp
void drawFrame()
{
    
}

void mainLoop()
{
    while (!glfwWindowShouldClose(window)) {
        glfwPollEvents();
        drawFrame();
    }
}
```

描画の最初は、前回の章で作成した同期オブジェクトのフェンスで待機します。`std::numeric_limits<uint64_t>::max()`を指定しているのはタイムアウトの引数です。この値はタイムアウトを無効にする特別な値です。

```cpp
device->waitForFences(inFlightFences[currentFrame], true, std::numeric_limits<uint64_t>::max());
```

# スワップチェインから画像を取得

待機が終了したら、まずは`acquireNextImageKHR()`でスワップチェインから次に描画するイメージのインデックスを取得します。

```cpp
auto result = device->acquireNextImageKHR(
    swapChain.get(),                             // swapchain
    std::numeric_limits<uint64_t>::max(),        // timeout
    imageAvailableSemaphores[currentFrame].get() // semaphore
);
uint32_t imageIndex;
if (result.result == vk::Result::eSuccess) {
    imageIndex = result.value;
} else {
    throw std::runtime_error("failed to acquire next image!");
}
```

ただし、前のフレームがまだこの画像を使用しているかもしれないので、`imagesInFlight`を確認し、フェンスがあればここでも待機します。

```cpp
if (imagesInFlight[imageIndex] != VK_NULL_HANDLE) {
    device->waitForFences(imagesInFlight[imageIndex], true, std::numeric_limits<uint64_t>::max());
}
imagesInFlight[imageIndex] = inFlightFences[currentFrame];

device->resetFences(inFlightFences[currentFrame]);
```

待機したあとは現在のフレームでこの画像は使用中だと宣言するために`imagesInFlight`にフェンスを入れています。

# レイトレーシングを実行

前々回に構築したコマンドバッファを、グラフィックスキューに対してサブミットします。これでレイトレーシングが実行されます。

```cpp
vk::PipelineStageFlags waitStage{ vk::PipelineStageFlagBits::eRayTracingShaderKHR };
graphicsQueue.submit(
    vk::SubmitInfo{}
    .setWaitSemaphores(imageAvailableSemaphores[currentFrame].get())
    .setWaitDstStageMask(waitStage)
    .setCommandBuffers(drawCommandBuffers[imageIndex].get())
    .setSignalSemaphores(renderFinishedSemaphores[currentFrame].get()),
    inFlightFences[currentFrame]
);
```

そして、それを表示します。
```cpp
graphicsQueue.presentKHR(
    vk::PresentInfoKHR{}
    .setWaitSemaphores(renderFinishedSemaphores[currentFrame].get())
    .setSwapchains(swapChain.get())
    .setImageIndices(imageIndex)
);
```

関数の最後でcurrentFrameを更新しておきます。

```cpp
currentFrame = (currentFrame + 1) % MAX_FRAMES_IN_FLIGHT;
```

これで`drawFrame()`は完成ですが、一度`mainLoop()`に移動して、ループ終了時に`waitIdle()`するように記述します。なにかリソースを使用している最中に突然終了するとエラーが発生するためです。

```cpp
void mainLoop()
{
    while (!glfwWindowShouldClose(window)) {
        glfwPollEvents();
        drawFrame();
    }
    device->waitIdle();
}
```

これで実行してみると、、

![](https://storage.googleapis.com/zenn-user-upload/3tuuxq43p3lp1twlgtjukyjg5l74)

三角形が描画されるはずです。

[最終的なC++コード(12_hello_triangle.cpp)](https://github.com/nishidate-yuki/vulkan_raytracing_from_scratch/blob/master/code/12_hello_triangle.cpp)


# おわりに

ここまで読んでくださった方、本当にありがとうございました。なにか不具合や間違った記述等があれば、コメントや[Twitter(@asta18425)](https://twitter.com/asta18425)にお願い致します。

この記事が誰かの参考になることを願っています。

