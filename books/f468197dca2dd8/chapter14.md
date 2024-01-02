
この章はレイトレーシングと直接関係ないので詳しい解説はありません。同期の詳細は[Vulkan Tutorialの同期の項](https://vulkan-tutorial.com/Drawing_a_triangle/Drawing/Rendering_and_presentation)をご覧ください。

# 同期オブジェクトの作成

まずはランタイムで同時に利用できるフレームの数をクラスの上で定義します。

```cpp
const int MAX_FRAMES_IN_FLIGHT = 2;
```

さらに同期オブジェクトをメンバに追加します。
```cpp
std::vector<vk::UniqueSemaphore> imageAvailableSemaphores;
std::vector<vk::UniqueSemaphore> renderFinishedSemaphores;
std::vector<vk::Fence> inFlightFences;
std::vector<vk::Fence> imagesInFlight;
```

次に新しい関数を追加します。
```cpp
void initVulkan()
{
    createSyncObjects();
}

void createSyncObjects()
{
    
}
```

オブジェクトを作成します。上記のチュートリアルのコードと全く同じです。

```cpp
imageAvailableSemaphores.resize(MAX_FRAMES_IN_FLIGHT);
renderFinishedSemaphores.resize(MAX_FRAMES_IN_FLIGHT);
inFlightFences.resize(MAX_FRAMES_IN_FLIGHT);
imagesInFlight.resize(swapChainImages.size());

for (size_t i = 0; i < MAX_FRAMES_IN_FLIGHT; i++) {
    imageAvailableSemaphores[i] = device->createSemaphoreUnique({});
    renderFinishedSemaphores[i] = device->createSemaphoreUnique({});
    inFlightFences[i] = device->createFence(
        { vk::FenceCreateFlagBits::eSignaled }
    );
}
```

最後に`cleanup()`のなかで同期オブジェクトを破棄します。

```cpp
void cleanup()
{
    for (size_t i = 0; i < MAX_FRAMES_IN_FLIGHT; i++) {
        device->destroyFence(inFlightFences[i]);
    }

    glfwDestroyWindow(window);
    glfwTerminate();
}
```


この章はこれだけです。いよいよ次の章が最後の章です。

[ここまでのC++コード(https://github.com/nishidate-yuki/vulkan_raytracing_from_scratch/blob/master/code/11_create_sync_objects.cpp)](https://github.com/nishidate-yuki/vulkan_raytracing_from_scratch/blob/master/code/11_create_sync_objects.cpp)