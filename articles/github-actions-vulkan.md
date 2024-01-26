---
title: "GitHub ActionsでVulkanプロジェクトをビルドする"
emoji: "🌋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [vulkan, cpp, githubactions]
published: true
---

CMake Presets、vcpkg manifest mode、Vulkan SDKを使ってVulkanプロジェクトをビルドする方法を紹介します。環境はWindowsとします。

# CMakeとvcpkgを使ったビルド

`CMakeLists.txt`と`vcpkg.json`がプロジェクトに存在していることを前提として、基本的なビルドの流れを書きます。

`CMakePresets.json`でActions用のプリセットを用意します。ここでは`github`という名前を付けました。vcpkgを使うように設定していますが、GitHub Actionsでは`VCPKG_ROOT`という環境変数はないため、`VCPKG_INSTALLATION_ROOT`という変数を使う必要があります。

```json:CMakePresets.json
{
    "version": 3,
    "cmakeMinimumRequired": {
        "major": 3,
        "minor": 19,
        "patch": 0
    },
    "configurePresets": [
        {
            "name": "github",
            "hidden": false,
            "generator": "Visual Studio 17 2022",
            "binaryDir": "${sourceDir}/build/${presetName}",
            "cacheVariables": {
                "CMAKE_TOOLCHAIN_FILE": "$env{VCPKG_INSTALLATION_ROOT}/scripts/buildsystems/vcpkg.cmake",
            }
        }
    ]
}
```

GitHub Actionsでは、CMakeとvcpkgは元々インストールされているので、準備は必要ありません。`github`プリセットを指定してCMakeを実行し、ビルドします。

```yaml:.github/workflows/build.yml
name: Build

on: [push, pull_request]

jobs:
  build:
    runs-on: windows-latest

    steps:
    - name: Checkout repository
      uses: actions/checkout@v4

    - name: Configure CMake
      run: |
        cmake . --preset github

    - name: Build
      run: |
        cmake --build build/github --config Release -j 8
```

# Vulkan SDKのインストール

Vulkanを使いたいので、SDKをインストールします。

Vulkanのセットアップ用のアクションとして [Setup Vulkan SDK](https://github.com/marketplace/actions/setup-vulkan-sdk) や [Install Vulkan SDK](https://github.com/marketplace/actions/install-vulkan-sdk) があります。しかし、対応バージョンが古かったり、GitHub上で警告が出たりするので、サードパーティのアクションを使わずに、自分で [LunarG のサイト](https://vulkan.lunarg.com/sdk/home)からインストーラをダウンロードします。
参考: [actions/runner-images/#18](https://github.com/actions/runner-images/issues/18)

```yaml:.github/workflows/build.yml
    env:
      VULKAN_SDK: C:\VulkanSDK\

    - name: Setup Vulkan
      run: |
          $ver = (Invoke-WebRequest -Uri "https://vulkan.lunarg.com/sdk/latest.json" | ConvertFrom-Json).windows
          echo Version $ver
          $ProgressPreference = 'SilentlyContinue'
          Invoke-WebRequest -Uri "https://sdk.lunarg.com/sdk/download/$ver/windows/VulkanSDK-$ver-Installer.exe" -OutFile VulkanSDK.exe
          echo Downloaded
          .\VulkanSDK.exe --root C:\VulkanSDK  --accept-licenses --default-answer --confirm-command install
```

# vcpkgのキャッシュ

vcpkgインストールで毎回ライブラリビルドが走ってしまうと時間がかかりすぎるので、キャッシュできるようにします。正直あまり理解できていませんが、ちゃんと二回目以降は高速になりました。参考: [チュートリアル: GitHub Actions Cache を使用して vcpkg バイナリ キャッシュを設定する](https://learn.microsoft.com/ja-jp/vcpkg/consume/binary-caching-github-actions-cache)


```yaml:.github/workflows/build.yml
    env: 
      VCPKG_BINARY_SOURCES: "clear;x-gha,readwrite"
    
    - name: Export GitHub Actions cache environment variables
      uses: actions/github-script@v6
      with:
        script: |
          core.exportVariable('ACTIONS_CACHE_URL', process.env.ACTIONS_CACHE_URL || '');
          core.exportVariable('ACTIONS_RUNTIME_TOKEN', process.env.ACTIONS_RUNTIME_TOKEN || '');
```

# 最終的なワークフロー

最終的なワークフローは以下のようになりました。

```yaml:.github/workflows/build.yml
name: Build

on: [push, pull_request]

jobs:
  build:
    runs-on: windows-latest
    env: 
      VCPKG_BINARY_SOURCES: "clear;x-gha,readwrite"
      VULKAN_SDK: C:\VulkanSDK\

    steps:
    - name: Checkout repository
      uses: actions/checkout@v4

    - name: Export GitHub Actions cache environment variables
      uses: actions/github-script@v6
      with:
        script: |
          core.exportVariable('ACTIONS_CACHE_URL', process.env.ACTIONS_CACHE_URL || '');
          core.exportVariable('ACTIONS_RUNTIME_TOKEN', process.env.ACTIONS_RUNTIME_TOKEN || '');

    - name: Install Vulkan SDK
      run: |
          $ver = (Invoke-WebRequest -Uri "https://vulkan.lunarg.com/sdk/latest.json" | ConvertFrom-Json).windows
          echo Version $ver
          $ProgressPreference = 'SilentlyContinue'
          Invoke-WebRequest -Uri "https://sdk.lunarg.com/sdk/download/$ver/windows/VulkanSDK-$ver-Installer.exe" -OutFile VulkanSDK.exe
          echo Downloaded
          .\VulkanSDK.exe --root C:\VulkanSDK  --accept-licenses --default-answer --confirm-command install

    - name: Configure CMake
      run: |
        cmake . --preset github

    - name: Build
      run: |
        cmake --build build/github --config Release -j 8
```

ここまでできればテストなども可能そうですね。ウィンドウの扱いは注意が必要そうですが。
では、この記事は以上です。ありがとうございました。
