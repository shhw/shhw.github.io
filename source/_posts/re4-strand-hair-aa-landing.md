---
title: Strand Hair Anti-Aliasing Landing Document
date: 2026-07-15 16:36:24
tags:
  - Rendering
  - Anti-Aliasing
  - Strand Hair
categories: 图形渲染
---

# Strand Hair Anti-Aliasing Landing Document

本文整理自 CAPCOM《Resident Evil 4 Hair Discussion》的 Strand Rendering 部分，并把其中的头发抗锯齿方案转换成可落地的实时渲染设计。文档不绑定特定引擎现状，目标是给任意现代渲染管线提供一套可实现、可调试、可验收的 strand hair AA 方案。

原始资料：

- CAPCOM, Resident Evil 4 Hair Discussion: https://www.docswell.com/s/CAPCOM_RandD/KVVYN3-RE2023
- 关键页：P3-P31 为 Strand Rendering，P60 补充自动 LOD。

## 1. 目标

Strand hair 的抗锯齿难点不只是边缘锯齿，而是大量亚像素发丝在时间、透明度、深度排序和后处理之间互相影响：

- 单纯用 TAA：粗发丝能稳定，但细发丝容易断裂、闪烁、拖影和糊成一片。
- 单纯用 alpha/dither：在 deferred 或低分辨率下会出现噪声、空洞和密度不稳定。
- 单纯画透明：发丝互相穿插，普通 alpha blend 依赖绘制顺序，结果不稳定。

落地目标：

- 粗发丝走硬件 raster + TAA，控制成本。
- 细发丝走 compute software rasterizer，在屏幕空间显式计算 coverage/transmittance，得到空间 AA。
- 半透明叠加使用 OIT/MLAB，避免发丝顺序导致的明显错误。
- 最终合成时给 TAA responsive signal，避免已经 AA 过的细发丝被再次时间累积抹糊。
- LOD 时稳定减少发丝数量，并用宽度或不透明度补偿视觉密度。

## 2. 总体管线

CAPCOM PPT P6 给出的顺序是：

```text
Hair classification and culling
-> Lighting
-> Hardware rendering
-> Software semi-transparent drawing
-> Composite to final buffer
```

建议落地 render graph：

```text
1. BuildHairSegments
   输入 Alembic/strand buffer，生成 segment buffer、root/strand id、宽度、材质参数。

2. ClassifyAndCull
   按屏幕投影面积、深度、视锥、LOD rank 分类：
   - Hardware list: 屏幕上足够粗的 segment。
   - Software list: 亚像素或需要连续透明覆盖的 segment。

3. HairLighting
   对 strand vertex 或 segment endpoint 预计算光照，输出 vertex color/lighting buffer。
   硬件路径和软件路径共用该结果。

4. HardwareHairPass
   将大 segment 展开成 camera-facing billboard/ribbon。
   输出 color、depth、motion vector。
   AA 主要依赖 TAA。

5. SoftwareSetup
   低分辨率 depth、froxel/tile binning、线段索引分配。

6. SoftwareHairRaster
   compute shader 按 tile rasterize anti-aliased 2D line。
   输出每像素 MLAB/OIT layers，记录 color、depth、transmittance。

7. HairComposite
   将 MLAB layers front-to-back resolve 到主 color。
   根据 transmittance 输出 responsive AA mask、可选 depth、可选 motion vector。

8. Temporal/PostProcess
   Hardware hair 正常参与 TAA。
   Software hair 区域降低历史权重或启用 responsive AA。
```

## 3. 核心思想：按屏幕投影尺寸分流

P7-P10 的关键结论是：同一套发丝，在不同分辨率下适合不同绘制路径。低分辨率时更多发丝落入亚像素范围，应由 software rasterizer 处理；4K/8K 时发丝屏幕宽度变大，hardware rasterizer 的比例自然上升。

### 3.1 分类依据

对每个 hair segment 计算屏幕空间近似面积：

```text
p0 = project(vertex0.position)
p1 = project(vertex1.position)
w0 = projectWidth(vertex0.width, vertex0.depth)
w1 = projectWidth(vertex1.width, vertex1.depth)
len = length(p1.xy - p0.xy)
avgW = 0.5 * (w0 + w1)
projectedArea = len * avgW + capArea(avgW)
```

也可以用更保守的指标：

```text
minWidthPx = min(w0, w1)
maxWidthPx = max(w0, w1)
thinScore = projectedArea 或 maxWidthPx
```

建议初始分类：

```text
if culled:
    discard
else if lodRank >= lodRatio:
    discard
else if projectedArea >= HardwareAreaThreshold && minWidthPx >= HardwareWidthThreshold:
    hardware
else:
    software
```

推荐初始参数：

```text
HardwareWidthThresholdPx = 1.25
HardwareAreaThresholdPx2 = 2.0
SoftwareMaxSegmentLengthPx = 64.0     // 过长 segment 可预切分
```

这些阈值必须由 debug view 校准，不能只靠公式决定。

### 3.2 为什么不能全走硬件

P12-P15 的对比说明：硬件画 camera-facing polygon 后，AA 依赖 TAA。粗发丝可以接受，但低分辨率下细线会变得断续、有空洞。TAA 对这些亚像素信息的恢复不稳定，尤其在头发运动、相机运动、透明叠加和景深后处理存在时更明显。

### 3.3 为什么不能全走软件

P12 也说明，大面积带宽度线段用 software rasterizer 成本高。粗发丝已经覆盖多个像素，硬件 rasterizer 更适合处理它们；software rasterizer 应集中服务最需要连续 coverage 的细发丝。

## 4. 硬件路径

硬件路径负责较粗或较大的 segment。

### 4.1 几何展开

每个 segment 展开为朝向相机的 ribbon/billboard：

```text
dir = normalize(p1_world - p0_world)
view = normalize(cameraPos - segmentMid)
side = normalize(cross(view, dir))

v0_left  = p0_world - side * width0 * 0.5
v0_right = p0_world + side * width0 * 0.5
v1_left  = p1_world - side * width1 * 0.5
v1_right = p1_world + side * width1 * 0.5
```

注意事项：

- 需要处理 `cross(view, dir)` 接近 0 的情况，可回退到屏幕空间法线或上一帧稳定方向。
- 宽度应支持 root/tip 变化。
- 需要正确输出每个顶点/像素 motion vector，否则 TAA 会把粗发丝拖糊或闪烁。

### 4.2 AA 策略

硬件路径的 AA 依赖：

- MSAA：可选，成本较高，对大量透明发丝未必划算。
- TAA：主要稳定手段，需要 motion vector。
- Alpha coverage：可用 smooth coverage 衰减边缘，但不要替代 software path。

硬件路径后续改进方向可以参考 P30 提到的 conservative rasterizer + GBAA + OIT，但第一阶段不建议把它作为必需项。

## 5. 软件路径准备

P16-P18 描述了 software semi-transparent drawing 的准备流程。目标是避免每个 tile 扫描全部发丝，而是先把可能影响该 tile/froxel 的 segment 建立索引。

### 5.1 低分辨率深度

建立一个 reduced depth buffer，用于快速遮挡剔除。P16 中使用横纵各 1/16 的深度。

```text
ReducedDepthSize = FullResolution / 16
ReducedDepth[x,y] = conservative nearest depth in the block
```

用途：

- 粗略剔除完全在实体表面后方的 hair segment。
- 减少 froxel binning 和 software raster 的无效 segment。

如果使用 reverse depth，要保证 min/max 方向一致。

### 5.2 Froxel/Tile Binning

建议把屏幕切成 16x16 pixel tile。每个 tile 也可以按深度切为若干 froxel slice：

```text
TileSize = 16x16
Froxel = TileXY + DepthSlice
```

四个 dispatch：

1. `BuildReducedDepth`
   生成 1/16 depth。

2. `CountSegmentsPerFroxel`
   对 software segment 做屏幕 bbox，落入对应 froxel 后 `InterlockedAdd` 计数。

3. `AllocateFroxelSegmentList`
   对计数做 prefix sum 或 global atomic allocation，得到每个 froxel 在线性 buffer 中的 offset/count，并清零写入计数。

4. `FillFroxelSegmentList`
   再次遍历 software segment，将 segment index 写入对应 froxel 的线性列表。

这种流程有重复 raster/bbox 计算，但优点是稳定、跨硬件可实现、内存大小可控。

## 6. 软件 Rasterizer：2D Anti-Aliased Line

P18 是 AA 的核心：software semi-transparent drawing 在单个 compute shader 内完成，16x16 pixel 一个 group，一个 thread 负责一个 line segment，根据顶点宽度计算 projected area/translucency，并用 2D anti-aliased line 绘制。

### 6.1 Thread/Tile 模型

```text
numthreads 可以为 16x16 或按实现调整。
每个 group 负责一个 tile。
group 读取该 tile/froxel 的 segment index list。
segment 从近到远处理。
当 tile 内 transmittance 已足够低，提前结束。
```

推荐：

```text
TileSize = 16
OpaqueEarlyOutTransmittance = 0.02
MaxSegmentsPerTileSoftLimit = 1024
```

### 6.2 Coverage 计算

对屏幕空间 line segment：

```text
a = p0_screen.xy
b = p1_screen.xy
p = pixelCenter.xy
t = saturate(dot(p - a, b - a) / dot(b - a, b - a))
closest = lerp(a, b, t)
widthPx = lerp(w0_px, w1_px, t)
radius = 0.5 * widthPx
d = length(p - closest)
```

一个简单可用的 AA coverage：

```text
coverage = saturate(radius + 0.5 - d)
```

更平滑的版本：

```text
coverage = smoothstep(radius + 0.5, radius - 0.5, d)
```

如果要更接近 Xiaolin Wu line，可以对主轴方向做分布式权重，但对变宽发丝而言，基于距离场的 capsule coverage 更容易扩展。

### 6.3 Alpha/Transmittance

不要把发丝当成二值覆盖。建议用 transmittance 表达“光穿过该发丝后的剩余比例”：

```text
materialAlpha = strand opacity from asset/material
alpha = coverage * materialAlpha
T = 1 - alpha
```

如果使用物理一点的 optical depth：

```text
tau = density * coverage * widthScale
T = exp(-tau)
alpha = 1 - T
```

软件 rasterizer 输出 fragment：

```text
Fragment {
    depth
    transmittance T
    premultipliedColor = lightingColor * (1 - T)
    optional: strandId/materialId/debugFlags
}
```

### 6.4 深度测试

segment 的深度可按屏幕空间 `t` 插值：

```text
z = interpolateDepth(z0, z1, t)
```

每个 pixel 写入 MLAB 前先做 depth test：

```text
if z is behind opaque scene depth:
    reject
```

对 thick/capsule 的深度可以近似使用中心线深度。高质量版本可考虑 ribbon plane 深度或 per-pixel closest point 深度。

### 6.5 光照插值

P11 建议先算 lighting，结果存为 vertex color，使硬件和软件路径都能复用：

```text
litColor = lerp(vertex0.lighting, vertex1.lighting, t)
fragmentColor = litColor * materialColor * (1 - T)
```

这样 software rasterizer 不需要在 tile 内做复杂光照，只负责覆盖、透明和合成。

## 7. MLAB/OIT 半透明合成

P19-P26 使用 Multi-Layer Alpha Blending, MLAB。它是一种固定内存的 OIT：每个像素保留 N 个 layer，按深度保存前面的 fragment，超出层数的内容合并到最后一层。

### 7.1 Layer 数量

P19-P20 提到 RE4 使用 8 层，因此前 7 层深度和颜色基本有序，最后 1 层负责合并溢出。

建议：

```text
OIT_LAYER_COUNT = 8
LowQuality = 4
HighQuality = 8 或 12
```

### 7.2 Fragment Packing

P21 使用 64-bit packing：

```text
16-bit depth
16-bit transmittance
32-bit color: R11G11B10Float
```

落地建议：

```text
uint64 packed =
    packDepth16(depth)
  | packTransmittance16(T)
  | packR11G11B10(color)
```

如果平台支持 group shared memory，应优先把 tile 内 MLAB layer 放在 group shared memory，最后 resolve 或 flush。

### 7.3 插入排序

每来一个 fragment，把它插入到当前 pixel 的 layer list。以 reverse depth 为例，越靠近相机 depth 值越大，可以用 `InterlockedMax` 协助交换。

伪代码：

```hlsl
Fragment f = incoming;

for (int i = 0; i < OIT_LAYER_COUNT; ++i)
{
    Fragment old = AtomicOrderedExchangeOrMax(pixel.layers[i], f);

    if (f is in front of old)
    {
        f = old;
    }
}

MergeIntoLastLayer(f);
```

真正实现时要根据平台 atomic 能力选择：

- 64-bit atomic：结果更稳定，首选。
- 32-bit atomic fallback：深度和颜色交换不能完全原子，可能不严格正确，但比丢失颜色信息好。
- Raster Ordered View / Pixel Sync：如果图形 API 和平台支持，也可作为硬件路径或全局 OIT 的实现方式。

### 7.4 Front-to-Back 合成公式

假设 fragment color 已经是 premultiplied contribution，`T` 是 transmittance：

```text
accColor = 0
accT = 1

for layer from front to back:
    accColor += accT * layer.color
    accT *= layer.T

outColor = sceneColor * accT + accColor
outTransmittance = accT
```

如果要把后方 fragment 合并进最后一层，使用同样的 front-to-back 规则：

```text
mergedColor = front.color + front.T * back.color
mergedT = front.T * back.T
```

这与 PPT P24 的思路一致。

## 8. Final Composite 与 TAA 关系

P27 是落地时最容易踩坑的一页：software rasterizer 已经做了空间 AA，如果最终合成后仍被普通 TAA 强力累积，会出现不必要的 blur/smear。

### 8.1 Responsive AA Mask

输出一个 responsive AA 信号给 TAA：

```text
softwareOpacity = 1 - outTransmittance
responsive = saturate(softwareOpacity * ResponsiveScale)
```

TAA resolve 时：

```text
historyWeight = lerp(DefaultHistoryWeight, ResponsiveHistoryWeight, responsive)
```

推荐初始值：

```text
DefaultHistoryWeight = 0.90 - 0.97
ResponsiveHistoryWeight = 0.10 - 0.35
ResponsiveScale = 2.0
```

含义：software hair 区域更相信当前帧，减少拖影；非 hair 区域保持普通 TAA 稳定性。

### 8.2 Depth 输出

为了让 DOF、motion blur、后处理知道前景头发存在，可根据 transmittance threshold 输出 hair depth：

```text
if (1 - outTransmittance > DepthOutputOpacityThreshold)
    outputDepth = nearestHairDepth
else
    keepSceneDepth
```

推荐：

```text
DepthOutputOpacityThreshold = 0.35
```

注意：阈值过低会让稀疏发丝过度影响 DOF；阈值过高则前景头发可能被背景后处理污染。

### 8.3 Motion Vector

硬件路径应输出准确 per-strand motion vector。

软件路径有两个选择：

- 高质量：为每个 segment 保存 previous frame endpoint，按 coverage 选择或混合 motion vector。
- 低成本：使用头/颈/发根所属骨骼的 motion vector。P27 中 RE4 使用类似近似，以节省处理时间。

建议第一版：

```text
softwareHairVelocity = characterHeadOrNeckVelocityAtHairDepth
```

第二版再升级为 per-segment previous/current projection。

## 9. LOD 与密度补偿

P29 描述手动/固定 LOD：随机打乱头发顺序，根据 LOD 百分比显示部分头发；减少数量时加粗头发；非过场降低发丝数量优先保证 gameplay。P60 补充自动 LOD：根据 strand hair AABB 占屏幕面积计算 LOD。

### 9.1 稳定随机保留

每根 strand 需要一个稳定 rank：

```text
rank = hash01(strandId)
visible = rank < lodRatio
```

不要每帧随机，否则会闪烁。LOD ratio 变化时，发丝集合应稳定增减。

### 9.2 屏幕面积自动 LOD

按 P60 思路：

```text
ScreenRate = projectedArea(hairAABB) / screenArea
LODLevel = saturate((ScreenRate - RateMin) / (RateMax - RateMin)) + MinLOD
lodRatio = clamp(LODLevel, MinLOD, 1.0)
```

更严谨写法：

```text
t = saturate((ScreenRate - RateMin) / max(RateMax - RateMin, epsilon))
lodRatio = lerp(MinLOD, 1.0, t)
```

推荐初始值：

```text
RateMin = 0.005
RateMax = 0.080
MinLOD = 0.25
GameplayLODScale = 0.50 - 0.70
CutsceneLODScale = 1.00
```

### 9.3 宽度/不透明度补偿

如果只减少发丝，头发会变稀。需要补偿。

宽度补偿：

```text
widthScale = lerp(1.0, 1.0 / max(lodRatio, 0.05), WidthCompensation)
```

建议 `WidthCompensation = 0.3 - 0.7`。完全 `1 / lodRatio` 可能把头发变成粗线，需要美术调曲线。

透明度补偿，更适合保持整体 optical density：

```text
T_compensated = pow(T_original, 1.0 / lodRatio)
alpha_compensated = 1 - T_compensated
```

推荐实践：

- software path 优先用 transmittance 补偿。
- hardware path 可用 width + alpha 混合补偿。
- root/tip 使用不同补偿强度，避免发梢变成粗毛刺。

## 10. 参数表

建议第一版暴露以下参数：

```text
HairAA.Enable = true
HairAA.TileSize = 16
HairAA.OITLayerCount = 8

HairAA.HardwareWidthThresholdPx = 1.25
HairAA.HardwareAreaThresholdPx2 = 2.0
HairAA.SoftwareMaxSegmentLengthPx = 64.0

HairAA.ReducedDepthScale = 16
HairAA.OpaqueEarlyOutTransmittance = 0.02
HairAA.DepthOutputOpacityThreshold = 0.35

HairAA.DefaultHistoryWeight = 0.95
HairAA.ResponsiveHistoryWeight = 0.25
HairAA.ResponsiveScale = 2.0

HairLOD.RateMin = 0.005
HairLOD.RateMax = 0.080
HairLOD.MinLOD = 0.25
HairLOD.WidthCompensation = 0.5
HairLOD.OpacityCompensation = 1.0
```

## 11. Debug Views

必须提供这些 debug view，否则很难调：

```text
1. Classification
   hardware = magenta, software = green, culled = black

2. Projected Width / Area
   显示每个 segment 的 widthPx 或 projectedArea

3. LOD Rank / LOD Ratio
   检查发丝是否稳定增减

4. Software Coverage
   单独显示 coverage，不含光照

5. Transmittance
   显示 outTransmittance，检查密度是否随 LOD 保持稳定

6. MLAB Layer Count / Overflow
   显示每像素使用了多少层，最后一层 merge 比例

7. Responsive AA Mask
   检查 TAA 历史权重是否只在 software hair 区域降低

8. Hair Depth Output
   检查 DOF/motion blur 使用的 hair depth 是否合理

9. Motion Vector
   硬件 path 和软件 path 分开显示
```

## 12. 分阶段实施

### Phase 1：最小可用版本

目标：能稳定画出 anti-aliased strand hair。

内容：

- strand/segment buffer。
- 屏幕投影分类。
- 硬件 ribbon pass。
- software tile rasterizer，但可以先不做 froxel，只做 tile list。
- 简化 OIT：4-8 层 MLAB。
- final composite + responsive AA mask。

验收：

- FHD 下细发丝不明显断裂。
- 开关 TAA 时，software path 不出现严重拖影。
- 静止画面无随机闪烁。

### Phase 2：性能版本

目标：把成本控制到可上线范围。

内容：

- reduced depth。
- froxel binning。
- front-to-back processing。
- tile opacity early out。
- LOD rank + 自动屏幕面积 LOD。
- group shared memory MLAB。

验收：

- software raster 时间随可见发丝数量近似线性。
- 低分辨率 software 占比高，高分辨率 hardware 占比高。
- gameplay LOD 和 cutscene LOD 切换没有明显 popping。

### Phase 3：质量版本

目标：解决后处理和极端镜头问题。

内容：

- per-segment previous/current motion vector。
- 更高质量 capsule coverage。
- 硬件 path 的 edge AA 改进。
- OIT overflow 降噪或动态 layer 策略。
- DOF/motion blur/hair depth 更精细控制。

验收：

- 快速转头、相机横移、近景发梢、强景深下仍稳定。
- MLAB overflow 热区可解释、可调。

## 13. 常见问题与处理

### 13.1 细发丝发虚

可能原因：

- TAA history weight 太高，没有 responsive AA。
- coverage 太宽或 smoothstep 范围过大。
- LOD 宽度补偿过强。

处理：

- 提高 responsive mask。
- 降低 `ResponsiveHistoryWeight`。
- 检查 coverage debug view。

### 13.2 细发丝闪烁

可能原因：

- LOD 使用了帧随机。
- segment 分类阈值在临界点抖动。
- motion vector 缺失。

处理：

- 使用 strandId hash 的稳定 rank。
- 分类加入 hysteresis。
- 软件路径至少输出骨骼近似 velocity。

### 13.3 头发变稀

可能原因：

- LOD 减少数量后没有补偿。
- transmittance 合成方向错误。
- MLAB layer 太少，后层被过度 merge。

处理：

- 启用 opacity/transmittance compensation。
- 用 front-to-back 公式检查合成。
- 打开 layer count/overflow debug view。

### 13.4 景深把头发当背景

可能原因：

- software hair 没有输出 depth。
- depth threshold 太高。

处理：

- 按 opacity threshold 输出 nearest hair depth。
- 分离 near hair depth 和 color transmittance 调试。

### 13.5 透明排序局部错误

可能原因：

- MLAB layer 不够。
- 32-bit fallback atomic 导致深度和颜色交换不完全一致。
- 发丝颜色差异大，最后一层 merge 误差变明显。

处理：

- 提高 OIT layer。
- 热点区域降 LOD 或切硬件/软件策略。
- 优先支持 64-bit atomic 或 ROV/Pixel Sync。

## 14. 性能预算参考

P28 给出的 RE4 参考数据是在 PS5、约 1920p/2160p checkerboard 场景下，hair rasterization-only 总耗时约 3.6-4.0 ms，其中 software drawing 接近 2.9-3.0 ms。该数字不是通用预算，但说明 software path 是主要成本。

建议预算拆分：

```text
Classification + setup: 0.2 - 0.6 ms
Lighting:             依光源和模型复杂度单独预算
Hardware pass:         0.2 - 0.8 ms
Software raster:       1.0 - 3.0 ms
Composite:             0.1 - 0.4 ms
```

性能优化优先级：

1. 降低进入 software path 的无效 segment。
2. reduced depth + froxel binning。
3. front-to-back early out。
4. LOD 减少 strand 数量。
5. 优化 MLAB packing 和 group shared memory 访问。

## 15. 验收清单

画质：

- FHD、1440p、4K 下分类比例符合预期。
- 低分辨率下细发丝连续，没有明显虚线感。
- TAA 开启后没有大面积 smear。
- 快速运动时不出现随机闪烁。
- LOD 变化时密度基本稳定。
- DOF/motion blur/post-process 下前景头发不被背景污染。

稳定性：

- strandId/rank 跨帧稳定。
- camera cut 后 history 正确 reset。
- dynamic resolution 改变时分类阈值仍合理。
- OIT buffer overflow 有可观测 debug。

性能：

- software raster cost 随 software segment 数量可解释。
- tile/froxel list 无异常爆炸。
- MLAB layer count 和 early-out 命中率可统计。

## 16. 最小 Shader 伪代码

```hlsl
// One compute group per 16x16 tile.
[numthreads(16, 16, 1)]
void CSHairSoftwareRaster(uint3 groupId : SV_GroupID, uint3 tid : SV_GroupThreadID)
{
    uint2 pixel = groupId.xy * 16 + tid.xy;
    InitMLABLayers(pixel);

    for each froxel in front_to_back_order
    {
        SegmentList list = GetSegments(groupId.xy, froxel);

        for each segment in list
        {
            Segment s = LoadSegment(segmentIndex);

            CoverageHit hit = EvaluateCapsuleCoverage(s, pixel);
            if (hit.coverage <= 0)
                continue;

            float depth = InterpolateDepth(s, hit.t);
            if (IsBehindOpaqueDepth(depth, pixel))
                continue;

            float3 lighting = lerp(s.light0, s.light1, hit.t);
            float T = ComputeTransmittance(s.opacity, hit.coverage);
            float3 color = lighting * s.baseColor * (1.0 - T);

            Fragment f = MakeFragment(depth, T, color);
            InsertMLAB(pixel, f);
        }

        if (TileTransmittanceBelowThreshold())
            break;
    }

    ResolveMLABToHairBuffer(pixel);
}
```

## 17. 结论

这套方案的本质是“不要让一种 AA 技术解决所有头发问题”：

- 粗发丝：硬件 rasterizer 负责几何覆盖，TAA 负责时间稳定。
- 细发丝：compute rasterizer 显式计算 coverage/transmittance，直接得到空间 AA。
- 多层透明：MLAB/OIT 负责近似顺序无关合成。
- 后处理阶段：responsive AA 和 depth threshold 避免 TAA/DOF/motion blur 破坏已经处理好的发丝。
- 性能阶段：屏幕面积分类、froxel binning、front-to-back early out 和 LOD 控制总成本。

先做 software AA + MLAB + responsive composite，再做 froxel 性能优化和 LOD，是最稳的落地路径。
