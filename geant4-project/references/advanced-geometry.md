# Advanced Geometry Reference

Geant4 provides powerful geometry construction tools beyond simple shapes.
This reference covers Boolean operations, parameterized volumes, and assembly volumes.

## Boolean Operations

Boolean operations combine solids using set operations (union, subtraction, intersection).

### G4UnionSolid — Union (OR)

```cpp
#include "G4UnionSolid.hh"

// 创建两个基本形状
G4Box* box = new G4Box("Box", 10*cm, 10*cm, 10*cm);
G4Tubs* tube = new G4Tubs("Tube", 0, 5*cm, 15*cm, 0, 360*degree);

// 创建并集
G4UnionSolid* unionSolid = new G4UnionSolid(
    "Box+Tube",        // 名称
    box,               // 第一个实体
    tube,              // 第二个实体
    0,                 // 旋转（无旋转）
    G4ThreeVector(0, 0, 5*cm)  // 第二个实体的位置
);
```

### G4SubtractionSolid — Subtraction (NOT)

```cpp
#include "G4SubtractionSolid.hh"

// 创建带孔的盒子
G4Box* box = new G4Box("Box", 10*cm, 10*cm, 10*cm);
G4Tubs* hole = new G4Tubs("Hole", 0, 3*cm, 15*cm, 0, 360*degree);

// 从盒子中减去圆柱
G4SubtractionSolid* boxWithHole = new G4SubtractionSolid(
    "Box-Hole",        // 名称
    box,               // 被减实体
    hole,              // 减去的实体
    0,                 // 旋转
    G4ThreeVector(0, 0, 0)  // 位置
);
```

### G4IntersectionSolid — Intersection (AND)

```cpp
#include "G4IntersectionSolid.hh"

// 创建两个形状的交集
G4Box* box = new G4Box("Box", 10*cm, 10*cm, 10*cm);
G4Sphere* sphere = new G4Sphere("Sphere", 0, 8*cm, 0, 360*degree, 0, 180*degree);

G4IntersectionSolid* intersection = new G4IntersectionSolid(
    "Box∩Sphere",      // 名称
    box,               // 第一个实体
    sphere,            // 第二个实体
    0,                 // 旋转
    G4ThreeVector(0, 0, 0)  // 位置
);
```

### Complex Boolean Operations

```cpp
// 多层布尔运算
// 创建一个带孔的圆柱壳
G4Tubs* outerTube = new G4Tubs("Outer", 0, 10*cm, 5*cm, 0, 360*degree);
G4Tubs* innerTube = new G4Tubs("Inner", 0, 7*cm, 6*cm, 0, 360*degree);
G4Tubs* holeTube = new G4Tubs("Hole", 0, 3*cm, 6*cm, 0, 360*degree);

// 第一步：从外圆柱减去内圆柱（创建圆柱壳）
G4SubtractionSolid* shell = new G4SubtractionSolid(
    "Shell", outerTube, innerTube
);

// 第二步：从圆柱壳减去孔
G4SubtractionSolid* shellWithHole = new G4SubtractionSolid(
    "ShellWithHole", shell, holeTube
);
```

### Rotation in Boolean Operations

```cpp
// 第二个实体有旋转
G4RotationMatrix* rot = new G4RotationMatrix();
rot->rotateZ(45.*degree);

G4UnionSolid* rotatedUnion = new G4UnionSolid(
    "RotatedUnion",
    box,
    tube,
    rot,                          // 旋转矩阵
    G4ThreeVector(5*cm, 0, 0)    // 位置
);
```

## Parameterized Volumes

Parameterized volumes allow creating multiple copies of a volume with varying properties.

### G4PVParameterised

```cpp
#include "G4PVParameterised.hh"
#include "G4VPVParameterisation.hh"

// 参数化类定义
class MyParameterisation : public G4VPVParameterisation
{
public:
    MyParameterisation(G4int nofCells);
    ~MyParameterisation() override;

    // 获取副本数量
    G4int GetNumberOfCopies() const override { return fNofCells; }

    // 计算每个副本的变换
    void ComputeTransformation(G4int copyNo, G4VPhysicalVolume* physVol) const override;

    // 计算每个副本的尺寸（可选）
    void ComputeDimensions(G4Box& box, G4int copyNo, const G4VPhysicalVolume* physVol) const override;

private:
    G4int fNofCells;
};

// 参数化类实现
MyParameterisation::MyParameterisation(G4int nofCells)
    : fNofCells(nofCells)
{}

void MyParameterisation::ComputeTransformation(G4int copyNo, G4VPhysicalVolume* physVol) const
{
    // 计算每个副本的位置
    G4double spacing = 2.0 * cm;
    G4double x = (copyNo - fNofCells/2) * spacing;
    physVol->SetTranslation(G4ThreeVector(x, 0., 0.));
}

void MyParameterisation::ComputeDimensions(G4Box& box, G4int copyNo, const G4VPhysicalVolume*) const
{
    // 可选：改变每个副本的尺寸
    G4double size = 1.0 * cm + copyNo * 0.1 * cm;
    box.SetXHalfLength(size);
    box.SetYHalfLength(size);
    box.SetZHalfLength(size);
}

// 使用参数化
G4int nCells = 20;
MyParameterisation* param = new MyParameterisation(nCells);

G4Box* cellShape = new G4Box("Cell", 1.*cm, 1.*cm, 1.*cm);
G4LogicalVolume* logicCell = new G4LogicalVolume(cellShape, material, "Cell");

new G4PVParameterised(
    "CellArray",        // 名称
    logicCell,          // 逻辑体积
    logicWorld,         // 母体积
    kXAxis,             // 复制轴
    nCells,             // 复制数量
    param               // 参数化对象
);
```

## Replicated Volumes

Replicated volumes create identical copies along an axis.

### G4PVReplica

```cpp
#include "G4PVReplica.hh"

// 创建一维复制
G4Box* cellShape = new G4Box("Cell", 1.*cm, 10.*cm, 10.*cm);
G4LogicalVolume* logicCell = new G4LogicalVolume(cellShape, material, "Cell");

// 沿 X 轴复制 20 次
G4PVReplica* replica = new G4PVReplica(
    "CellReplica",      // 名称
    logicCell,          // 逻辑体积
    logicWorld,         // 母体积
    kXAxis,             // 复制轴
    20,                 // 复制数量
    2.0*cm              // 间距
);
```

### Multi-Dimensional Replication

```cpp
// 先复制 X 轴
G4PVReplica* xReplica = new G4PVReplica(
    "XReplica", logicCell, logicWorld, kXAxis, 10, 2.0*cm
);

// 再复制 Y 轴（在 X 复制的基础上）
G4LogicalVolume* logicXRow = new G4LogicalVolume(xReplica, material, "XRow");
G4PVReplica* yReplica = new G4PVReplica(
    "YReplica", logicXRow, logicWorld, kYAxis, 10, 2.0*cm
);
```

## Assembly Volumes

Assembly volumes group multiple volumes together for easy placement.

### G4AssemblyVolume

```cpp
#include "G4AssemblyVolume.hh"

// 创建组装体积
G4AssemblyVolume* assembly = new G4AssemblyVolume();

// 添加子体积
G4Box* box = new G4Box("Box", 5*cm, 5*cm, 5*cm);
G4LogicalVolume* logicBox = new G4LogicalVolume(box, material, "Box");

G4Tubs* tube = new G4Tubs("Tube", 0, 2*cm, 10*cm, 0, 360*degree);
G4LogicalVolume* logicTube = new G4LogicalVolume(tube, material, "Tube");

// 添加到组装（相对位置）
assembly->AddPlacedVolume(logicBox, G4ThreeVector(0, 0, 0), nullptr);
assembly->AddPlacedVolume(logicTube, G4ThreeVector(0, 0, 5*cm), nullptr);

// 放置组装体积
assembly->MakeImprint(logicWorld, G4ThreeVector(10*cm, 0, 0), nullptr);
```

### Assembly with Rotation

```cpp
// 创建组装
G4AssemblyVolume* assembly = new G4AssemblyVolume();
assembly->AddPlacedVolume(logicBox, G4ThreeVector(0, 0, 0), nullptr);
assembly->AddPlacedVolume(logicTube, G4ThreeVector(0, 0, 5*cm), nullptr);

// 放置时旋转
G4RotationMatrix* rot = new G4RotationMatrix();
rot->rotateZ(45.*degree);
assembly->MakeImprint(logicWorld, G4ThreeVector(10*cm, 0, 0), rot);
```

## Divided Volumes

Divided volumes are similar to replicated volumes but with more control.

### G4PVDivision

```cpp
#include "G4PVDivision.hh"

// 创建分割体积
G4Box* motherShape = new G4Box("Mother", 10*cm, 10*cm, 10*cm);
G4LogicalVolume* logicMother = new G4LogicalVolume(motherShape, material, "Mother");

// 沿 X 轴分割为 10 个单元
G4PVDivision* division = new G4PVDivision(
    "Division",         // 名称
    logicMother,        // 母体积
    logicWorld,         // 祖母体积
    kXAxis,             // 分割轴
    10,                 // 分割数量
    0                   // 偏移
);
```

## Common Patterns

### Layered Calorimeter

```cpp
// 创建多层 calorimeter
G4int nLayers = 50;
G4double layerThickness = 0.5 * cm;
G4double absorberThickness = 1.0 * cm;
G4double totalThickness = nLayers * (layerThickness + absorberThickness);

// 吸收层
G4Box* absorberShape = new G4Box("Absorber", totalThickness/2, 10*cm, 10*cm);
G4LogicalVolume* logicAbsorber = new G4LogicalVolume(absorberShape, absorberMat, "Absorber");

// 探测层
G4Box* detectorShape = new G4Box("Detector", layerThickness/2, 10*cm, 10*cm);
G4LogicalVolume* logicDetector = new G4LogicalVolume(detectorShape, detectorMat, "Detector");

// 放置各层
for (G4int i = 0; i < nLayers; i++) {
    G4double xPos = i * (layerThickness + absorberThickness);

    // 吸收层
    new G4PVPlacement(0, G4ThreeVector(xPos, 0, 0),
                      logicAbsorber, "Absorber", logicWorld, false, i, true);

    // 探测层
    new G4PVPlacement(0, G4ThreeVector(xPos + absorberThickness/2, 0, 0),
                      logicDetector, "Detector", logicWorld, false, i, true);
}
```

### Pixelated Detector

```cpp
// 创建像素探测器
G4int nPixelsX = 100, nPixelsY = 100;
G4double pixelSize = 0.1 * mm;

G4Box* pixelShape = new G4Box("Pixel", pixelSize/2, pixelSize/2, 1*mm);
G4LogicalVolume* logicPixel = new G4LogicalVolume(pixelShape, siliconMat, "Pixel");

// 使用参数化创建像素阵列
class PixelParameterisation : public G4VPVParameterisation
{
public:
    PixelParameterisation(G4int nx, G4int ny, G4double size)
        : fNx(nx), fNy(ny), fSize(size) {}

    void ComputeTransformation(G4int copyNo, G4VPhysicalVolume* physVol) const override
    {
        G4int ix = copyNo % fNx;
        G4int iy = copyNo / fNx;
        G4double x = (ix - fNx/2) * fSize;
        G4double y = (iy - fNy/2) * fSize;
        physVol->SetTranslation(G4ThreeVector(x, y, 0.));
    }

private:
    G4int fNx, fNy;
    G4double fSize;
};

PixelParameterisation* pixelParam = new PixelParameterisation(nPixelsX, nPixelsY, pixelSize);
new G4PVParameterised("PixelArray", logicPixel, logicDetector, kUndefined,
                       nPixelsX * nPixelsY, pixelParam);
```

### Complex Detector with Boolean Operations

```cpp
// 创建一个带冷却通道的电磁量能器
G4Box* calorimeter = new G4Box("Calorimeter", 20*cm, 20*cm, 50*cm);
G4Tubs* coolingPipe = new G4Tubs("CoolingPipe", 0, 1*cm, 51*cm, 0, 360*degree);

// 减去冷却通道
G4SubtractionSolid* calorimeterWithCooling = new G4SubtractionSolid(
    "CalorimeterWithCooling",
    calorimeter,
    coolingPipe,
    0,
    G4ThreeVector(10*cm, 10*cm, 0)
);

// 添加更多冷却通道
for (G4int i = 0; i < 4; i++) {
    G4double x = (i - 2) * 10*cm;
    calorimeterWithCooling = new G4SubtractionSolid(
        "CalorimeterWithCooling",
        calorimeterWithCooling,
        coolingPipe,
        0,
        G4ThreeVector(x, 10*cm, 0)
    );
}
```

## Touchable Navigation

```cpp
// 在 SteppingAction 中导航几何
#include "G4TouchableHandle.hh"
#include "G4VTouchable.hh"
#include "G4NavigationHistory.hh"

void SteppingAction::UserSteppingAction(const G4Step* step)
{
    // 获取触摸信息
    G4TouchableHandle touchable = step->GetPreStepPoint()->GetTouchableHandle();

    // 获取体积信息
    G4String volumeName = touchable->GetVolume()->GetLogicalVolume()->GetName();
    G4int copyNo = touchable->GetCopyNumber();

    // 获取层级信息
    G4int depth = touchable->GetHistory()->GetDepth();
    G4String motherName = touchable->GetVolume(depth-1)->GetLogicalVolume()->GetName();

    // 获取世界坐标
    G4ThreeVector worldPos = step->GetPreStepPoint()->GetPosition();

    // 获取局部坐标（相对于当前体积）
    G4ThreeVector localPos = touchable->GetHistory()->GetTopTransform().TransformPoint(worldPos);
}
```

## Best Practices

1. **Overlap Checking**: Always enable during development: `new G4PVPlacement(..., true)`
2. **Naming**: Use descriptive names for debugging
3. **Performance**: Parameterized/replicated volumes are more efficient than loops
4. **Memory**: Boolean operations create new solids; reuse when possible
5. **Units**: Always specify units for positions and sizes

## Common Issues

| Problem | Solution |
|---------|----------|
| Overlap warnings | Adjust positions or use overlap checking |
| Volume not visible | Check placement and mother volume |
| Wrong orientation | Verify rotation matrix order |
| Performance issues | Use parameterized volumes for arrays |
| Navigation errors | Ensure proper volume hierarchy |
