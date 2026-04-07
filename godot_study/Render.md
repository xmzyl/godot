# Godot 渲染技术学习大纲（主攻 RD 路线）

基于本地工程路径：`d:\MyDev\godot\godot`  
目标：从 Godot 源码中建立可迁移的“现代渲染器（RenderingDevice, RD）”心智模型，并能定位一帧内关键阶段、资源流与性能瓶颈。

---

## 1. 总体渲染心智模型（必读一次）

**目标**：搞清楚“一帧里渲染系统做了哪些阶段、顺序是什么、谁调度谁”。

- 帧调度入口（关键函数 `_draw`）：[rendering_server_default.cpp:L69-L109](file:///d:/MyDev/godot/godot/servers/rendering/rendering_server_default.cpp#L69-L109)  
  关注调用顺序：
  - `RSG::rasterizer->begin_frame(frame_step)`
  - `RSG::scene->update()`
  - `RSG::canvas->update()`
  - `RSG::particles_storage->update_particles()`
  - `RSG::scene->render_probes()`
  - `RSG::viewport->draw_viewports(p_swap_buffers)`
  - `RSG::canvas_render->update()`
  - `RSG::rasterizer->end_frame(p_swap_buffers)`

- RSG 全局对象装配（选择 rasterizer / scene / storages）：[rendering_server_default.cpp:L239-L272](file:///d:/MyDev/godot/godot/servers/rendering/rendering_server_default.cpp#L239-L272)

---

## 2. Viewport：渲染任务组织与 2D/3D 调度枢纽

**目标**：把 Viewport 当作“渲染任务调度器”，理解它如何组织 3D、2D、清屏、SDF、XR、上屏。

- 3D 入口：`RSG::scene->render_camera(...)`：[renderer_viewport.cpp:L291-L321](file:///d:/MyDev/godot/godot/servers/rendering/renderer_viewport.cpp#L291-L321)
- 每个 Viewport 的完整绘制流程（非常关键）：[renderer_viewport.cpp:L324-L431](file:///d:/MyDev/godot/godot/servers/rendering/renderer_viewport.cpp#L324-L431)
  - 关注：是否创建 `render_buffers`、clear request、3D/2D 顺序、2D SDF 的生成与标记

---

## 3. 渲染器选择与“上屏”路径（RD vs GLES3）

**目标**：明确 Godot 4.x 至少有两条主渲染路径：
- RenderingDevice（RD）：Vulkan / D3D12 / Metal 等现代 API 抽象
- gl_compatibility：GLES3 / OpenGL 3.3 路径

- RD compositor（负责 begin/end frame + blit 到屏幕等）：  
  - [renderer_compositor_rd.h](file:///d:/MyDev/godot/godot/servers/rendering/renderer_rd/renderer_compositor_rd.h)  
  - [renderer_compositor_rd.cpp:L108-L162](file:///d:/MyDev/godot/godot/servers/rendering/renderer_rd/renderer_compositor_rd.cpp#L108-L162)

- RD 中 Forward+ / Mobile 渲染器选择逻辑：  
  [renderer_compositor_rd.cpp:L313-L336](file:///d:/MyDev/godot/godot/servers/rendering/renderer_rd/renderer_compositor_rd.cpp#L313-L336)

- GLES3 总入口（对比用）：[rasterizer_gles3.cpp](file:///d:/MyDev/godot/godot/drivers/gles3/rasterizer_gles3.cpp)

---

## 4. RenderingDevice（RD）：核心抽象与资源/命令模型

**目标**：理解 RD 的 API 语义：
- 资源：buffer/texture/sampler/pipeline 等
- 列表：draw_list / compute_list
- 同步与屏障（由后端/traits 约束）
- shader/pipeline 创建、变体与缓存

- RD 核心接口：  
  [rendering_device.h](file:///d:/MyDev/godot/godot/servers/rendering/rendering_device.h)
- RD 对外文档（接口语义反查）：  
  [RenderingDevice.xml](file:///d:/MyDev/godot/godot/doc/classes/RenderingDevice.xml)

---

## 5. RD 后端驱动层：Vulkan / D3D12 / Metal 如何接入

**目标**：把 RD 分两层理解：
- `RenderingContextDriver`：设备/表面/交换链等“上下文”
- `RenderingDeviceDriver`：资源与命令的“真正后端实现”

- 后端抽象接口：  
  [rendering_device_driver.h](file:///d:/MyDev/godot/godot/servers/rendering/rendering_device_driver.h)

- Vulkan：  
  [rendering_context_driver_vulkan.h](file:///d:/MyDev/godot/godot/drivers/vulkan/rendering_context_driver_vulkan.h)  
  [rendering_device_driver_vulkan.h](file:///d:/MyDev/godot/godot/drivers/vulkan/rendering_device_driver_vulkan.h)

- D3D12：  
  [rendering_device_driver_d3d12.h](file:///d:/MyDev/godot/godot/drivers/d3d12/rendering_device_driver_d3d12.h)  
  [rendering_device_driver_d3d12.cpp](file:///d:/MyDev/godot/godot/drivers/d3d12/rendering_device_driver_d3d12.cpp)

- Metal：  
  [rendering_context_driver_metal.h](file:///d:/MyDev/godot/godot/drivers/metal/rendering_context_driver_metal.h)  
  [rendering_device_driver_metal.h](file:///d:/MyDev/godot/godot/drivers/metal/rendering_device_driver_metal.h)  
  [drivers/metal/README.md](file:///d:/MyDev/godot/godot/drivers/metal/README.md)

---

## 6. 2D 渲染（Canvas）：从 Viewport 收集到真正绘制

**目标**：理解 2D 的“收集(cull) → 光照/阴影 → SDF/遮挡 → 批处理与材质/shader”。

- RD 路径的 2D 渲染实现：  
  [renderer_canvas_render_rd.h](file:///d:/MyDev/godot/godot/servers/rendering/renderer_rd/renderer_canvas_render_rd.h)  
  [renderer_canvas_render_rd.cpp](file:///d:/MyDev/godot/godot/servers/rendering/renderer_rd/renderer_canvas_render_rd.cpp)

- GLES3 路径对比：  
  [rasterizer_canvas_gles3.h](file:///d:/MyDev/godot/godot/drivers/gles3/rasterizer_canvas_gles3.h)

---

## 7. 3D 渲染：Forward Clustered（Forward+）与 Mobile（RD 路线重点）

**目标**：把 3D 渲染拆成模块读：
- RenderSceneBuffers（命名纹理/深度/历史帧/后处理输入输出）
- 渲染列表（实例、材质、pass、排序）
- 灯光与阴影（聚簇/光源列表/阴影 atlas 等）
- 环境与后处理（雾、曝光、GI、AA、upscale 等）

- RD 3D 渲染基类接口（掌握这些虚函数即可顺藤摸瓜）：  
  [renderer_scene_render_rd.h](file:///d:/MyDev/godot/godot/servers/rendering/renderer_rd/renderer_scene_render_rd.h)

- RenderSceneBuffersRD（命名纹理、缩放/TAA/MSAA/去色带/VRS 等）：  
  [render_scene_buffers_rd.h](file:///d:/MyDev/godot/godot/servers/rendering/renderer_rd/storage_rd/render_scene_buffers_rd.h)

- Forward+ shader（读材质宏/光照聚簇落地实现时使用）：  
  [scene_forward_clustered.glsl](file:///d:/MyDev/godot/godot/servers/rendering/renderer_rd/shaders/forward_clustered/scene_forward_clustered.glsl)

---

## 8. 效果模块（按需深入）

**目标**：每个模块按“输入 → 输出 → 串联位置”方式理解。

- GI（SDFGI / VoxelGI 等 RD 实现入口与相关 shader include）：  
  [gi.h](file:///d:/MyDev/godot/godot/servers/rendering/renderer_rd/environment/gi.h)

- resolve 等通用效果（compute/raster 两套实现选择）：  
  [resolve.h](file:///d:/MyDev/godot/godot/servers/rendering/renderer_rd/effects/resolve.h)

---

## 9. Shader 系统：从 Godot Shader 到 GPU 可执行

**目标**：搞清楚：
- 变体与 feature flags / specialization constants
- 编译与缓存（磁盘缓存、pipeline cache id、版本化）
- RD 与 GLES3 两条路径的 shader 生成方式差异（`*.glsl.gen.h` 等）

- GLES3 shader cache 线索（用于对比 shader 缓存与初始化）：  
  [rasterizer_gles3.cpp:L321-L360](file:///d:/MyDev/godot/godot/drivers/gles3/rasterizer_gles3.cpp#L321-L360)

---

## 10. 性能与调试：时间戳与 profile zone

**目标**：用引擎自带标记体系定位瓶颈，而不是靠猜。

- timestamp 采集与汇总、viewport `vp_` 时间片处理：  
  [rendering_server_default.cpp:L129-L206](file:///d:/MyDev/godot/godot/servers/rendering/rendering_server_default.cpp#L129-L206)

---

# 主攻 RD：推荐阅读路线（不细化版）

## A. 先跑通“最小 RD 3D 闭环”（一口气读通一次）

- 一帧调度：[_draw](file:///d:/MyDev/godot/godot/servers/rendering/rendering_server_default.cpp#L69-L109)
- Viewport 触发 3D：[_draw_3d → render_camera](file:///d:/MyDev/godot/godot/servers/rendering/renderer_viewport.cpp#L291-L321)
- SceneCull 把剔除结果交给 scene_render：[`scene_render->render_scene`](file:///d:/MyDev/godot/godot/servers/rendering/renderer_scene_cull.cpp#L3477-L3491)
- RD 场景渲染器组装 `RenderDataRD` 并分派到 `_render_scene`：  
  [renderer_scene_render_rd.cpp:L1327-L1473](file:///d:/MyDev/godot/godot/servers/rendering/renderer_rd/renderer_scene_render_rd.cpp#L1327-L1473)
- ForwardClustered 的 `_render_scene` 组织 passes：  
  [render_forward_clustered.cpp:L1674-L1859](file:///d:/MyDev/godot/godot/servers/rendering/renderer_rd/forward_clustered/render_forward_clustered.cpp#L1674-L1859)
- RD draw 提交的最小形态：  
  [draw_list_begin/end 最小闭环](file:///d:/MyDev/godot/godot/servers/rendering/renderer_rd/forward_clustered/render_forward_clustered.cpp#L680-L687)
- 最终 present：  
  [swap_buffers](file:///d:/MyDev/godot/godot/servers/rendering/renderer_rd/renderer_compositor_rd.cpp#L120-L122)

---

## B. 再把 RD 的“职责分层”理清（决定你读代码是否迷路）

- 剔除/组织世界数据：`RendererSceneCull`  
  关注阴影/SDFGI 更新、以及各类列表如何生成并传递：  
  [renderer_scene_cull.cpp:L3411-L3499](file:///d:/MyDev/godot/godot/servers/rendering/renderer_scene_cull.cpp#L3411-L3499)

- 把列表画出来：`RendererSceneRenderRD`  
  重点看 `RenderSceneDataRD` 与 `RenderDataRD` 如何被填充：  
  [renderer_scene_render_rd.cpp:L1336-L1473](file:///d:/MyDev/godot/godot/servers/rendering/renderer_rd/renderer_scene_render_rd.cpp#L1336-L1473)  
  `RenderDataRD` 结构字段速览：  
  [render_data_rd.h](file:///d:/MyDev/godot/godot/servers/rendering/renderer_rd/storage_rd/render_data_rd.h)

- 选择具体 3D 渲染实现（Forward+ / Mobile）：`RendererCompositorRD`  
  [renderer_compositor_rd.cpp:L313-L336](file:///d:/MyDev/godot/godot/servers/rendering/renderer_rd/renderer_compositor_rd.cpp#L313-L336)

---

## C. RenderSceneBuffersRD：理解“命名纹理体系”（RD 路线地基）

- Viewport 配置 3D render buffers：  
  [renderer_viewport.cpp:L272-L287](file:///d:/MyDev/godot/godot/servers/rendering/renderer_viewport.cpp#L272-L287)
- RD render buffers 创建 color/depth/MSAA/VRS + 配置模块自定义数据：  
  [render_scene_buffers_rd.cpp:L151-L209](file:///d:/MyDev/godot/godot/servers/rendering/renderer_rd/storage_rd/render_scene_buffers_rd.cpp#L151-L209)
- ForwardClustered “按需确保纹理存在”（specular/normal_roughness/voxelgi/FSR2…）：  
  [render_forward_clustered.cpp:L49-L108](file:///d:/MyDev/godot/godot/servers/rendering/renderer_rd/forward_clustered/render_forward_clustered.cpp#L49-L108)

---

## D. ForwardClustered：按“阶段”读（把 `_render_scene` 当目录）

- 场景准备：motion vectors / TAA / upscaling / SSR / SDFGI / multiview 的条件：  
  [render_forward_clustered.cpp:L1694-L1860](file:///d:/MyDev/godot/godot/servers/rendering/renderer_rd/forward_clustered/render_forward_clustered.cpp#L1694-L1860)

- 环境与 UBO 更新 + cluster 参数写入：  
  [render_forward_clustered.cpp:L689-L760](file:///d:/MyDev/godot/godot/servers/rendering/renderer_rd/forward_clustered/render_forward_clustered.cpp#L689-L760)

- draw list 的提交语义落地：  
  [render_forward_clustered.cpp:L680-L687](file:///d:/MyDev/godot/godot/servers/rendering/renderer_rd/forward_clustered/render_forward_clustered.cpp#L680-L687)

---

## E. RD API 与后端驱动：建议在能画出“一帧 pass 图”后再深入

- RD API：  
  [rendering_device.h](file:///d:/MyDev/godot/godot/servers/rendering/rendering_device.h)
- 后端抽象：  
  [rendering_device_driver.h](file:///d:/MyDev/godot/godot/servers/rendering/rendering_device_driver.h)
- Windows 重点：  
  [rendering_device_driver_d3d12.h](file:///d:/MyDev/godot/godot/drivers/d3d12/rendering_device_driver_d3d12.h)
- descriptor/framebuffer 缓存（理解为什么不会“每帧疯狂 new”）：  
  [uniform_set_cache_rd.h](file:///d:/MyDev/godot/godot/servers/rendering/renderer_rd/uniform_set_cache_rd.h)  
  [framebuffer_cache_rd.h](file:///d:/MyDev/godot/godot/servers/rendering/renderer_rd/framebuffer_cache_rd.h)

---

# 推荐练习（边读边做）

- 练习 1：画 RD 3D 的调用链图  
  `_draw` → `draw_viewports` → `render_camera` → `scene_render->render_scene` → `RenderForwardClustered::_render_scene` → `draw_list_begin/end` → `swap_buffers`

- 练习 2：列出 ForwardClustered 一帧会用到的命名纹理（RB_SCOPE/RB_TEX）并标注“谁写/谁读”  
  入口从：  
  - [render_scene_buffers_rd.cpp:L151-L209](file:///d:/MyDev/godot/godot/servers/rendering/renderer_rd/storage_rd/render_scene_buffers_rd.cpp#L151-L209)  
  - [render_forward_clustered.cpp:L49-L108](file:///d:/MyDev/godot/godot/servers/rendering/renderer_rd/forward_clustered/render_forward_clustered.cpp#L49-L108)

- 练习 3：选一个特性做纵向切片（TAA / SSR / SDFGI / 体积雾等）  
  只回答三个问题：输入是什么？输出是什么？在一帧的哪里串起来？

---