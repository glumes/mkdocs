---
title: "Vulkan 在 FFmpeg 中的支持"
date: 2022-01-18T00:16:16+08:00
slug: "ffmpeg-vulkan-support"
subtitle: ""
categories: ["FFmpeg"]
tags: ["FFmpeg"]
toc: true
 
draft: false
original: true
image: ""
---

周末时候看到一篇推送说 FFmpeg 升级到 5.0 版本了。

其中提到 FFmpeg 引入了 Vulkan 驱动的新滤镜，用于视频水平、垂直翻转。

看到 FFmpeg 引入了 Vulkan ，想着这是要有什么大动作啊，直接利好 Vulkan 嘛？

后来又仔细看了下 FFmpeg 的 Changelog ，原来早在 4.3 版本就已经开始支持 Vulkan 了。

<!--more-->

![](https://image.glumes.com/blog_image20220326201219.png)

![](https://image.glumes.com/blog_image20220326201238.png)

那时候就已经有滤镜支持了，比如 scale_vulkan、chromaber_vulkan 等。

而且还支持在 Linux 平台上通过 Vulkan 使用 AMD 的高级媒体框架（AMF）库，可以用 GPU 来进行 H.264/HEVC 的编码。（Windows 平台用的是 DirectX 接口）

这里提一下 AMF 框架，实际上我也是第一次接触这个。

AMF 全称是 Advanced Media Framework ，翻译为高级媒体框架。它是 AMD 公司出品的，为开发人员提供对 GPU 的访问以进行多媒体处理，通过 AMF 可以进行视频编解码、转码、色彩空间转换等功能。

简单说就是提供了对自家显卡产品能力的调用，可以用它来做编解码的工作。既然 AMD 有了，那么相信 NVIDIA 也有类似的产品。

由此可见后面的趋势：渲染 API 不仅仅是用来做渲染，还是可以用做编解码的，毕竟它是可以直接用 GPU 打交道的。

所以 FFmpeg 5.0 中引入了 Vulkan 新滤镜应该也不是什么大新闻了，毕竟在 4.3 版本就已经有了支持，只是多了几个滤镜，按照开发人员的话来说，就是多了几个 shader 嘛。

---

接下来就看看这几个新增的 翻转shader 有何不同之处：

如果不了解 Vulkan 流程的话，建议看看 Vulkan 相关的文章，毕竟这里面概念挺多的，但很多流程还是固定的，只要抓到重点就好了。

大概的流程：Vulkan 作为 FFmpeg 中的一个滤镜，那么它肯定要接收代表解码后的 AVFrame 数据，通过将 AVFrame 数据转换为它渲染链结构的输入，经过渲染后，将渲染结果转换为 AVFrame 数据并往下进行传递。

理解上面的流程，剩下的就是去理解 Vulkan 的渲染链了。

核心代码如下：

```cpp
static int process_frames(AVFilterContext *avctx, AVFrame *outframe, AVFrame *inframe)
{
    // 省略起始代码
    // 得到输入数据
    AVVkFrame *in = (AVVkFrame *)inframe->data[0];
    AVVkFrame *out = (AVVkFrame *)outframe->data[0];
    const int planes = av_pix_fmt_count_planes(s->vkctx.output_format);
    const VkFormat *input_formats = av_vkfmt_from_pixfmt(s->vkctx.input_format);
    const VkFormat *output_formats = av_vkfmt_from_pixfmt(s->vkctx.output_format);

    ff_vk_start_exec_recording(vkctx, s->exec);
    cmd_buf = ff_vk_get_exec_buf(s->exec);

    for (int i = 0; i < planes; i++) {
        // 将输入数据绑定到 ImageView 上
        RET(ff_vk_create_imageview(vkctx, s->exec,
                                   &s->input_images[i].imageView, in->img[i],
                                   input_formats[i],
                                   ff_comp_identity_map));

        RET(ff_vk_create_imageview(vkctx, s->exec,
                                   &s->output_images[i].imageView, out->img[i],
                                   output_formats[i],
                                   ff_comp_identity_map));

        s->input_images[i].imageLayout  = VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL;
        s->output_images[i].imageLayout = VK_IMAGE_LAYOUT_GENERAL;
    }
    // 绑定资源描述符
    ff_vk_update_descriptor_set(vkctx, s->pl, 0);
    // 设置好内存屏障
    for (int i = 0; i < planes; i++) {
        // 省略一大串代码
        vk->CmdPipelineBarrier(cmd_buf, VK_PIPELINE_STAGE_TOP_OF_PIPE_BIT,
                               VK_PIPELINE_STAGE_COMPUTE_SHADER_BIT, 0,
                               0, NULL, 0, NULL, FF_ARRAY_ELEMS(barriers), barriers);
        // 省略一大串代码
    }
    // 设置好 pipeline 和 资源描述符集 descriptorSet
    ff_vk_bind_pipeline_exec(vkctx, s->exec, s->pl);
    vk->CmdDispatch(cmd_buf, FFALIGN(s->vkctx.output_width, CGS)/CGS,
                    s->vkctx.output_height, 1);

    ff_vk_add_exec_dep(vkctx, s->exec, inframe, VK_PIPELINE_STAGE_TOP_OF_PIPE_BIT);
    ff_vk_add_exec_dep(vkctx, s->exec, outframe, VK_PIPELINE_STAGE_TOP_OF_PIPE_BIT);
    // 提交到队列
    err = ff_vk_submit_exec_queue(vkctx, s->exec);
    if (err)
        return err;
    ff_vk_qf_rotate(&s->qf);
    return 0;
}
```

以上代码要是看的费劲的话，还是只看核心的 shader 部分吧：


![](https://image.glumes.com/blog_image20220326201302.png)                                                 

可以看出，做水平或者垂直翻转也只是更改了 texture 采样坐标而已，如果你会 OpenGL 的话，一样可以做出类似的 filter 。



