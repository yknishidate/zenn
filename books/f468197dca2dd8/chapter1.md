
# Vulkan Ray Tracingについて

Vulkan Ray Tracingは、Vulkanに追加された拡張機能群の総称です。NVIDIAやAMDなど、複数のベンダーのハードウェアレイトレーシング機能に統一的にアクセスできるAPIです。

2020年11月に[Vulkan Ray Tracing の最終仕様](https://www.khronos.org/blog/vulkan-ray-tracing-final-specification-release)がリリースされ、12月には最終仕様に対応したVulkan SDK 1.2.162.0 が公開されました。

# この記事について

この記事では、Vulkan Ray Tracingでゼロから三角形をレンダリングするまでの手順をステップごとに記述します。

![](https://storage.googleapis.com/zenn-user-upload/rr5crszad0xyh2a33lxmbh21gd0u)
*記事の最終的な結果画像。レイトレで三角形を出す*

## 更新履歴

2021年1月: 記事公開 (Version 1)

2024年1月: 大幅刷新 (Version 2)
- CMake + vcpkgによるクロスプラットフォームなプロジェクトに変更
- Storage imageを削除し、Swapchain imagesに直接描画する形式に変更
- コードの読みやすさを大幅に改善
- コード量を大幅に削減（1500行→1100行）

## 記事の特徴

- クロスベンダー仕様(`KHR`)に対応している
- サードパーティの Vulkan ライブラリを使用せずゼロから記述する
- 公式 C++ラッパー `vulkan.hpp` を使うため冗長な記述が少ない
- 小さなヘルパーヘッダを 1 つだけ提供する

### ヘルパーについて
レイトレーシングに関係のないVulkanのセットアップ部分についてはヘルパーでスキップします。このヘルパーは単純な関数をまとめた程度のもので、大きなクラスは存在しません。そのため設計としては良くありませんが、簡単に読むことができるようになっています。また、このヘルパーの大部分は [Vulkan Tutorial](https://vulkan-tutorial.com/) に準拠しているため、こちらを参照することで中身を理解できるはずです。

# 対象読者
[Vulkan Tutorial](https://vulkan-tutorial.com/) など、Vulkanの一般的な入門記事を一周していることを推奨します。

# リポジトリ
この記事で使用するプログラムは以下のリポジトリに置いています。
https://github.com/nishidate-yuki/vulkan_raytracing_from_scratch
