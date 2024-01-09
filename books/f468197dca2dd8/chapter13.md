---
title: "三角形の描画"
---

TODO: 説明追加

```cpp
void updateDescriptorSet(vk::ImageView imageView) {
    std::vector<vk::WriteDescriptorSet> writes(2);

    // [0]: For AS
    vk::WriteDescriptorSetAccelerationStructureKHR accelInfo{};
    accelInfo.setAccelerationStructures(*topAccel.accel);
    writes[0].setDstSet(*descSet);
    writes[0].setDstBinding(0);
    writes[0].setDescriptorCount(1);
    writes[0].setDescriptorType(
        vk::DescriptorType::eAccelerationStructureKHR);
    writes[0].setPNext(&accelInfo);

    // [0]: For storage image
    vk::DescriptorImageInfo imageInfo{};
    imageInfo.setImageView(imageView);
    imageInfo.setImageLayout(vk::ImageLayout::eGeneral);
    writes[1].setDstSet(*descSet);
    writes[1].setDstBinding(1);
    writes[1].setDescriptorType(vk::DescriptorType::eStorageImage);
    writes[1].setImageInfo(imageInfo);

    // Update
    device->updateDescriptorSets(writes, nullptr);
}

void recordCommandBuffer(vk::Image image) {
    // Begin
    commandBuffer->begin(vk::CommandBufferBeginInfo{});

    // Set image layout to general
    vkutils::setImageLayout(*commandBuffer, image,  //
                            vk::ImageLayout::ePresentSrcKHR,
                            vk::ImageLayout::eGeneral);

    // Bind pipeline
    commandBuffer->bindPipeline(vk::PipelineBindPoint::eRayTracingKHR,
                                *pipeline);

    // Bind desc set
    commandBuffer->bindDescriptorSets(
        vk::PipelineBindPoint::eRayTracingKHR,  // pipelineBindPoint
        *pipelineLayout,                        // layout
        0,                                      // firstSet
        *descSet,                               // descSets
        nullptr                                 // dynamicOffsets
    );

    // Trace rays
    commandBuffer->traceRaysKHR(  //
        raygenRegion,             // raygen
        missRegion,               // miss
        hitRegion,                // hit
        {},                       // callable
        WIDTH, HEIGHT, 1          // width, height, depth
    );

    // Set image layout to present src
    vkutils::setImageLayout(*commandBuffer, image,  //
                            vk::ImageLayout::eGeneral,
                            vk::ImageLayout::ePresentSrcKHR);

    // End
    commandBuffer->end();
}

void drawFrame() {
    static int frame = 0;
    std::cout << frame << '\n';

    // Create semaphore
    vk::UniqueSemaphore imageAvailableSemaphore =
        device->createSemaphoreUnique({});

    // Acquire next image
    auto result = device->acquireNextImageKHR(
        *swapchain, std::numeric_limits<uint64_t>::max(),
        *imageAvailableSemaphore);
    if (result.result != vk::Result::eSuccess) {
        std::cerr << "Failed to acquire next image.\n";
        std::abort();
    }

    // Update descriptor sets using current image
    uint32_t imageIndex = result.value;
    updateDescriptorSet(*swapchainImageViews[imageIndex]);

    // Record command buffer
    recordCommandBuffer(swapchainImages[imageIndex]);

    // Submit command buffer
    vk::PipelineStageFlags waitStage{vk::PipelineStageFlagBits::eTopOfPipe};
    vk::SubmitInfo submitInfo{};
    submitInfo.setWaitDstStageMask(waitStage);
    submitInfo.setCommandBuffers(*commandBuffer);
    submitInfo.setWaitSemaphores(*imageAvailableSemaphore);
    queue.submit(submitInfo);

    // Wait
    queue.waitIdle();

    // Present
    vk::PresentInfoKHR presentInfo{};
    presentInfo.setSwapchains(*swapchain);
    presentInfo.setImageIndices(imageIndex);
    if (queue.presentKHR(presentInfo) != vk::Result::eSuccess) {
        std::cerr << "Failed to present.\n";
        std::abort();
    }

    frame++;
}
```


[最終的なC++コード(10_draw_triangle.hpp)](https://github.com/nishidate-yuki/vulkan_raytracing_from_scratch/blob/master/code/10_draw_triangle.hpp)

[シェーダファイル](https://github.com/nishidate-yuki/vulkan_raytracing_from_scratch/tree/master/shaders)
