---
title: "シェーダの作成"
---

# シェーダの作成

ここで一度メインのプログラムを離れ、シェーダを書いていきます。

今回使用するシェーダは3つです。

1. Raygenシェーダ
2. Closest Hitシェーダ
3. Missシェーダ

まずはプロジェクトに`shaders`フォルダを作成し、その下にシェーダファイルを置くこととします。

```
project/
 - vcpkg.json
 - CMakeLists.txt
 - code/
   - main.cpp
   - vkutils.hpp
 - shaders/
   - raygen.rgen (new)
   - closesthit.rchit (new)
   - miss.rmiss (new)
```

## Raygenシェーダ

Raygenシェーダはレイトレーシングパイプラインの起点となるシェーダです。レイを飛ばす方向を決め、トレースを開始します。

```glsl:raygen.rgen
#version 460
#extension GL_EXT_ray_tracing : enable

layout(location = 0) rayPayloadEXT vec3 payload;

layout(binding = 0) uniform accelerationStructureEXT topLevelAS;
layout(binding = 1, rgba8) uniform image2D image;

void main()
{
    vec2 uv = (vec2(gl_LaunchIDEXT.xy) + vec2(0.5)) / vec2(gl_LaunchSizeEXT.xy);
    vec3 origin = vec3(0, 0, 5);
    vec3 target = vec3(uv * 2.0 - 1.0, 2);
    vec3 direction = normalize(target - origin);

    payload = vec3(0.0);

    traceRayEXT(
        topLevelAS,
        gl_RayFlagsOpaqueEXT,
        0xff,       // cullMask
        0, 0, 0,    // sbtRecordOffset, sbtRecordStride, missIndex
        origin,
        0.001,      // tMin
        direction,
        10000.0,    // tMax
        0           // payloadLocation
    );

    imageStore(image, ivec2(gl_LaunchIDEXT.xy), vec4(payload, 0.0));
}
```

レイトレーシング用のシェーダでは、先頭に`#extension GL_EXT_ray_tracing : enable`という行を追加します。`gl_LaunchIDEXT`という変数を使ってシェーダの起動IDを取得し、レイの原点と方向を計算しています。`traceRayEXT()`関数でレイを飛ばし、`rayPayloadEXT`に指定したペイロード変数で他のシェーダによる結果を受け取ります。今回は単純に色を表す`vec3`を受け取り、`imageStore()`関数で画像に書き込んでいます。

## Closest Hitシェーダ

Closest Hitシェーダはレイがヒットした点のうち、最も近い点で実行されるシェーダです。今回は三角形内の重心座標を計算して`payload`に保存します。

```glsl:closesthit.rchit
#version 460
#extension GL_EXT_ray_tracing : enable

layout(location = 0) rayPayloadInEXT vec3 payload;
hitAttributeEXT vec3 attribs;

void main()
{
    vec3 baryCoords = vec3(1.0 - attribs.x - attribs.y, attribs.x, attribs.y);
    payload = baryCoords;
}
```


## Missシェーダ

Missシェーダはレイがヒットしなかった場合に実行されるシェーダです。今回は背景色を`payload`に保存します。

```glsl:minn.rmiss
#version 460
#extension GL_EXT_ray_tracing : enable

layout(location = 0) rayPayloadInEXT vec3 payLoad;

void main()
{
    payLoad = vec3(0.0, 0.5, 0.2);
}
```

# シェーダのコンパイル

今回は glslangValidator を使ってGLSLをSPIR-V（スピアブイ）形式にコンパイルします。

Windows環境では、`shaders`ディレクトリに以下のようなバッチファイルを作ります。異なるプラットフォームの場合は適宜読み替えて下さい。

```sh:compile.bat
@echo off
set GLSLANG_VALIDATOR=%VULKAN_SDK%/Bin/glslangValidator.exe

for %%s in (raygen.rgen closesthit.rchit miss.rmiss) do (
    %GLSLANG_VALIDATOR% %%s -V -o %%s.spv --target-env vulkan1.2
)
```

これを実行して3つの`spv`ファイルが作成されていることを確認します。

```sh
project/
  - shaders/
    - raygen.rgen
    - closesthit.rchit
    - miss.rmiss
    - raygen.rgen.spv (new)
    - closesthit.rchit.spv (new)
    - miss.rmiss.spv (new)
```

# シェーダ読み込み

ではC++に戻ります。

それぞれのシェーダを読み込んだのち、`vk::ShaderModule`を作成します。シェーダは複数あるので配列にしておきます。パイプラインを作成する際に必要となるその他の情報も配列で用意しておきます。

```cpp
class Application {
    // ...
    std::vector<vk::UniqueShaderModule> shaderModules;
    std::vector<vk::PipelineShaderStageCreateInfo> shaderStages;
    std::vector<vk::RayTracingShaderGroupCreateInfoKHR> shaderGroups;
    // ...
};
```

シェーダを読み込んでいきます。まず、`prepareShaders()`関数を作成し、`initVulkan()`から呼び出します。先ほど追加したメンバ変数をリサイズしておきます。

```cpp
void initVulkan() {
    // ...
    prepareShaders();
}

void prepareShaders() {
}
```

3つのシェーダで似たような処理を行うので、ヘルパー関数を作成してまとめておきます。ここではシェーダモジュールを作成したあと、そのシェーダのステージ情報を作成し、シェーダグループにセットします。

```cpp
void addShader(uint32_t shaderIndex,
                uint32_t groupIndex,
                const std::string& filename,
                vk::ShaderStageFlagBits stage) {
    shaderModules[shaderIndex] =
        vkutils::createShaderModule(*device, SHADER_DIR + filename);

    shaderStages[shaderIndex].setStage(stage);
    shaderStages[shaderIndex].setModule(*shaderModules[shaderIndex]);
    shaderStages[shaderIndex].setPName("main");

    shaderGroups[groupIndex].setGeneralShader(VK_SHADER_UNUSED_KHR);
    shaderGroups[groupIndex].setClosestHitShader(VK_SHADER_UNUSED_KHR);
    shaderGroups[groupIndex].setAnyHitShader(VK_SHADER_UNUSED_KHR);
    shaderGroups[groupIndex].setIntersectionShader(VK_SHADER_UNUSED_KHR);

    switch (stage) {
        case vk::ShaderStageFlagBits::eRaygenKHR:
        case vk::ShaderStageFlagBits::eMissKHR:
            shaderGroups[groupIndex].setType(
                vk::RayTracingShaderGroupTypeKHR::eGeneral);
            shaderGroups[groupIndex].setGeneralShader(shaderIndex);
            break;
        case vk::ShaderStageFlagBits::eClosestHitKHR:
            shaderGroups[groupIndex].setType(
                vk::RayTracingShaderGroupTypeKHR::eTrianglesHitGroup);
            shaderGroups[groupIndex].setClosestHitShader(shaderIndex);
            break;
        default:
            break;
    }
}

void prepareShaders() {
    shaderStages.resize(3);
    shaderModules.resize(3);
    shaderGroups.resize(3);

    addShader(0, 0, "raygen.rgen.spv",  //
                vk::ShaderStageFlagBits::eRaygenKHR);
    addShader(1, 1, "miss.rmiss.spv",  //
                vk::ShaderStageFlagBits::eMissKHR);
    addShader(2, 2, "closesthit.rchit.spv",
                vk::ShaderStageFlagBits::eClosestHitKHR);
}
```

シェーダステージとシェーダグループはこの記事では両方とも3つですが、数は異なる場合があるので注意してください。例えば、一つのヒットグループにClosest hitシェーダとAny hitシェーダの2つが含まれることもあります。

以上でこの章は完了です。

[ここまでのC++コード(06_create_shader.hpp)](https://github.com/nishidate-yuki/vulkan_raytracing_from_scratch/blob/master/code/06_create_shader.hpp)

[シェーダファイル](https://github.com/nishidate-yuki/vulkan_raytracing_from_scratch/tree/master/shaders)
