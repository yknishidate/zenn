---
title: "レイトレーシングパイプラインの作成"
---

# レイトレーシングパイプラインの作成

いよいよレイトレーシングパイプラインを作成していきます。まずは `createRayTracingPipeline()` を作成して `initVulkan()` から呼び出します。

```cpp
class Application
{
    void initVulkan() {
        // ...
        createRayTracingPipeline();
    }

    createRayTracingPipeline() {

    }
};
```

パイプラインレイアウトを作成したのち、パイプラインを作成します。パイプラインはシェーダの情報を渡し、レイの最大再帰深度を指定します。ここでは`1`にしておきます。パフォーマンスの観点から、再帰深度はなるべく浅くするべきであるとされています。

```cpp
void createRayTracingPipeline() {
    std::cout << "Create pipeline\n";

    // Create pipeline layout
    vk::PipelineLayoutCreateInfo layoutCreateInfo{};
    layoutCreateInfo.setSetLayouts(*descSetLayout);
    pipelineLayout = device->createPipelineLayoutUnique(layoutCreateInfo);

    // Create pipeline
    vk::RayTracingPipelineCreateInfoKHR pipelineCreateInfo{};
    pipelineCreateInfo.setLayout(*pipelineLayout);
    pipelineCreateInfo.setStages(shaderStages);
    pipelineCreateInfo.setGroups(shaderGroups);
    pipelineCreateInfo.setMaxPipelineRayRecursionDepth(1);
    auto result = device->createRayTracingPipelineKHRUnique(
        nullptr, nullptr, pipelineCreateInfo);
    if (result.result != vk::Result::eSuccess) {
        std::cerr << "Failed to create ray tracing pipeline.\n";
        std::abort();
    }
    pipeline = std::move(result.value);
}
```

[ここまでのC++コード(08_create_pipeline.hpp)](https://github.com/nishidate-yuki/vulkan_raytracing_from_scratch/blob/master/code/08_create_pipeline.hpp)

[シェーダファイル](https://github.com/nishidate-yuki/vulkan_raytracing_from_scratch/tree/master/shaders)
