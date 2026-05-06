# Sensitive Detectors and Hit Collections Reference

## Table of Contents

- [Class Hierarchy](#class-hierarchy)
- [Official Example: B2 Tracker](#official-example-b2-tracker)
- [Creating a Sensitive Detector](#creating-a-sensitive-detector)
- [Tracker Sensitive Detector](#tracker-sensitive-detector)
- [Accessing Hits in EventAction](#accessing-hits-in-eventaction)
- [Common Patterns](#common-patterns)
- [When to Use Sensitive Detectors vs SteppingAction](#when-to-use-sensitive-detectors-vs-steppingaction)

Sensitive detectors allow you to record detailed information about particle interactions
in specific volumes. This is the proper way to collect data in Geant4.

## Class Hierarchy

```
G4VSensitiveDetector
├── G4SensitiveDetector (base implementation)
└── Your custom sensitive detector
    ├── MyCalorimeterSD (energy deposition)
    └── MyTrackerSD (position tracking)

G4VHit
├── G4StepPoint (not commonly used directly)
└── Your custom hit class
    ├── MyCalorimeterHit
    └── MyTrackerHit

G4HCofThisEvent (Hit Collection of This Event)
└── G4THitsCollection<YourHitType>
```

## Official Example: B2 Tracker

The official B2 example in `/opt/geant4/examples/basic/B2` demonstrates the recommended pattern
for sensitive detectors. Key patterns to follow:

1. Use `G4Allocator` for efficient memory management of hits
2. Use `G4ThreadLocal` for thread-safe allocator storage
3. Implement custom `operator new` and `operator delete` for hits
4. Use namespaces to organize your code (e.g., `namespace B2 { ... }`)

## Creating a Sensitive Detector

### Step 1: Define the Hit Class

```cpp
// include/MyCalorimeterHit.hh
#ifndef MY_CALORIMETER_HIT_HH
#define MY_CALORIMETER_HIT_HH

#include "G4VHit.hh"
#include "G4THitsCollection.hh"
#include "G4ThreeVector.hh"

class MyCalorimeterHit : public G4VHit
{
public:
    MyCalorimeterHit();
    ~MyCalorimeterHit() override;

    // 拷贝构造和赋值运算符
    MyCalorimeterHit(const MyCalorimeterHit&);
    const MyCalorimeterHit& operator=(const MyCalorimeterHit&);

    // 比较运算符（用于排序）
    G4int operator==(const MyCalorimeterHit&) const;

    // 打印命中信息
    void Print() override;

    // ===== 数据成员 =====
    // 设置和获取能量沉积
    void SetEnergyDeposit(G4double edep) { fEnergyDeposit = edep; }
    G4double GetEnergyDeposit() const { return fEnergyDeposit; }

    // 设置和获取位置
    void SetPosition(G4ThreeVector pos) { fPosition = pos; }
    G4ThreeVector GetPosition() const { return fPosition; }

    // 设置和获取粒子类型
    void SetParticleName(G4String name) { fParticleName = name; }
    G4String GetParticleName() const { return fParticleName; }

private:
    G4double fEnergyDeposit;   // 能量沉积 (MeV)
    G4ThreeVector fPosition;   // 命中位置 (mm)
    G4String fParticleName;    // 粒子名称
};

// 定义命中集合类型
typedef G4THitsCollection<MyCalorimeterHit> MyCalorimeterHitsCollection;

#endif
```

```cpp
// src/MyCalorimeterHit.cc
#include "MyCalorimeterHit.hh"
#include "G4UnitsTable.hh"

MyCalorimeterHit::MyCalorimeterHit()
    : fEnergyDeposit(0.), fPosition(G4ThreeVector()), fParticleName("")
{}

MyCalorimeterHit::~MyCalorimeterHit() {}

MyCalorimeterHit::MyCalorimeterHit(const MyCalorimeterHit& right)
    : G4VHit(right),
      fEnergyDeposit(right.fEnergyDeposit),
      fPosition(right.fPosition),
      fParticleName(right.fParticleName)
{}

const MyCalorimeterHit& MyCalorimeterHit::operator=(const MyCalorimeterHit& right)
{
    fEnergyDeposit = right.fEnergyDeposit;
    fPosition = right.fPosition;
    fParticleName = right.fParticleName;
    return *this;
}

G4int MyCalorimeterHit::operator==(const MyCalorimeterHit& right) const
{
    return (this == &right) ? 1 : 0;
}

void MyCalorimeterHit::Print()
{
    G4cout << "能量沉积: " << G4BestUnit(fEnergyDeposit, "Energy")
           << " 位置: " << G4BestUnit(fPosition, "Length")
           << " 粒子: " << fParticleName << G4endl;
}
```

### Step 2: Define the Sensitive Detector Class

```cpp
// include/MyCalorimeterSD.hh
#ifndef MY_CALORIMETER_SD_HH
#define MY_CALORIMETER_SD_HH

#include "G4VSensitiveDetector.hh"
#include "MyCalorimeterHit.hh"

class G4Step;
class G4HCofThisEvent;
class G4TouchableHistory;

class MyCalorimeterSD : public G4VSensitiveDetector
{
public:
    MyCalorimeterSD(const G4String& name, const G4String& hitsCollectionName);
    ~MyCalorimeterSD() override;

    // 初始化（在每个事件开始时调用）
    void Initialize(G4HCofThisEvent* hitCollection) override;

    // 处理每个步（核心方法）
    G4bool ProcessHits(G4Step* step, G4TouchableHistory* history) override;

    // 结束事件处理
    void EndOfEvent(G4HCofThisEvent* hitCollection) override;

private:
    MyCalorimeterHitsCollection* fHitsCollection;  // 命中集合
};

#endif
```

```cpp
// src/MyCalorimeterSD.cc
#include "MyCalorimeterSD.hh"
#include "G4Step.hh"
#include "G4Track.hh"
#include "G4VTouchable.hh"
#include "G4HCofThisEvent.hh"
#include "G4UnitsTable.hh"

MyCalorimeterSD::MyCalorimeterSD(const G4String& name, const G4String& hitsCollectionName)
    : G4VSensitiveDetector(name),
      fHitsCollection(nullptr)
{
    collectionName.insert(hitsCollectionName);
}

MyCalorimeterSD::~MyCalorimeterSD() {}

void MyCalorimeterSD::Initialize(G4HCofThisEvent* hce)
{
    // 创建命中集合
    fHitsCollection = new MyCalorimeterHitsCollection(
        SensitiveDetectorName, collectionName[0]
    );

    // 将命中集合添加到事件中
    // GetCollectionID(0) 获取第一个集合的 ID
    G4int hcID = G4SDManager::GetSDMpointer()->GetCollectionID(collectionName[0]);
    hce->AddHitsCollection(hcID, fHitsCollection);
}

G4bool MyCalorimeterSD::ProcessHits(G4Step* step, G4TouchableHistory*)
{
    // 获取能量沉积
    G4double edep = step->GetTotalEnergyDeposit();
    if (edep <= 0.) return false;

    // 创建新的命中
    MyCalorimeterHit* hit = new MyCalorimeterHit();

    // 设置能量沉积
    hit->SetEnergyDeposit(edep);

    // 设置命中位置（使用步的中点）
    G4ThreeVector pos = step->GetPreStepPoint()->GetPosition();
    hit->SetPosition(pos);

    // 设置粒子类型
    hit->SetParticleName(step->GetTrack()->GetDefinition()->GetParticleName());

    // 添加到命中集合
    fHitsCollection->insert(hit);

    return true;
}

void MyCalorimeterSD::EndOfEvent(G4HCofThisEvent*)
{
    // 可选：在事件结束时处理命中数据
    // 例如：打印统计信息
    if (verboseLevel > 0) {
        G4int nofHits = fHitsCollection->entries();
        G4cout << "===== 命中统计 =====" << G4endl;
        G4cout << "总命中数: " << nofHits << G4endl;

        G4double totalEnergy = 0.;
        for (G4int i = 0; i < nofHits; i++) {
            totalEnergy += (*fHitsCollection)[i]->GetEnergyDeposit();
        }
        G4cout << "总能量沉积: " << G4BestUnit(totalEnergy, "Energy") << G4endl;
    }
}
```

### Step 3: Register Sensitive Detector in DetectorConstruction

```cpp
// src/DetectorConstruction.cc
#include "DetectorConstruction.hh"
#include "MyCalorimeterSD.hh"
#include "G4SDManager.hh"

G4VPhysicalVolume* DetectorConstruction::Construct()
{
    // ... 创建几何体 ...

    // ===== 注册敏感探测器 =====
    // 创建敏感探测器实例
    MyCalorimeterSD* calorimeterSD = new MyCalorimeterSD(
        "CalorimeterSD",     // 敏感探测器名称
        "CalorimeterHits"    // 命中集合名称
    );

    // 注册到 G4SDManager
    G4SDManager::GetSDMpointer()->AddNewDetector(calorimeterSD);

    // 将敏感探测器关联到逻辑体积
    logicDetector->SetSensitiveDetector(calorimeterSD);

    return physWorld;
}
```

## Tracker Sensitive Detector

For tracking particle positions along a trajectory:

```cpp
// include/MyTrackerSD.hh
class MyTrackerSD : public G4VSensitiveDetector
{
public:
    MyTrackerSD(const G4String& name, const G4String& hitsCollectionName);
    ~MyTrackerSD() override;

    void Initialize(G4HCofThisEvent* hitCollection) override;
    G4bool ProcessHits(G4Step* step, G4TouchableHistory* history) override;

private:
    MyTrackerHitsCollection* fHitsCollection;
};

// src/MyTrackerSD.cc
G4bool MyTrackerSD::ProcessHits(G4Step* step, G4TouchableHistory*)
{
    // 只记录带电粒子
    G4double charge = step->GetTrack()->GetDefinition()->GetPDGCharge();
    if (charge == 0.) return false;

    MyTrackerHit* hit = new MyTrackerHit();

    // 记录进入位置
    hit->SetPosition(step->GetPreStepPoint()->GetPosition());

    // 记录动量
    hit->SetMomentum(step->GetPreStepPoint()->GetMomentum());

    // 记录粒子类型
    hit->SetParticleName(step->GetTrack()->GetDefinition()->GetParticleName());

    // 记录时间
    hit->SetTime(step->GetPreStepPoint()->GetGlobalTime());

    fHitsCollection->insert(hit);
    return true;
}
```

## Accessing Hits in EventAction

```cpp
// src/EventAction.cc
#include "EventAction.hh"
#include "MyCalorimeterHit.hh"
#include "G4HCofThisEvent.hh"
#include "G4SDManager.hh"

void EventAction::EndOfEventAction(const G4Event* event)
{
    // 获取命中集合
    G4HCofThisEvent* hce = event->GetHCofThisEvent();
    if (!hce) return;

    // 获取命中集合 ID
    G4int hcID = G4SDManager::GetSDMpointer()->GetCollectionID("CalorimeterHits");
    MyCalorimeterHitsCollection* hitsCollection =
        static_cast<MyCalorimeterHitsCollection*>(hce->GetHC(hcID));

    if (!hitsCollection) return;

    // 遍历所有命中
    G4int nofHits = hitsCollection->entries();
    G4double totalEnergy = 0.;

    for (G4int i = 0; i < nofHits; i++) {
        MyCalorimeterHit* hit = (*hitsCollection)[i];
        totalEnergy += hit->GetEnergyDeposit();

        // 可选：写入 ntuple
        auto analysisManager = G4AnalysisManager::Instance();
        analysisManager->FillNtupleDColumn(2, hit->GetPosition().x());
        analysisManager->FillNtupleDColumn(3, hit->GetPosition().y());
        analysisManager->FillNtupleDColumn(4, hit->GetPosition().z());
    }

    // 记录总能量沉积
    fTotalEnergyDeposit = totalEnergy;
}
```

## Common Patterns

### Multiple Detectors

```cpp
// 在 DetectorConstruction 中注册多个敏感探测器
MyCalorimeterSD* calorimeterSD = new MyCalorimeterSD("CalorimeterSD", "CalHits");
MyTrackerSD* trackerSD = new MyTrackerSD("TrackerSD", "TrackerHits");

G4SDManager::GetSDMpointer()->AddNewDetector(calorimeterSD);
G4SDManager::GetSDMpointer()->AddNewDetector(trackerSD);

logicCalorimeter->SetSensitiveDetector(calorimeterSD);
logicTracker->SetSensitiveDetector(trackerSD);
```

### Filtering by Particle Type

```cpp
G4bool MyCalorimeterSD::ProcessHits(G4Step* step, G4TouchableHistory*)
{
    // 只记录伽马射线
    if (step->GetTrack()->GetDefinition()->GetParticleName() != "gamma") {
        return false;
    }

    // ... 处理命中 ...
}
```

### Filtering by Volume

```cpp
G4bool MyCalorimeterSD::ProcessHits(G4Step* step, G4TouchableHistory*)
{
    // 获取体积名称
    G4String volumeName = step->GetPreStepPoint()
        ->GetTouchableHandle()
        ->GetVolume()
        ->GetLogicalVolume()
        ->GetName();

    // 只记录特定体积
    if (volumeName != "Target") return false;

    // ... 处理命中 ...
}
```

## When to Use Sensitive Detectors vs SteppingAction

| Use Case | Recommended Approach |
|----------|---------------------|
| Simple energy deposit recording | SteppingAction (simpler) |
| Detailed hit information with custom data | SensitiveDetector (more structured) |
| Multiple detector volumes with different data | SensitiveDetector (one per volume) |
| Complex hit processing and analysis | SensitiveDetector (better organization) |
| Quick prototyping | SteppingAction (faster to implement) |
