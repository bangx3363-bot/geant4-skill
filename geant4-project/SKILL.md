---
name: geant4-project
description: >
  Create, modify, and debug Geant4 particle physics simulation projects. Use this skill whenever
  the user mentions Geant4, G4, particle simulation, Monte Carlo simulation, detector construction,
  physics lists, particle guns, beam simulation, nuclear physics experiments, or radiation transport.
  Also trigger when the user wants to simulate particles hitting targets, track particles through
  materials, or study particle interactions — even if they don't explicitly say "Geant4".
---

# Geant4 Project Skill

This skill helps users create, modify, and understand Geant4 particle physics simulation projects.
Geant4 is a toolkit for simulating the passage of particles through matter using Monte Carlo methods.

## Who is this for

Users who are learning Geant4 or need to quickly set up simulation projects. The generated code
includes Chinese comments to help Chinese-speaking learners understand each step.

## Project Structure

Every Geant4 project follows this standard layout:

```
project-name/
├── CMakeLists.txt              # Build configuration
├── project-name.cc             # Main program (entry point)
├── include/
│   ├── ActionInitialization.hh      # User action initialization (required)
│   ├── DetectorConstruction.hh      # Detector geometry header
│   ├── PrimaryGeneratorAction.hh    # Particle source header
│   ├── RunAction.hh                 # (optional) Run-level actions
│   ├── EventAction.hh               # (optional) Event-level actions
│   ├── SteppingAction.hh            # (optional) Step-level actions
│   └── SensitiveDetector.hh         # (optional) Sensitive detector
└── src/
    ├── ActionInitialization.cc      # User action initialization
    ├── DetectorConstruction.cc      # Detector geometry implementation
    ├── PrimaryGeneratorAction.cc    # Particle source implementation
    └── ...                          # Corresponding .cc files for optional headers
```

When creating a new project, always follow this structure. The naming convention is:
- Class names use PascalCase: `MyProjectDetectorConstruction`
- File names match class names: `MyProjectDetectorConstruction.hh/.cc`
- The main file uses the project name: `MyProject.cc`

## Step-by-Step Project Creation

When the user wants to create a new Geant4 project, follow this order:

### 1. Understand the simulation goal

Ask clarifying questions if the user's request is vague:
- What particles? (neutrons, photons, electrons, protons, ions...)
- What energy range? (affects physics list choice)
- What geometry? (targets, detectors, layers...)
- What data do they want to collect? (energy deposit, particle tracks, flux...)

### 2. Create CMakeLists.txt

Use this template. Change the `project()` name and executable name to match the user's project:

```cmake
# 设置项目
cmake_minimum_required(VERSION 3.16...3.21)
project( PROJECT_NAME )

# 查找 Geant4 包，启用所有 UI 和可视化驱动
option(WITH_GEANT4_UIVIS "Build with Geant4 UI and Vis drivers" ON)
if(WITH_GEANT4_UIVIS)
  find_package(Geant4 REQUIRED ui_all vis_all)
else()
  find_package(Geant4 REQUIRED)
endif()

# 设置 Geant4 包含目录
include(${Geant4_USE_FILE})
include_directories(${PROJECT_SOURCE_DIR}/include)

# 收集源文件和头文件
file(GLOB sources ${PROJECT_SOURCE_DIR}/src/*.cc)
file(GLOB headers ${PROJECT_SOURCE_DIR}/include/*.hh)

# 创建可执行文件并链接 Geant4 库
add_executable(${PROJECT_NAME} ${PROJECT_NAME}.cc ${sources} ${headers})
target_link_libraries(${PROJECT_NAME} ${Geant4_LIBRARIES})

# 安装
install(TARGETS ${PROJECT_NAME} DESTINATION bin)
```

### 3. Create main program (.cc)

The main program sets up the RunManager with three required initializations. Use `ActionInitialization` pattern (recommended by Geant4) instead of setting user actions directly.

```cpp
// C++ 标准头文件
#include <iostream>

// Geant4 头文件
#include "G4RunManagerFactory.hh"
#include "G4SteppingVerbose.hh"
#include "G4UImanager.hh"
#include "G4VisExecutive.hh"
#include "G4UIExecutive.hh"
#include "Randomize.hh"

// 用户头文件
#include "DetectorConstruction.hh"
#include "ActionInitialization.hh"
// 物理列表：选择合适的内置物理列表
#include "QGSP_BERT_HP.hh"

int main(int argc, char** argv)
{
    // 创建 UI 交互对象（如果命令行无参数则进入交互模式）
    G4UIExecutive* ui = nullptr;
    if (argc == 1) {
        ui = new G4UIExecutive(argc, argv);
    }

    // 使用 G4SteppingVerboseWithUnits（推荐）
    G4int precision = 4;
    G4SteppingVerbose::UseBestUnit(precision);

    // 创建运行管理器 —— 使用工厂模式（支持多线程）
    auto* runManager =
        G4RunManagerFactory::CreateRunManager(G4RunManagerType::Default);

    // ===== 三步初始化 =====
    // 第一步：探测器几何构造
    runManager->SetUserInitialization(new DetectorConstruction);
    // 第二步：物理过程（选择合适的物理列表）
    G4VModularPhysicsList* physicsList = new QGSP_BERT_HP;
    physicsList->SetVerboseLevel(1);
    runManager->SetUserInitialization(physicsList);
    // 第三步：用户动作初始化（使用 ActionInitialization 模式）
    runManager->SetUserInitialization(new ActionInitialization);

    // 可视化管理器
    G4VisManager* visManager = new G4VisExecutive;
    visManager->Initialize();

    // UI 管理器
    G4UImanager* UImanager = G4UImanager::GetUIpointer();

    if (ui) {
        // 交互模式：执行可视化宏文件
        UImanager->ApplyCommand("/control/execute init_vis.mac");
        ui->SessionStart();
        delete ui;
    } else {
        // 批处理模式：执行命令行指定的宏文件
        G4String command = "/control/execute ";
        G4String fileName = argv[1];
        UImanager->ApplyCommand(command + fileName);
    }

    // 清理（注意：用户动作、物理列表和探测器由运行管理器管理，不需要手动删除）
    delete visManager;
    delete runManager;

    return 0;
}
```

### 3.1 Create ActionInitialization

The `ActionInitialization` class consolidates all user actions:

```cpp
// include/ActionInitialization.hh
#ifndef ACTION_INITIALIZATION_HH
#define ACTION_INITIALIZATION_HH

#include "G4VUserActionInitialization.hh"

class ActionInitialization : public G4VUserActionInitialization
{
public:
    ActionInitialization();
    ~ActionInitialization() override;

    // 多线程模式下的主运行管理器调用
    void BuildForMaster() const override;
    // 单线程模式或工作线程调用
    void Build() const override;
};

#endif
```

```cpp
// src/ActionInitialization.cc
#include "ActionInitialization.hh"
#include "PrimaryGeneratorAction.hh"
#include "RunAction.hh"
#include "EventAction.hh"
#include "SteppingAction.hh"

ActionInitialization::ActionInitialization() {}
ActionInitialization::~ActionInitialization() {}

void ActionInitialization::BuildForMaster() const
{
    // 多线程模式：主运行管理器只需要 RunAction
    RunAction* runAction = new RunAction;
    SetUserAction(runAction);
}

void ActionInitialization::Build() const
{
    // 设置所有用户动作
    SetUserAction(new PrimaryGeneratorAction);

    RunAction* runAction = new RunAction;
    SetUserAction(runAction);

    EventAction* eventAction = new EventAction(runAction);
    SetUserAction(eventAction);

    SetUserAction(new SteppingAction(eventAction));
}
```

### 4. Create DetectorConstruction

This defines the geometry and materials. For beginners, start with simple shapes.

```cpp
// include/DetectorConstruction.hh
#ifndef DETECTOR_CONSTRUCTION_HH
#define DETECTOR_CONSTRUCTION_HH

#include "G4VUserDetectorConstruction.hh"

class G4VPhysicalVolume;

class XXXDetectorConstruction : public G4VUserDetectorConstruction
{
public:
    XXXDetectorConstruction();
    ~XXXDetectorConstruction() override;

    G4VPhysicalVolume* Construct() override;
};

#endif
```

```cpp
// src/DetectorConstruction.cc
#include "XXXDetectorConstruction.hh"

#include "G4Box.hh"
#include "G4LogicalVolume.hh"
#include "G4PVPlacement.hh"
#include "G4NistManager.hh"
#include "G4SystemOfUnits.hh"

XXXDetectorConstruction::XXXDetectorConstruction() {}
XXXDetectorConstruction::~XXXDetectorConstruction() {}

G4VPhysicalVolume* XXXDetectorConstruction::Construct()
{
    // 获取材料数据库
    G4NistManager* nist = G4NistManager::Instance();

    // ===== 定义世界体积 =====
    // 世界体积是模拟的根体积，所有其他几何体都在其中
    G4double worldSize = 1.0 * m;
    G4Material* worldMat = nist->FindOrBuildMaterial("G4_AIR");

    G4Box* solidWorld = new G4Box("World", worldSize/2, worldSize/2, worldSize/2);
    G4LogicalVolume* logicWorld = new G4LogicalVolume(solidWorld, worldMat, "World");
    G4VPhysicalVolume* physWorld = new G4PVPlacement(
        0,                  // 旋转（无旋转）
        G4ThreeVector(),    // 位置（原点）
        logicWorld,         // 逻辑体积
        "World",            // 名称
        nullptr,            // 母体积（无，因为是世界体积）
        false,              // 无布尔操作
        0,                  // 复制号
        true                // 检查重叠
    );

    // ===== 定义靶体积 =====
    G4Material* targetMat = nist->FindOrBuildMaterial("G4_Al");
    G4double targetSizeX = 20 * cm;
    G4double targetSizeY = 50 * cm;
    G4double targetSizeZ = 50 * cm;

    G4Box* solidTarget = new G4Box("Target", targetSizeX/2, targetSizeY/2, targetSizeZ/2);
    G4LogicalVolume* logicTarget = new G4LogicalVolume(solidTarget, targetMat, "Target");
    new G4PVPlacement(
        0,                          // 旋转
        G4ThreeVector(25*cm, 0, 0), // 位置
        logicTarget,                // 逻辑体积
        "Target",                   // 名称
        logicWorld,                 // 母体积
        false,
        0,
        true
    );

    return physWorld;
}
```

### 5. Create PrimaryGeneratorAction

This defines the particle source (particle gun).

```cpp
// include/PrimaryGeneratorAction.hh
#ifndef PRIMARY_GENERATOR_ACTION_HH
#define PRIMARY_GENERATOR_ACTION_HH

#include "G4VUserPrimaryGeneratorAction.hh"

class G4ParticleGun;
class G4Event;

class XXXPrimaryGeneratorAction : public G4VUserPrimaryGeneratorAction
{
public:
    XXXPrimaryGeneratorAction();
    ~XXXPrimaryGeneratorAction() override;

    void GeneratePrimaries(G4Event*) override;

private:
    G4ParticleGun* fParticleGun;
};

#endif
```

```cpp
// src/PrimaryGeneratorAction.cc
#include "XXXPrimaryGeneratorAction.hh"

#include "G4ParticleGun.hh"
#include "G4ParticleTable.hh"
#include "G4SystemOfUnits.hh"

XXXPrimaryGeneratorAction::XXXPrimaryGeneratorAction()
{
    // 创建粒子枪，每次事件发射1个粒子
    fParticleGun = new G4ParticleGun(1);

    // 获取粒子表
    G4ParticleTable* particleTable = G4ParticleTable::GetParticleTable();

    // 设置粒子类型（例如：neutron, gamma, e-, proton, alpha...）
    fParticleGun->SetParticleDefinition(particleTable->FindParticle("neutron"));

    // 设置粒子能量
    fParticleGun->SetParticleEnergy(10 * MeV);

    // 设置发射位置
    fParticleGun->SetParticlePosition(G4ThreeVector(-20 * cm, 0, 0));

    // 设置发射方向（沿 +x 方向）
    fParticleGun->SetParticleMomentumDirection(G4ThreeVector(1, 0, 0));
}

XXXPrimaryGeneratorAction::~XXXPrimaryGeneratorAction()
{
    delete fParticleGun;
}

void XXXPrimaryGeneratorAction::GeneratePrimaries(G4Event* event)
{
    fParticleGun->GeneratePrimaryVertex(event);
}
```

### 6. Provide visualization macro (init_vis.mac)

```
# init_vis.mac — 可视化初始化宏文件
/vis/open OGL
/vis/viewer/set/viewpointThetaPhi 30 30
/vis/drawVolume
/vis/viewer/set/style wireframe
/vis/scene/add/trajectories
/vis/scene/add/hits
/vis/scene/endOfEventAction accumulate
```

## Physics List Selection Guide

The physics list determines which particles and interactions are simulated. See
`references/physics-lists.md` for detailed guidance. Quick reference:

| Use case | Recommended physics list |
|----------|------------------------|
| General purpose, high energy | QGSP_BERT |
| Neutron transport (high precision) | QGSP_BERT_HP |
| Electromagnetic only | EmStandard |
| Low energy EM | EmLivermore, EmPenelope |
| Heavy ions | QGSP_BIC_HP |
| Medical physics | QGSP_BIC_EMV |

Always explain to the user why you chose a particular physics list.

## Common Geometry Shapes

See `references/geometry.md` for full reference. For advanced operations (Boolean, parameterized, assembly volumes), see `references/advanced-geometry.md`. Quick shapes:

| Shape | Class | Constructor |
|-------|-------|-------------|
| Box | G4Box | `G4Box("name", halfX, halfY, halfZ)` |
| Cylinder | G4Tubs | `G4Tubs("name", rMin, rMax, halfZ, startAngle, spanAngle)` |
| Sphere | G4Sphere | `G4Sphere("name", rMin, rMax, startPhi, spanPhi, startTheta, spanTheta)` |
| Cone | G4Cons | `G4Cons("name", rMin1, rMax1, rMin2, rMax2, halfZ, startAngle, spanAngle)` |
| Torus | G4Torus | `G4Torus("name", rMin, rMax, rtor, startPhi, spanPhi)` |

## Common Materials

Use `G4NistManager` for standard materials:
```cpp
G4NistManager* nist = G4NistManager::Instance();
nist->FindOrBuildMaterial("G4_AIR");        // 空气
nist->FindOrBuildMaterial("G4_WATER");      // 水
nist->FindOrBuildMaterial("G4_Al");         // 铝
nist->FindOrBuildMaterial("G4_Fe");         // 铁
nist->FindOrBuildMaterial("G4_Pb");         // 铅
nist->FindOrBuildMaterial("G4_Si");         // 硅
nist->FindOrBuildMaterial("G4_Ge");         // 锗
```

For custom materials or compounds, see `references/geometry.md`.

## Data Output and Analysis

For collecting simulation data, the user needs action classes:

- **RunAction**: Called at the beginning/end of a run. Good for opening/closing output files.
- **EventAction**: Called at the beginning/end of each event. Good for accumulating per-event data.
- **SteppingAction**: Called at each step of particle tracking. Good for recording energy deposits, positions, etc.

When the user wants to collect data, offer to create these classes and explain what data they should
record based on their simulation goal.

For detailed hit collection with custom data, see `references/sensitive-detectors.md`. For spatial
energy deposition or dose distributions, see `references/scoring.md`.

## Modifying Existing Projects

When the user wants to modify a Geant4 project:

1. Read the existing code first to understand the current setup
2. Identify which class needs modification
3. Explain what you're changing and why
4. Make minimal changes — don't rewrite working code unless necessary

## Common Modifications

| User wants to... | Modify this class |
|------------------|-------------------|
| Change geometry/materials | DetectorConstruction |
| Change particle type/energy/source | PrimaryGeneratorAction |
| Add more particles | PrimaryGeneratorAction |
| Record energy deposits | SteppingAction |
| Change physics processes | Physics list in main |
| Add scoring/output | RunAction + EventAction |
| Add sensitive detector | New SensitiveDetector class |
| Add magnetic/electric field | DetectorConstruction (FieldManager) |
| Create complex geometry | Boolean operations in DetectorConstruction |
| Dose distribution / energy map | Scoring system |

## Magnetic and Electric Fields

To add magnetic or electric fields, configure G4FieldManager in DetectorConstruction.
See `references/field-manager.md` for detailed guidance including uniform fields, field maps,
and stepper configuration.

## Scoring and Dose Distribution

For energy deposition maps, dose distributions, or particle flux counting, use Geant4's
scoring system. See `references/scoring.md` for mesh-based scoring, primitive scorers,
and multi-functional detectors.

## Debugging Tips

When something goes wrong, check:
1. Geometry overlaps — Geant4 will warn about overlapping volumes
2. Material names — must match Geant4 NIST database exactly
3. Units — always specify units (cm, mm, MeV, GeV, etc.)
4. Particle names — check spelling (e.g., "e-" not "electron", "gamma" not "photon")
5. Physics list — make sure it covers the energy range and particles you need

## Code Style

- Use Chinese comments for all generated code to help learners
- Explain each step with brief comments
- Include units in comments for clarity
- Use descriptive variable names in English
