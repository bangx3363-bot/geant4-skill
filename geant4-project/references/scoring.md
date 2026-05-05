# Scoring Reference

Geant4 provides a scoring system for collecting spatial distributions of quantities
like energy deposition, particle flux, and dose. This is useful for:
- Dose distribution in medical physics
- Energy deposition calorimetry
- Particle flux mapping
- Activation studies

## Scoring Methods

### 1. Mesh-Based Scoring (Recommended for Beginners)

Uses predefined mesh shapes to collect spatial data.

### 2. Primitive Scorers

Lower-level scorers that can be attached to sensitive detectors.

### 3. Multi-Functional Detectors

Combine multiple scorers in one sensitive detector.

## Mesh-Based Scoring

### Using Macro Commands

The easiest way to use scoring is through macro commands:

```mac
# score.mac — 计分设置

# ===== 创建计分网格 =====
# 创建盒形网格
/score/create/boxMesh boxMesh

# 设置网格参数
# 参数：名称 nx ny nz (网格单元数)
# 或者：名称 xHalf yHalf zHalf (半尺寸)
/score/mesh/boxSize 50. 50. 50. cm
/score/mesh/nBin 10 10 10

# 设置网格位置（可选）
/score/mesh/translate/xyz 0. 0. 0.

# ===== 添加计分量 =====
# 能量沉积
/score/quantity/energyDeposit eDep

# 粒子通量
/score/quantity/flux flux

# 粒子数量
/score/quantity/nOfTrack nTracks

# 剂量
/score/quantity/doseDeposit dose

# ===== 体积过滤 =====
# 只计分特定体积中的事件
/score/filter/particle gamma
/score/filter/particle e-

# ===== 输出结果 =====
# 输出到文件
/score/close

# 在屏幕上显示
/score/listAllScorer
```

### Programmatic Scoring

```cpp
// src/DetectorConstruction.cc
#include "G4ScoringManager.hh"
#include "G4ScoringBox.hh"
#include "G4PSEnergyDeposit.hh"
#include "G4MultiFunctionalDetector.hh"
#include "G4SDManager.hh"

G4VPhysicalVolume* DetectorConstruction::Construct()
{
    // ... 创建几何体 ...

    // ===== 创建计分网格 =====
    G4ScoringBox* scoringBox = new G4ScoringBox(
        "ScoringBox",      // 名称
        logicWorld,        // 母体积
        50.*cm,            // x 半尺寸
        50.*cm,            // y 半尺寸
        50.*cm,            // z 半尺寸
        10,                // x 网格数
        10,                // y 网格数
        10                 // z 网格数
    );

    // 创建多探测器
    G4MultiFunctionalDetector* scorer = new G4MultiFunctionalDetector("Scorer");

    // 添加能量沉积计分器
    G4VPrimitiveScorer* energyScorer = new G4PSEnergyDeposit("eDep");
    scorer->RegisterPrimitive(energyScorer);

    // 添加通量计分器
    G4VPrimitiveScorer* fluxScorer = new G4PSFlatSurfaceCurrent("flux");
    scorer->RegisterPrimitive(fluxScorer);

    // 注册到敏感探测器管理器
    G4SDManager::GetSDMpointer()->AddNewDetector(scorer);

    // 将计分器关联到网格
    scoringBox->SetSensitiveDetector(scorer);

    return physWorld;
}
```

## Primitive Scorers

### Available Scorers

| Scorer | Description | Unit |
|--------|-------------|------|
| G4PSEnergyDeposit | Energy deposit | MeV |
| G4PSFlatSurfaceCurrent | Particle current through flat surface | particle |
| G4PSFlatSurfaceFlux | Particle flux through flat surface | particle/cm² |
| G4PSPassageCellCurrent | Particle current through volume | particle |
| G4PSPassageCellFlux | Particle flux through volume | particle/cm² |
| G4PSNofStep | Number of steps | step |
| G4PSTrackLength | Track length | mm |
| G4PSTrackLengthEstimator | Track length estimator | mm |
| G4PSTimeOfArrival | Time of arrival | ns |

### Multi-Functional Detector Example

```cpp
// 创建多探测器
G4MultiFunctionalDetector* mfd = new G4MultiFunctionalDetector("MyDetector");

// 添加多个计分器
mfd->RegisterPrimitive(new G4PSEnergyDeposit("eDep"));
mfd->RegisterPrimitive(new G4PSNofStep("nSteps"));
mfd->RegisterPrimitive(new G4PSTrackLength("trackLength"));

// 注册到 SDManager
G4SDManager::GetSDMpointer()->AddNewDetector(mfd);

// 关联到逻辑体积
logicDetector->SetSensitiveDetector(mfd);
```

## Scoring Box Configuration

```cpp
// 创建计分盒
G4ScoringBox* scoringBox = new G4ScoringBox(
    "Name",         // 名称
    motherVolume,   // 母体积
    halfX,          // x 半尺寸
    halfY,          // y 半尺寸
    halfZ,          // z 半尺寸
    nBinsX,         // x 方向网格数
    nBinsY,         // y 方向网格数
    nBinsZ          // z 方向网格数
);

// 设置位置
scoringBox->SetTranslation(G4ThreeVector(0., 0., 0.));

// 设置旋转
G4RotationMatrix* rot = new G4RotationMatrix();
rot->rotateZ(45.*degree);
scoringBox->SetRotation(rot);
```

## Scoring Cylinder

```cpp
// 创建圆柱形计分网格
G4ScoringCylinder* scoringCylinder = new G4ScoringCylinder(
    "CylinderScorer",
    motherVolume,
    radius,         // 半径
    halfZ,          // 半高
    nBinsR,         // 径向网格数
    nBinsPhi,       // 角度网格数
    nBinsZ          // z 方向网格数
);
```

## Scoring Sphere

```cpp
// 创建球形计分网格
G4ScoringSphere* scoringSphere = new G4ScoringSphere(
    "SphereScorer",
    motherVolume,
    radius,         // 半径
    nBinsR,         // 径向网格数
    nBinsTheta,     // 极角网格数
    nBinsPhi        // 方位角网格数
);
```

## Filtering

### Particle Filters

```cpp
// 只计分特定粒子
#include "G4PSFilter.hh"

// 创建过滤器
G4PSFilter* gammaFilter = new G4PSFilter();
gammaFilter->add("gamma");
gammaFilter->add("e-");

// 应用到计分器
energyScorer->SetFilter(gammaFilter);
```

### Volume Filters

```cpp
// 只计分特定体积
#include "G4PSTrackLength.hh"

G4PSTrackLength* trackLengthScorer = new G4PSTrackLength("trackLength");

// 设置体积过滤器
G4String volumeName = "Target";
trackLengthScorer->SetVolumeFilter(volumeName);
```

## Reading Scoring Results

### Via Macro Commands

```mac
# 查看计分结果
/score/listAllScorer

# 导出到文件
/score/dump/quantityToFile boxMesh eDep eDep.txt
/score/dump/quantityToFile boxMesh flux flux.txt

# 在屏幕上显示
/score/dump/quantityToScreen boxMesh eDep
```

### Programmatic Access

```cpp
// 在 RunAction 中读取计分结果
#include "G4ScoringManager.hh"
#include "G4ScoringBox.hh"

void RunAction::EndOfRunAction(const G4Run*)
{
    // 获取计分管理器
    G4ScoringManager* scoringManager = G4ScoringManager::GetScoringManager();

    // 获取计分网格
    G4ScoringBox* scoringBox = dynamic_cast<G4ScoringBox*>(
        scoringManager->FindScoringWorld("ScoringBox")
    );

    if (scoringBox) {
        // 获取计分结果
        G4THitsMap<G4double>* energyMap = scoringBox->GetHitsMap("eDep");

        // 遍历结果
        std::map<G4int, G4double*>::iterator it;
        for (it = energyMap->GetMap()->begin(); it != energyMap->GetMap()->end(); it++) {
            G4int cellID = it->first;
            G4double energy = *(it->second);

            // 写入文件或 ntuple
            G4cout << "Cell " << cellID << ": " << energy/MeV << " MeV" << G4endl;
        }
    }
}
```

## Output Formats

### ASCII Output

```mac
# 输出为文本文件
/score/dump/quantityToFile boxMesh eDep eDep_ascii.txt
```

### ROOT Output

```cmake
# CMakeLists.txt 中添加 ROOT 支持
find_package(ROOT REQUIRED)
include_directories(${ROOT_INCLUDE_DIRS})
target_link_libraries(${PROJECT_NAME} ${Geant4_LIBRARIES} ${ROOT_LIBRARIES})
```

### CSV Output for Python Analysis

```cpp
// 自定义 CSV 输出
void RunAction::EndOfRunAction(const G4Run*)
{
    std::ofstream csvFile("scoring_results.csv");
    csvFile << "cellID,x,y,z,energyDeposit" << G4endl;

    // ... 遍历计分结果 ...

    csvFile.close();
}
```

## Common Patterns

### Dose Distribution in Medical Physics

```mac
# 创建人体模型计分网格
/score/create/boxMesh doseMesh
/score/mesh/boxSize 20. 30. 40. cm
/score/mesh/nBin 100 150 200

# 计分剂量
/score/quantity/doseDeposit dose

# 输出
/score/dump/quantityToFile doseMesh dose dose_distribution.txt
```

### Calorimeter Energy Deposition

```cpp
// 多层 calorimeter 计分
for (G4int layer = 0; layer < nLayers; layer++) {
    G4String name = "Layer" + std::to_string(layer);
    G4ScoringBox* layerScorer = new G4ScoringBox(
        name,
        logicCalorimeter,
        layerThickness, absorberThickness, absorberThickness,
        1, 1, 1  // 每层一个网格单元
    );
    layerScorer->SetTranslation(G4ThreeVector(0., 0., layer * layerSpacing));
}
```

### Particle Flux Mapping

```mac
# 创建通量映射
/score/create/boxMesh fluxMap
/score/mesh/boxSize 1. 1. 1. m
/score/mesh/nBin 50 50 50

# 添加通量计分
/score/quantity/flux flux

# 过滤中子
/score/filter/particle neutron

# 输出
/score/dump/quantityToFile fluxMap flux neutron_flux.txt
```

## Best Practices

1. **Grid Resolution**: Start with coarse grids for testing, refine as needed
2. **Memory Usage**: Large grids consume significant memory
3. **Run Time**: Scoring adds overhead; use only when needed
4. **Units**: Always check output units (MeV, particle, particle/cm², etc.)
5. **Validation**: Compare scoring results with analytical calculations when possible

## Common Issues

| Problem | Solution |
|---------|----------|
| No scoring output | Check that sensitive detector is registered and linked |
| Wrong units | Verify scorer type and output format |
| Memory overflow | Reduce grid size or use fewer scorers |
| Scoring not active | Ensure scoring macro is executed before /run/beamOn |
