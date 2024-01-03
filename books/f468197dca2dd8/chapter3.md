---
title: "プロジェクト設定"
---

# ベースコード
さっそくコードを書いていきます。まずはベースとなるコードを記述します。

```cpp
#include "vkutils.hpp"

class Application
{
public:
    void run() {
        initWindow();
        initVulkan();
    }

private:
    void initWindow() {}

    void initVulkan() {}
};

int main() {
    Application app;
    app.run();
}
```

エラーなく実行できることを確認してください。

# GLFWでウィンドウを出す

GLFWでウィンドウを出していきます。まずはウィンドウサイズをクラスの上で定義します。
```cpp
constexpr uint32_t WIDTH = 800;
constexpr uint32_t HEIGHT = 600;
```

次にウィンドウのハンドルをクラス変数に追加します。
```cpp
private:
    GLFWwindow* window;
```

`initWindow()`の中でウィンドウを作成しましょう。
```cpp
void initWindow() {
+   glfwInit();
+   glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);
+   glfwWindowHint(GLFW_RESIZABLE, GLFW_FALSE);
+   window = glfwCreateWindow(WIDTH, HEIGHT, "Vulkan", nullptr, nullptr);
}
```

 `run()` にメインループを追加し、終了したらウィンドウを削除します。
```diff cpp
void run() {
    initWindow();
    initVulkan();

+   while (!glfwWindowShouldClose(window)) {
+       glfwPollEvents();
+   }

+   glfwDestroyWindow(window);
+   glfwTerminate();
}
```

これで実行するとウィンドウが出るようになったはずです。
![](https://storage.googleapis.com/zenn-user-upload/jvumbprttxc40c7ysrstf8ejrwi9)

ベースとなるコードができました。次の章では Vulkan のセットアップを行っていきます。

[ここまでのC++コード(00_base_code.cpp)](https://github.com/nishidate-yuki/vulkan_raytracing_from_scratch/blob/master/code/00_base_code.cpp)

