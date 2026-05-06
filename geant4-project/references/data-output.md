# Data Output and Analysis Reference

## Table of Contents

- [Action Classes Overview](#action-classes-overview)
- [Pattern 1: Per-Event Aggregation](#pattern-1-per-event-aggregation)
- [Pattern 2: Per-Step Recording](#pattern-2-per-step-recording)
- [Registering Actions via ActionInitialization](#registering-actions-via-actioninitialization)
- [CMakeLists.txt for ROOT Output](#cmakeliststxt-for-root-output)
- [Alternative: CSV Output](#alternative-csv-output)
- [What Data to Collect](#what-data-to-collect)
- [Analysis with Python](#analysis-with-python)

## Action Classes Overview

| Class | When Called | Typical Use |
|-------|------------|-------------|
| RunAction | Start/end of run | Open/close files, print summary |
| EventAction | Start/end of event | Reset per-event accumulators |
| SteppingAction | Each tracking step | Record energy deposits, positions |
| TrackingAction | Start/end of track | Record particle creation |

**Important**: Register actions via `ActionInitialization`, not `SetUserAction` in main().
See [Registering Actions via ActionInitialization](#registering-actions-via-actioninitialization).

---

## Pattern 1: Per-Event Aggregation

Use this pattern when you want **one row per event** in the output (e.g., total energy deposit per event).

**Data flow**: SteppingAction accumulates → EventAction writes one row at event end

### RunAction

```cpp
// include/RunAction.hh
#ifndef RUN_ACTION_HH
#define RUN_ACTION_HH
#include "G4UserRunAction.hh"

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
    auto analysisManager = G4AnalysisManager::Instance();
    analysisManager->OpenFile("output.root");

    // 创建 ntuple（每行对应一个事件）
    analysisManager->CreateNtuple("Events", "Per-event data");
    analysisManager->CreateNtupleIColumn("eventID");      // 列 0
    analysisManager->CreateNtupleDColumn("energyDeposit"); // 列 1
    analysisManager->FinishNtuple();
}

void RunAction::EndOfRunAction(const G4Run*)
{
    auto analysisManager = G4AnalysisManager::Instance();
    analysisManager->Write();
    analysisManager->CloseFile();
}
```

### EventAction

```cpp
// include/EventAction.hh
#ifndef EVENT_ACTION_HH
#define EVENT_ACTION_HH
#include "G4UserEventAction.hh"

class RunAction;

class EventAction : public G4UserEventAction
{
public:
    EventAction(RunAction* runAction);
    ~EventAction() override;
    void BeginOfEventAction(const G4Event*) override;
    void EndOfEventAction(const G4Event*) override;
    void AddEnergyDeposit(G4double edep) { fEdep += edep; }

private:
    RunAction* fRunAction;
    G4double fEdep = 0.;
};
#endif
```

```cpp
// src/EventAction.cc
#include "EventAction.hh"
#include "RunAction.hh"
#include "G4AnalysisManager.hh"

EventAction::EventAction(RunAction* runAction)
    : fRunAction(runAction) {}

EventAction::~EventAction() {}

void EventAction::BeginOfEventAction(const G4Event*)
{
    fEdep = 0.;  // 每个事件开始时重置
}

void EventAction::EndOfEventAction(const G4Event* event)
{
    // 事件结束时写入一行数据
    auto analysisManager = G4AnalysisManager::Instance();
    analysisManager->FillNtupleIColumn(0, event->GetEventID());
    analysisManager->FillNtupleDColumn(1, fEdep);
    analysisManager->AddNtupleRow();  // 提交这一行
}
```

### SteppingAction

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

SteppingAction::SteppingAction(EventAction* eventAction)
    : fEventAction(eventAction) {}

SteppingAction::~SteppingAction() {}

void SteppingAction::UserSteppingAction(const G4Step* step)
{
    G4double edep = step->GetTotalEnergyDeposit();
    if (edep <= 0.) return;

    // 只累加能量，不写 ntuple（由 EventAction 统一写入）
    fEventAction->AddEnergyDeposit(edep);
}
```

---

## Pattern 2: Per-Step Recording

Use this pattern when you want **one row per step** (e.g., recording each hit's position and energy).

**Data flow**: EventAction caches eventID → SteppingAction reads it and writes one row per step

### RunAction

```cpp
// src/RunAction.cc
#include "RunAction.hh"
#include "G4AnalysisManager.hh"

RunAction::RunAction() {}
RunAction::~RunAction() {}

void RunAction::BeginOfRunAction(const G4Run*)
{
    auto analysisManager = G4AnalysisManager::Instance();
    analysisManager->OpenFile("output.root");

    // 创建 ntuple（每行对应一个步）
    analysisManager->CreateNtuple("Steps", "Per-step data");
    analysisManager->CreateNtupleIColumn("eventID");  // 列 0
    analysisManager->CreateNtupleDColumn("edep");      // 列 1
    analysisManager->CreateNtupleDColumn("x");         // 列 2
    analysisManager->CreateNtupleDColumn("y");         // 列 3
    analysisManager->CreateNtupleDColumn("z");         // 列 4
    analysisManager->CreateNtupleSColumn("volume");    // 列 5
    analysisManager->FinishNtuple();
}

void RunAction::EndOfRunAction(const G4Run*)
{
    auto analysisManager = G4AnalysisManager::Instance();
    analysisManager->Write();
    analysisManager->CloseFile();
}
```

### EventAction

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
    void BeginOfEventAction(const G4Event* event) override;
    G4int GetEventID() const { return fEventID; }

private:
    G4int fEventID = 0;
};
#endif
```

```cpp
// src/EventAction.cc
#include "EventAction.hh"

EventAction::EventAction() {}
EventAction::~EventAction() {}

void EventAction::BeginOfEventAction(const G4Event* event)
{
    fEventID = event->GetEventID();  // 缓存当前事件 ID
}
```

### SteppingAction

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
    void UserSteppingAction(const G4Step* step) override;

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
#include "G4LogicalVolume.hh"
#include "G4AnalysisManager.hh"

SteppingAction::SteppingAction(EventAction* eventAction)
    : fEventAction(eventAction) {}

SteppingAction::~SteppingAction() {}

void SteppingAction::UserSteppingAction(const G4Step* step)
{
    G4double edep = step->GetTotalEnergyDeposit();
    if (edep <= 0.) return;

    // 可选：只记录特定体积
    G4String volumeName = step->GetPreStepPoint()
        ->GetTouchableHandle()->GetVolume()->GetLogicalVolume()->GetName();
    if (volumeName != "Target") return;

    // 获取位置
    G4ThreeVector pos = step->GetPostStepPoint()->GetPosition();

    // 直接写入 ntuple（每步一行）
    // 注意：G4Track 没有 GetEventID()，通过 EventAction 缓存获取
    auto analysisManager = G4AnalysisManager::Instance();
    analysisManager->FillNtupleIColumn(0, fEventAction->GetEventID());
    analysisManager->FillNtupleDColumn(1, edep);
    analysisManager->FillNtupleDColumn(2, pos.x());
    analysisManager->FillNtupleDColumn(3, pos.y());
    analysisManager->FillNtupleDColumn(4, pos.z());
    analysisManager->FillNtupleSColumn(5, volumeName);
    analysisManager->AddNtupleRow();  // 每步提交一行
}
```

---

## Registering Actions via ActionInitialization

**Always use `ActionInitialization`** — do not call `SetUserAction` in main().

Use the registration pattern that matches the data collection pattern.

### Per-event aggregation registration

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
    SetUserAction(new RunAction);
}

void ActionInitialization::Build() const
{
    SetUserAction(new PrimaryGeneratorAction);

    auto runAction = new RunAction;
    SetUserAction(runAction);

    // 每事件汇总：EventAction 需要 RunAction，并在事件结束时写入一行
    auto eventAction = new EventAction(runAction);
    SetUserAction(eventAction);

    // SteppingAction 通过 eventAction->AddEnergyDeposit() 累加能量
    SetUserAction(new SteppingAction(eventAction));
}
```

### Per-step recording registration

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
    SetUserAction(new RunAction);
}

void ActionInitialization::Build() const
{
    SetUserAction(new PrimaryGeneratorAction);

    auto runAction = new RunAction;
    SetUserAction(runAction);

    // 每步记录：EventAction 只缓存当前 eventID，所以使用无参构造
    auto eventAction = new EventAction;
    SetUserAction(eventAction);

    // SteppingAction 通过 eventAction->GetEventID() 获取事件编号
    SetUserAction(new SteppingAction(eventAction));
}
```

---

## CMakeLists.txt for ROOT Output

```cmake
# 查找 ROOT 包（如果需要 ROOT 输出）
find_package(ROOT REQUIRED)
include_directories(${ROOT_INCLUDE_DIRS})
target_link_libraries(${PROJECT_NAME} ${Geant4_LIBRARIES} ${ROOT_LIBRARIES})
```

---

## Alternative: CSV Output

For simple text output without ROOT:

```cpp
// src/RunAction.cc
#include "RunAction.hh"
#include <fstream>

static std::ofstream outputFile;

void RunAction::BeginOfRunAction(const G4Run*)
{
    outputFile.open("output.csv");
    outputFile << "eventID,energyDeposit" << G4endl;
}

void RunAction::EndOfRunAction(const G4Run*)
{
    outputFile.close();
}

// 在 EventAction 或 SteppingAction 中写入：
// outputFile << eventID << "," << edep << G4endl;
```

---

## What Data to Collect

| Simulation Goal | Where to Record | Pattern |
|----------------|-----------------|---------|
| Total energy per event | EventAction::EndOfEventAction | Per-event |
| Energy deposit per volume | SteppingAction | Per-step |
| Particle position at each step | SteppingAction | Per-step |
| Track length | SteppingAction | Per-step |
| Time of arrival | SteppingAction | Per-step |
| Particle type | SteppingAction | Per-step |
| Number of steps per event | EventAction counter | Per-event |

---

## Analysis with Python

```python
import uproot
import matplotlib.pyplot as plt

# 读取 ROOT 文件
file = uproot.open("output.root")
tree = file["Events"]  # 或 "Steps"

# 读取数据
energy = tree["energyDeposit"].array()

# 绘制能量沉积分布
plt.hist(energy, bins=100)
plt.xlabel("Energy Deposit (MeV)")
plt.ylabel("Counts")
plt.title("Energy Deposition")
plt.savefig("energy_deposit.png")
```
