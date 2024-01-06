---
title: "ディスクリプタの作成"
---

この章ではディスクリプタを作成します。ディスクリプタ自体はレイトレ特有のものではないので、サクッと作ってしまいます。

# ディスクリプタプールを作成

まずは必要なメンバ変数を追加します。

```cpp
class Application{
    // ...
    vk::UniqueDescriptorPool descPool;
    vk::UniqueDescriptorSetLayout descSetLayout;
    vk::UniqueDescriptorSet descSet;
};
```

`createDescriptorPool()`を作成して呼び出します。シェーダで `binding=0` にTLASを、`binding=1` にストレージイメージを割り当てたため、それらを`setPoolSizes()`に渡してディスクリプタプールを作成します。`vk::DescriptorPoolCreateFlagBits::eFreeDescriptorSet`を指定すると、このプールから作成したディスクリプタセットはプールが破棄される際に自動で解放されるようになります。

```cpp
void initVulkan() {
    //...

    createDescriptorPool();
}

void createDescriptorPool(){
    std::vector<vk::DescriptorPoolSize> poolSizes = {
        {vk::DescriptorType::eAccelerationStructureKHR, 1},
        {vk::DescriptorType::eStorageImage, 1},
    };

    vk::DescriptorPoolCreateInfo createInfo{};
    createInfo.setPoolSizes(poolSizes);
    createInfo.setMaxSets(1);
    createInfo.setFlags(vk::DescriptorPoolCreateFlagBits::eFreeDescriptorSet);
    descPool = device->createDescriptorPoolUnique(createInfo);
}
```

# ディスクリプタセットレイアウトを作成

次にディスクリプタセットレイアウトを作成するため、`createDescSetLayout()`を作成して呼び出します。先ほどと同様に、`binding=0`にはTLAS、`binding=1`にはストレージイメージの情報を渡します。

```cpp
void initVulkan() {
    //...

    createDescSetLayout();
}

void createDescSetLayout() {
    std::vector<vk::DescriptorSetLayoutBinding> bindings(2);

    // [0]: For AS
    bindings[0].setBinding(0);
    bindings[0].setDescriptorType(
        vk::DescriptorType::eAccelerationStructureKHR);
    bindings[0].setDescriptorCount(1);
    bindings[0].setStageFlags(vk::ShaderStageFlagBits::eRaygenKHR);

    // [1]: For storage image
    bindings[1].setBinding(1);
    bindings[1].setDescriptorType(vk::DescriptorType::eStorageImage);
    bindings[1].setDescriptorCount(1);
    bindings[1].setStageFlags(vk::ShaderStageFlagBits::eRaygenKHR);

    vk::DescriptorSetLayoutCreateInfo createInfo{};
    createInfo.setBindings(bindings);
    descSetLayout = device->createDescriptorSetLayoutUnique(createInfo);
}
```

# ディスクリプタセットの作成

最後にディスクリプタセットを作成します。

```cpp
void initVulkan() {
    //...

    createDescriptorSet();
}

void createDescriptorSet() {
    std::cout << "Create desc set\n";

    vk::DescriptorSetAllocateInfo allocateInfo{};
    allocateInfo.setDescriptorPool(*descPool);
    allocateInfo.setSetLayouts(*descSetLayout);
    descSet = std::move(
        device->allocateDescriptorSetsUnique(allocateInfo).front());
}
```

[ここまでのC++コード(07_create_descriptor.hpp)](https://github.com/nishidate-yuki/vulkan_raytracing_from_scratch/blob/master/code/07_create_descriptor.hpp)
