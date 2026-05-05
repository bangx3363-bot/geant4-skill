# Data Output and Analysis Reference

Collecting data from Geant4 simulations requires action classes that hook into
different stages of the simulation.

## Action Classes Overview

| Class | When Called | Typical Use |
|-------|------------|-------------|
| RunAction | Start/end of run | Open/close files, print summary |
| EventAction | Start/end of event | Reset per-event accumulators |
| SteppingAction | Each tracking step | Record energy deposits, positions |
| TrackingAction | Start/end of track | Record particle creation |

## RunAction

```cpp
// include/RunAction.hh
#ifndef RUN_ACTION_HH
#define RUN_ACTION_HH

#include "G4UserRunAction.hh"
#include "G4Run.hh"

class RunAction : public G4UserRunAction
{
public:
    RunAction();
    ~RunAction() override;

    void BeginOfRunAction(const G4Run*) override;
    void EndOfRunAction(const G4Run*) override;
};

#endif
```

```cpp
// src/RunAction.cc
#include "RunAction.hh"
#include "G4AnalysisManager.hh"

RunAction::RunAction() {}
RunAction::~RunAction() {}

void RunAction::BeginOfRunAction(const G4Run*)
{
    // 创建分析管理器（ROOT 或 CSV 输出）
    auto analysisManager = G4AnalysisManager::Instance();
    analysisManager->OpenFile("output.root");

    // 创建直方图或 ntuple
    analysisManager->CreateNtuple("Hits", "粒子命中数据");
    analysisManager->CreateNtupleIColumn("eventID");      // 事件编号
    analysisManager->CreateNtupleDColumn("energyDeposit"); // 能量沉积
    analysisManager->CreateNtupleDColumn("x");             // x 位置
    analysisManager->CreateNtupleDColumn("y");             // y 位置
    analysisManager->CreateNtupleDColumn("z");             // z 位置
    analysisManager->FinishNtuple();
}

void RunAction::EndOfRunAction(const G4Run*)
{
    auto analysisManager = G4AnalysisManager::Instance();
    analysisManager->Write();
    analysisManager->CloseFile();
}
```

## EventAction

```cpp
// include/EventAction.hh
#ifndef EVENT_ACTION_HH
#define EVENT_ACTION_HH

#include "G4UserEventAction.hh"

class EventAction : public G4UserEventAction
{
public:
    EventAction();
    ~EventAction() override;

    void BeginOfEventAction(const G4Event*) override;
    void EndOfEventAction(const G4Event*) override;

    // 累加器：每个事件重置
    void AddEnergyDeposit(G4double edep) { fTotalEnergyDeposit += edep; }

private:
    G4double fTotalEnergyDeposit;
};

#endif
```

```cpp
// src/EventAction.cc
#include "EventAction.hh"
#include "G4AnalysisManager.hh"

EventAction::EventAction() : fTotalEnergyDeposit(0.) {}
EventAction::~EventAction() {}

void EventAction::BeginOfEventAction(const G4Event*)
{
    // 每个事件开始时重置累加器
    fTotalEnergyDeposit = 0.;
}

void EventAction::EndOfEventAction(const G4Event* event)
{
    // 事件结束时记录数据
    auto analysisManager = G4AnalysisManager::Instance();

    analysisManager->FillNtupleIColumn(0, event->GetEventID());
    analysisManager->FillNtupleDColumn(1, fTotalEnergyDeposit);
    analysisManager->AddNtupleRow();
}
```

## SteppingAction

```cpp
// include/SteppingAction.hh
#ifndef STEPPING_ACTION_HH
#define STEPPING_ACTION_HH

#include "G4UserSteppingAction.hh"

class EventAction;

class SteppingAction : public G4UserSteppingAction
{
public:
    SteppingAction(EventAction* eventAction);
    ~SteppingAction() override;

    void UserSteppingAction(const G4Step*) override;

private:
    EventAction* fEventAction;
};

#endif
```

```cpp
// src/SteppingAction.cc
#include "SteppingAction.hh"
#include "EventAction.hh"
#include "G4Step.hh"
#include "G4Track.hh"
#include "G4LogicalVolume.hh"

SteppingAction::SteppingAction(EventAction* eventAction)
    : fEventAction(eventAction) {}

SteppingAction::~SteppingAction() {}

void SteppingAction::UserSteppingAction(const G4Step* step)
{
    // 获取当前步的能量沉积
    G4double edep = step->GetTotalEnergyDeposit();
    if (edep <= 0.) return;

    // 获取位置信息
    G4ThreeVector pos = step->GetPostStepPoint()->GetPosition();

    // 获取所在体积名称（可选：只记录特定体积的数据）
    G4String volumeName = step->GetPreStepPoint()
        ->GetTouchableHandle()->GetVolume()->GetLogicalVolume()->GetName();

    // 只记录靶体积中的能量沉积
    if (volumeName == "Target") {
        fEventAction->AddEnergyDeposit(edep);

        // 写入 ntuple
        auto analysisManager = G4AnalysisManager::Instance();
        analysisManager->FillNtupleDColumn(2, pos.x());
        analysisManager->FillNtupleDColumn(3, pos.y());
        analysisManager->FillNtupleDColumn(4, pos.z());
    }
}
```

## Registering Actions in Main

```cpp
// In main program
#include "RunAction.hh"
#include "EventAction.hh"
#include "SteppingAction.hh"

// After creating runManager:
auto eventAction = new EventAction();
runManager->SetUserAction(eventAction);
runManager->SetUserAction(new RunAction());
runManager->SetUserAction(new SteppingAction(eventAction));
```

## CMakeLists.txt for ROOT Output

If using ROOT output format, add to CMakeLists.txt:
```cmake
# 查找 ROOT 包（如果需要 ROOT 输出）
find_package(ROOT REQUIRED)
include_directories(${ROOT_INCLUDE_DIRS})
target_link_libraries(${PROJECT_NAME} ${Geant4_LIBRARIES} ${ROOT_LIBRARIES})
```

## Alternative: CSV Output

For simple text output without ROOT:
```cpp
#include <fstream>

class RunAction : public G4UserRunAction
{
    std::ofstream fOutputFile;
public:
    void BeginOfRunAction(const G4Run*) override {
        fOutputFile.open("output.csv");
        fOutputFile << "eventID,energyDeposit,x,y,z" << G4endl;
    }
    void EndOfRunAction(const G4Run*) override {
        fOutputFile.close();
    }
    std::ofstream& GetOutputFile() { return fOutputFile; }
};
```

## What Data to Collect

| Simulation Goal | Record in SteppingAction |
|----------------|--------------------------|
| Energy deposit | TotalEnergyDeposit, volume name |
| Particle position | PreStepPoint/PostStepPoint position |
| Particle type | Track->GetParticleDefinition()->GetParticleName() |
| Track length | StepLength |
| Time | GlobalTime, LocalTime |
| Momentum | PostStepPoint momentum |
| Which volume hit | TouchableHandle volume name |

## Analysis with Python

After generating ROOT files, users often analyze with Python:
```python
import uproot
import matplotlib.pyplot as plt

# 读取 ROOT 文件
file = uproot.open("output.root")
tree = file["Hits"]

# 读取数据
energy = tree["energyDeposit"].array()
x = tree["x"].array()

# 绘制能量沉积分布
plt.hist(energy, bins=100)
plt.xlabel("Energy Deposit (MeV)")
plt.ylabel("Counts")
plt.title("Energy Deposition in Target")
plt.savefig("energy_deposit.png")
```
