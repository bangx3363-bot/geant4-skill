# Visualization Reference

Geant4 supports multiple visualization systems. For beginners, OpenGL (OGL) is recommended.

## Visualization Drivers

| Driver | Command | Description |
|--------|---------|-------------|
| OpenGL | `/vis/open OGL` | Best for interactive use (recommended) |
| Qt | `/vis/open Qt` | Qt-based viewer with more UI features |
| RayTracer | `/vis/open RayTracer` | High quality ray-traced images |
| VRML | `/vis/open VRML` | For VRML2 output |
| HepRep | `/vis/open HepRepFile` | For HepRep XML output |
| DAWN | `/vis/open DAWN` | For PostScript output |

## Basic Visualization Macro (init_vis.mac)

```mac
# init_vis.mac — 可视化初始化

# 打开 OpenGL 窗口
/vis/open OGL

# 设置观察角度（theta, phi）
/vis/viewer/set/viewpointThetaPhi 30 30

# 设置观察距离
/vis/viewer/set/viewpointDistance 50 cm

# 绘制几何体
/vis/drawVolume

# 显示样式：wireframe（线框）或 surface（表面）
/vis/viewer/set/style wireframe

# 添加轨迹显示
/vis/scene/add/trajectories

# 添加命中点显示
/vis/scene/add/hits

# 每个事件结束后累积显示（不刷新）
/vis/scene/endOfEventAction accumulate

# 设置刷新频率
/vis/scene/endOfRunAction accumulate
```

## Viewing Styles

```mac
# 线框模式（显示所有边缘）
/vis/viewer/set/style wireframe

# 表面模式（实体渲染）
/vis/viewer/set/style surface

# 透视模式（透明显示内部结构）
/vis/viewer/set/style surface
/vis/viewer/set/transparency 0.5

# 显示坐标轴
/vis/scene/add/axes
```

## Camera Controls

```mac
# 设置观察角度
/vis/viewer/set/viewpointThetaPhi 90 0    # 从 x 轴看
/vis/viewer/set/viewpointThetaPhi 0 90    # 从 z 轴看（俯视）
/vis/viewer/set/viewpointThetaPhi 30 30   # 等轴测视角

# 缩放
/vis/viewer/zoom 2.0        # 放大 2 倍
/vis/viewer/zoom 0.5        # 缩小 2 倍

# 平移
/vis/viewer/pan 1 cm 1 cm

# 自动调整大小以显示所有几何体
/vis/viewer/set/autoRefresh true
```

## Color Coding by Volume

```mac
# 设置特定体积的颜色
/vis/geometry/set/colour World 0.5 0.5 0.5 0.1    # 半透明灰色
/vis/geometry/set/colour Target 1 0 0 0.8          # 红色
/vis/geometry/set/colour Detector 0 1 0 0.8        # 绿色
```

## Trajectory Display

```mac
# 显示轨迹
/vis/scene/add/trajectories

# 轨迹颜色方案
/vis/modeling/trajectories/create/drawByCharge
/vis/modeling/trajectories/drawByCharge-0/default/setDrawStepLine true

# 按粒子类型着色
/vis/modeling/trajectories/create/drawByParticleID
/vis/modeling/trajectories/drawByParticleID-0/default/setDrawStepLine true

# 限制显示的轨迹数（避免内存问题）
/vis/scene/endOfEventAction accumulate 100
```

## Saving Images

```mac
# 保存当前视图为 PNG
/vis/ogl/printPNG filename.png

# 保存为 EPS
/vis/ogl/printEPS filename.eps

# 保存为 PDF
/vis/ogl/printPDF filename.pdf
```

## Animation

```mac
# 创建旋转动画
/vis/enable true
/vis/viewer/set/autoRefresh false
/vis/sceneHandler/attachCurrentScene

# 旋转 360 度，每步 10 度
/vis/viewer/set/globalLineWidthScale 2
/vis/viewer/set/viewpointThetaPhi 90 0
/vis/ogl/printPNG frame_000.png
/vis/viewer/set/viewpointThetaPhi 90 30
/vis/ogl/printPNG frame_030.png
# ... repeat for each angle
```

## Batch Mode Visualization

For running without display (e.g., on a server):
```bash
# 生成图像文件而不显示窗口
./myproject run_with_vis.mac
```

In the macro file:
```mac
# 使用 DAWN 生成 PostScript（无需显示）
/vis/open DAWNFILE
/vis/drawVolume
/vis/ogl/printPNG output.png
```

## Performance Tips

1. **Disable visualization for batch runs**: Add `/vis/disable` at the start of batch macros
2. **Limit trajectories**: Too many trajectories can make the viewer slow
3. **Use wireframe for complex geometries**: Faster than surface rendering
4. **Reduce transparency**: Transparent volumes are slower to render

## Common Issues

| Problem | Solution |
|---------|----------|
| Black window | Check if display is available, try `/vis/viewer/flush` |
| No trajectories shown | Make sure `/vis/scene/add/trajectories` is in macro |
| Slow rendering | Reduce trajectory count, use wireframe |
| Geometry not visible | Check `/vis/drawVolume` is called after geometry is built |
| Colors not showing | Check alpha values (4th parameter) are not 0 |
