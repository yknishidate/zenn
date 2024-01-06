---
title: "プロジェクト設定"
---

まずはプロジェクトを作成します。

# 準備

以下の要求を満たすように準備をしてください。

- Vulkan Ray Tracingに対応したGPUとドライバ
- [Vulkan SDK](https://vulkan.lunarg.com/#new_tab) 1.2.162 or later
- C++ 17対応コンパイラ
- CMake 3.19 or later
- vcpkg

# プロジェクト作成

以下のようにファイルを追加します。

```
project/
 - vcpkg.json
 - CMakeLists.txt
 - code/
   - main.cpp
   - vkutils.hpp
 - shaders/
   - (あとでファイルを追加)
```

記事で利用するヘルパー [vkutils.hpp](https://github.com/nishidate-yuki/vulkan_raytracing_from_scratch/blob/master/code/vkutils.hpp) はこのリンクからダウンロードできます。

## vcpkg と `vcpkg.json`

vcpkgはC++のパッケージマネージャで、この記事ではGLFWを追加するために利用します。多くのVulkanチュートリアルでは、GLFW を公式ページからダウンロードする方法が紹介されていますが、パッケージマネージャを使用すればGUI操作なしで簡単にインストールできます。

vcpkgでは通常の`vcpkg install xx`コマンドを使うとグローバルにインストールされてしまうため、`vcpkg.json`をプロジェクトに追加し、CMakeを走らせる際にローカルにインストールできるようにします。こうすることで、マシン環境を汚さずにライブラリを使用できます。この方式をマニフェストモードと呼びます。

```json:vcpkg.json
{
  "$schema": "https://raw.githubusercontent.com/microsoft/vcpkg/master/scripts/vcpkg.schema.json",
  "name": "vulkan-raytracing",
  "version-string": "0.1.0",
  "dependencies": [
    "glfw3"
  ]
}
```

## `CMakeLists.txt`

C++17を指定し、VulkanのヘッダーとGLFWを追加します。

```cmake:CMakeLists.txt
cmake_minimum_required(VERSION 3.19)

project(vulkan_raytracing)

set(CMAKE_CXX_STANDARD 17)

find_package(glfw3 CONFIG REQUIRED)

file(GLOB_RECURSE PROJECT_SOURCES "code/*.cpp")
file(GLOB_RECURSE PROJECT_HEADERS "code/*.hpp")
add_executable(${PROJECT_NAME} ${PROJECT_SOURCES} ${PROJECT_HEADERS})

# Lib
target_link_libraries(${PROJECT_NAME} PRIVATE glfw)

# Include
target_include_directories(${PROJECT_NAME} PRIVATE $ENV{VULKAN_SDK}/Include)

# Define
target_compile_definitions(${PROJECT_NAME} PRIVATE
    "SHADER_DIR=std::string{\"${PROJECT_SOURCE_DIR}/shaders/\"}"
)

# Set startup project
if(MSVC)
    set_property(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} 
                 PROPERTY VS_STARTUP_PROJECT ${PROJECT_NAME})
endif()
```

## CMakeを走らせる

準備ができたらCMakeを走らせます。環境変数に`VCPKG_ROOT`が追加されていることを確認しておいてください。環境変数の展開はWindows形式で記載しているので適宜読み替えてください。コマンドを実行すると、vcpkgによってGLFWがインストールされ、プロジェクトが作成されます。

```sh
# For windows
cmake . -B build -DCMAKE_TOOLCHAIN_FILE=%VCPKG_ROOT%/scripts/buildsystems/vcpkg.cmake
```

これでプロジェクト作成は終了です。実装に入っていきましょう。
