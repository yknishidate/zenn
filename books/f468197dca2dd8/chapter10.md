---
title: "レイトレーシングパイプラインの作成"
---

ラスタライズではグラフィックスパイプラインを作成しますが、レイトレーシングでは全く異なる**レイトレーシングパイプライン**を作成する必要があります。

大まかな流れは以下のようになります。

1. ディスクリプタセットレイアウトを作成
2. パイプラインレイアウトを作成
3. シェーダを作成・コンパイル
4. シェーダ読み込み
5. シェーダグループの設定
6. レイトレーシングパイプラインを作成

---

まずはパイプライン、パイプラインレイアウト、ディスクリプタレイアウトをメンバ変数を追加します。

```cpp
vk::UniquePipeline pipeline;
vk::UniquePipelineLayout pipelineLayout;
vk::UniqueDescriptorSetLayout descriptorSetLayout;
```

次に`createRayTracingPipeLine()`を追加して`initVulkan()`で呼び出します。

```cpp
void initVulkan()
{
    ...
    createRayTracingPipeLine();
}

void createRayTracingPipeLine()
{
    
}
```

# ディスクリプタセットレイアウトを作成

ディスクリプタセットを作成するには、その中の`vk::DescriptorSetLayoutBinding`を用意する必要があります。

今回使用するものは次の2つです。

- トップレベルアクセラレーション構造
- 結果を保存するストレージイメージ

それぞれバインディングを`0`と`1`に設定します。

```cpp
vk::DescriptorSetLayoutBinding accelerationStructureLayoutBinding{};
accelerationStructureLayoutBinding
    .setBinding(0)
    .setDescriptorType(vk::DescriptorType::eAccelerationStructureKHR)
    .setDescriptorCount(1)
    .setStageFlags(vk::ShaderStageFlagBits::eRaygenKHR);

vk::DescriptorSetLayoutBinding resultImageLayoutBinding{};
resultImageLayoutBinding
    .setBinding(1)
    .setDescriptorType(vk::DescriptorType::eStorageImage)
    .setDescriptorCount(1)
    .setStageFlags(vk::ShaderStageFlagBits::eRaygenKHR);
```

こうすることでシェーダからは下のような形でアクセスできるようにします。

```glsl:example.rgen
layout(binding = 0) uniform accelerationStructureEXT topLevelAS;
layout(binding = 1) uniform image2D image;
```

次に2つをまとめてディスクリプタセットレイアウトを作成します。

```cpp
std::vector<vk::DescriptorSetLayoutBinding> binding{ accelerationStructureLayoutBinding, resultImageLayoutBinding };
descriptorSetLayout = device->createDescriptorSetLayoutUnique(
    vk::DescriptorSetLayoutCreateInfo{}
    .setBindings(binding)
);
```

# パイプラインレイアウトを作成

さらにそれを使ってパイプラインレイアウトを作成します。パイプラインレイアウトには複数のディスクリプタセットレイアウトを含むことが出来ますが、ここでは1つだけです。

```cpp
pipelineLayout = device->createPipelineLayoutUnique(
    vk::PipelineLayoutCreateInfo{}
    .setSetLayouts(descriptorSetLayout.get())
);
```

# シェーダの作成・コンパイル

ここで一度メインのプログラムを離れ、シェーダを書いていきます。

今回使用するシェーダは3つです。

1. Raygenシェーダ
2. Closest Hitシェーダ
3. Missシェーダ

まずはプロジェクトに`shaders`フォルダを作成し、その下にシェーダファイルを置くこととします。

```
--- project directory/
    |--- main.cpp
    |--- vkutils.hpp
    |--- shaders/
         |--- raygen.rgen
	 |--- closesthit.rchit
	 |--- minn.rmiss
```


## Raygenシェーダ

さっそくシェーダを書いていきます。

```glsl:raygen.rgen
#version 460
#extension GL_EXT_ray_tracing : enable

layout(binding = 0, set = 0) uniform accelerationStructureEXT topLevelAS;
layout(binding = 1, set = 0, rgba8) uniform image2D image;

layout(location = 0) rayPayloadEXT vec3 payLoad;

void main()
{
	const vec2 pixelCenter = vec2(gl_LaunchIDEXT.xy) + vec2(0.5);
	const vec2 inUV = pixelCenter/vec2(gl_LaunchSizeEXT.xy);
	vec2 d = inUV * 2.0 - 1.0;

	vec4 origin = vec4(0, 0, 5, 1);
	vec4 target = vec4(d.x, d.y, 2, 1) ;
	vec4 direction = vec4(normalize(target.xyz - origin.xyz), 0) ;

	float tmin = 0.001;
	float tmax = 10000.0;

    payLoad = vec3(0.0);

    traceRayEXT(
        topLevelAS,
        gl_RayFlagsOpaqueEXT,
        0xff, // cullMask
        0,    // sbtRecordOffset
        0,    // sbtRecordStride
        0,    // missIndex
        origin.xyz,
        tmin,
        direction.xyz,
        tmax,
        0     // payloadLocation
    );

	imageStore(image, ivec2(gl_LaunchIDEXT.xy), vec4(payLoad, 0.0));
}
```

先ほど説明した`binding`でリソースにアクセスしていることが分かります。

`main()`の内容は

- レイを飛ばす方向を決める
- `traceRayEXT`でレイをトレースする
- トレースした結果が格納された`payLoad`を`image`に保存する

という流れになっています。


## Closest Hitシェーダ
```glsl:closesthit.rchit
#version 460
#extension GL_EXT_ray_tracing : enable

layout(location = 0) rayPayloadInEXT vec3 payLoad;
hitAttributeEXT vec3 attribs;

void main()
{
  const vec3 barycentricCoords = vec3(1.0f - attribs.x - attribs.y, attribs.x, attribs.y);
  payLoad = barycentricCoords;
}
```

`barycentricCoords`というのは日本語では重心座標です。ヒットした三角形内の重心座標を計算して`payLoad`に保存しています。

## Missシェーダ
```glsl:minn.rmiss
#version 460
#extension GL_EXT_ray_tracing : enable

layout(location = 0) rayPayloadInEXT vec3 payLoad;

void main()
{
    payLoad = vec3(0.0, 0.5, 0.2);
}
```

このシェーダは三角形にヒットしなかった場合に実行されるので、この場合は背景色を決めていることになります。

## コンパイル

今回は`glslc.exe`を使って手動でシェーダのコンパイルを行います。

Windows環境でVulkan SDKをインストールすると`VULKAN_SDK`という環境変数が追加されているはずなので、`shaders`フォルダから以下のようにコンパイルできます。異なるプラットフォームの場合は適宜読み替えて下さい。

```
%VULKAN_SDK%/Bin/glslc.exe raygen.rgen -o raygen.rgen.spv --target-env=vulkan1.2
%VULKAN_SDK%/Bin/glslc.exe closesthit.rchit -o closesthit.rchit.spv --target-env=vulkan1.2
%VULKAN_SDK%/Bin/glslc.exe miss.rmiss -o miss.rmiss.spv --target-env=vulkan1.2
```

これを実行して3つの`spv`ファイルが作成されていることを確認します。

# シェーダ読み込み

メインプログラムの`createRayTracingPipeLine()`に戻ります。

読み込むシェーダはシェーダモジュールという構造体にラップする必要があります。全てのシェーダを保持するためにシェーダモジュールの配列を作成します。

```cpp
std::vector<vk::UniqueShaderModule> shaderModules;
```

さらにそれぞれのモジュールがRaygenなのかMissなのかClosestHitなのか、といった情報を格納するシェーダステージの配列も作成します。配列の中のインデックスを表す変数も置いておきます。

```cpp
std::array<vk::PipelineShaderStageCreateInfo, 3> shaderStages;
const uint32_t shaderIndexRaygen = 0;
const uint32_t shaderIndexMiss = 1;
const uint32_t shaderIndexClosestHit = 2;
```

さらに次章のシェーダバインディングテーブルを作成する際に必要なシェーダグループの配列も必要です。こちらは関数内の変数ではなく、メンバ変数として追加します。

```cpp
private:
    ...
    
    std::vector<vk::RayTracingShaderGroupCreateInfoKHR> shaderGroups;
```

用意ができたのでシェーダを読み込んでいきます。

まずはRaygenシェーダです。

```cpp
shaderModules.push_back(vkutils::createShaderModule(device.get(), "shaders/raygen.rgen.spv"));
shaderStages[shaderIndexRaygen] =
    vk::PipelineShaderStageCreateInfo{}
    .setStage(vk::ShaderStageFlagBits::eRaygenKHR)
    .setModule(shaderModules.back().get())
    .setPName("main");
shaderGroups.push_back(
    vk::RayTracingShaderGroupCreateInfoKHR{}
    .setType(vk::RayTracingShaderGroupTypeKHR::eGeneral)
    .setGeneralShader(shaderIndexRaygen)
    .setClosestHitShader(VK_SHADER_UNUSED_KHR)
    .setAnyHitShader(VK_SHADER_UNUSED_KHR)
    .setIntersectionShader(VK_SHADER_UNUSED_KHR)
);
```

ファイルのロードとシェーダモジュールの作成はヘルパーに用意されている関数を使います。作成したシェーダモジュールは配列に追加し、`Raygen`シェーダであることを示すシェーダステージも作成して配列内の`shaderIndexRaygen`の場所に入れておきます。
さらにシェーダグループの配列にも構造体を追加します。こちらはシェーダグループタイプとインデックスを指定します。

同様にMissシェーダとClosestシェーダも読み込みます。

```cpp
shaderModules.push_back(vkutils::createShaderModule(device.get(), "shaders/miss.rmiss.spv"));
shaderStages[shaderIndexMiss] =
    vk::PipelineShaderStageCreateInfo{}
    .setStage(vk::ShaderStageFlagBits::eMissKHR)
    .setModule(shaderModules.back().get())
    .setPName("main");
shaderGroups.push_back(
    vk::RayTracingShaderGroupCreateInfoKHR{}
    .setType(vk::RayTracingShaderGroupTypeKHR::eGeneral)
    .setGeneralShader(shaderIndexMiss)
    .setClosestHitShader(VK_SHADER_UNUSED_KHR)
    .setAnyHitShader(VK_SHADER_UNUSED_KHR)
    .setIntersectionShader(VK_SHADER_UNUSED_KHR)
);

shaderModules.push_back(vkutils::createShaderModule(device.get(), "shaders/closesthit.rchit.spv"));
shaderStages[shaderIndexClosestHit] =
    vk::PipelineShaderStageCreateInfo{}
    .setStage(vk::ShaderStageFlagBits::eClosestHitKHR)
    .setModule(shaderModules.back().get())
    .setPName("main");
shaderGroups.push_back(
    vk::RayTracingShaderGroupCreateInfoKHR{}
    .setType(vk::RayTracingShaderGroupTypeKHR::eTrianglesHitGroup)
    .setGeneralShader(VK_SHADER_UNUSED_KHR)
    .setClosestHitShader(shaderIndexClosestHit)
    .setAnyHitShader(VK_SHADER_UNUSED_KHR)
    .setIntersectionShader(VK_SHADER_UNUSED_KHR)
);
```


これでレイトレーシングパイプライン作成の準備が全て終了しました。最後に`createRayTracingPipelineKHR()`で作成します。

```cpp
auto result = device->createRayTracingPipelineKHRUnique(nullptr, nullptr,
    vk::RayTracingPipelineCreateInfoKHR{}
    .setStages(shaderStages)
    .setGroups(shaderGroups)
    .setMaxPipelineRayRecursionDepth(1)
    .setLayout(pipelineLayout.get())
);
if (result.result == vk::Result::eSuccess) {
    pipeline = std::move(result.value);
} else {
    throw std::runtime_error("failed to create ray tracing pipeline.");
}
```


以上でこの章は完了です。次の章ではシェーダバインディングテーブルを作成していきます。

[ここまでのC++コード(07_create_ray_tracing_pipeline.cpp)](https://github.com/nishidate-yuki/vulkan_raytracing_from_scratch/blob/master/code/07_create_ray_tracing_pipeline.cpp)

[シェーダファイル](https://github.com/nishidate-yuki/vulkan_raytracing_from_scratch/tree/master/shaders)
