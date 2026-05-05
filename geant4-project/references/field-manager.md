# Field Manager Reference

Geant4 supports magnetic and electric fields that affect charged particle trajectories.
Fields are configured through G4FieldManager objects attached to logical volumes.

## Basic Concepts

- **G4FieldManager**: Manages field properties for a volume
- **G4MagneticField**: Base class for magnetic fields
- **G4ElectricField**: Base class for electric fields
- **G4UniformMagField**: Uniform magnetic field (constant B)
- **G4UniformElectricField**: Uniform electric field (constant E)
- **G4TransportationManager**: Manages global field setup

## Official Example: B5 Magnetic Field

The official B5 example in `/opt/geant4/examples/basic/B5` demonstrates the recommended pattern
for magnetic fields. Key patterns:

1. Create a custom `G4MagneticField` class with `G4GenericMessenger` for runtime control
2. Use `ConstructSDandField()` method in DetectorConstruction for field setup
3. Use `G4ThreadLocal` for thread-safe field storage

## Uniform Magnetic Field

### Simple Setup in DetectorConstruction

```cpp
// src/DetectorConstruction.cc
#include "DetectorConstruction.hh"
#include "G4UniformMagField.hh"
#include "G4FieldManager.hh"
#include "G4TransportationManager.hh"
#include "G4SystemOfUnits.hh"

G4VPhysicalVolume* DetectorConstruction::Construct()
{
    // ... 创建几何体 ...

    // ===== 设置均匀磁场 =====
    // 磁场强度：1 Tesla，方向沿 Z 轴
    G4ThreeVector fieldDirection(0., 0., 1.);
    G4double fieldStrength = 1.0 * tesla;
    G4UniformMagField* magField = new G4UniformMagField(fieldDirection * fieldStrength);

    // 获取场管理器
    G4FieldManager* fieldManager = new G4FieldManager(magField);

    // 设置场管理器到逻辑体积
    logicWorld->SetFieldManager(fieldManager, true);

    return physWorld;
}
```

### Field for Specific Volume

```cpp
// 磁场只在特定体积中生效
G4UniformMagField* detectorField = new G4UniformMagField(
    G4ThreeVector(0., 0., 0.5 * tesla)
);

G4FieldManager* detectorFieldManager = new G4FieldManager(detectorField);
logicDetector->SetFieldManager(detectorFieldManager, true);
```

## Uniform Electric Field

```cpp
// 设置均匀电场
G4ThreeVector fieldDirection(0., 1., 0.);  // 沿 Y 方向
G4double fieldStrength = 100. * kilovolt / cm;
G4UniformElectricField* eField = new G4UniformElectricField(
    fieldDirection * fieldStrength
);

G4FieldManager* fieldManager = new G4FieldManager(eField);
logicDetector->SetFieldManager(fieldManager, true);
```

## Electromagnetic Field (Combined)

```cpp
// 同时设置电场和磁场
#include "G4ElectroMagneticField.hh"

class MyEMField : public G4ElectroMagneticField
{
public:
    MyEMField() {}
    ~MyEMField() override {}

    void GetFieldValue(const G4double point[4], G4double* field) const override
    {
        // point[0-2]: x, y, z 位置 (mm)
        // point[3]: 时间 (ns)
        // field[0-2]: 电场 Ex, Ey, Ez (V/m)
        // field[3-5]: 磁场 Bx, By, Bz (Tesla)

        // 示例：均匀电磁场
        field[0] = 0.;          // Ex
        field[1] = 0.;          // Ey
        field[2] = 100.*kilovolt/cm;  // Ez
        field[3] = 0.;          // Bx
        field[4] = 0.;          // By
        field[5] = 1.0*tesla;   // Bz
    }

    G4bool DoesFieldChangeEnergy() const override { return true; }
};

// 使用自定义电磁场
MyEMField* emField = new MyEMField();
G4FieldManager* fieldManager = new G4FieldManager(emField);
logicDetector->SetFieldManager(fieldManager, true);
```

## Non-Uniform Fields

### Field Map

For spatially varying fields, use G4FieldManager with a custom field class:

```cpp
// include/MyFieldMap.hh
class MyFieldMap : public G4MagneticField
{
public:
    MyFieldMap(const G4String& filename);
    ~MyFieldMap() override;

    void GetFieldValue(const G4double point[4], G4double* field) const override;

private:
    // 存储场数据的数组
    std::vector<G4double> fBx, fBy, fBz;
    G4double fXmin, fXmax, fYmin, fYmax, fZmin, fZmax;
    G4int fNx, fNy, fNz;
};

// src/MyFieldMap.cc
void MyFieldMap::GetFieldValue(const G4double point[4], G4double* field) const
{
    G4double x = point[0], y = point[1], z = point[2];

    // 检查是否在场区域内
    if (x < fXmin || x > fXmax || y < fYmin || y > fYmax || z < fZmin || z > fZmax) {
        field[0] = 0.;
        field[1] = 0.;
        field[2] = 0.;
        return;
    }

    // 插值计算场值（简化示例）
    G4int ix = (G4int)((x - fXmin) / (fXmax - fXmin) * fNx);
    G4int iy = (G4int)((y - fYmin) / (fYmax - fYmin) * fNy);
    G4int iz = (G4int)((z - fZmin) / (fZmax - fZmin) * fNz);

    G4int index = ix + iy * fNx + iz * fNx * fNy;
    field[0] = fBx[index];
    field[1] = fBy[index];
    field[2] = fBz[index];
}
```

## Field Properties Configuration

### Setting Stepping Parameters

```cpp
// 配置场管理器的步进参数
G4FieldManager* fieldManager = new G4FieldManager(magField);

// 设置最小步长（默认 1 mm）
fieldManager->SetDeltaOneStep(0.01 * mm);

// 设置最小精度（默认 0.01）
fieldManager->SetDeltaIntersection(0.001 * mm);

// 设置步进器类型
#include "G4ClassicalRK4.hh"
#include "G4HelixSimpleRunge.hh"
#include "G4HelixImplicitEuler.hh"

// 对于磁场中的粒子，使用螺旋步进器
G4EquationOfMotion* equation = new G4Mag_EqRhs(magField);
G4MagIntegratorStepper* stepper = new G4HelixSimpleRunge(equation);

// 设置弦查找器
G4ChordFinder* chordFinder = new G4ChordFinder(magField, 0.01*mm, stepper);
fieldManager->SetChordFinder(chordFinder);
```

### Common Stepper Types

| Stepper | Description | Best For |
|---------|-------------|----------|
| G4ClassicalRK4 | Classical Runge-Kutta 4th order | General purpose |
| G4HelixSimpleRunge | Helix-based stepper | Pure magnetic field |
| G4HelixImplicitEuler | Implicit Euler for helices | High precision |
| G4CashKarpRKF45 | Adaptive step size | Varying fields |
| G4SimpleRunge | Simple Runge-Kutta | Electric fields |

## Equation of Motion

```cpp
// 不同类型的方程
#include "G4Mag_EqRhs.hh"           // 磁场
#include "G4EqMagElectricField.hh"  // 电场
#include "G4ElectroMagneticEqRhs.hh" // 电磁场

// 磁场方程
G4EquationOfMotion* magEquation = new G4Mag_EqRhs(magField);

// 电场方程
G4EquationOfMotion* eEquation = new G4EqMagElectricField(eField);

// 电磁场方程
G4EquationOfMotion* emEquation = new G4ElectroMagneticEqRhs(emField);
```

## Common Magnetic Field Configurations

### Solenoid Field (along Z)

```cpp
G4UniformMagField* solenoidField = new G4UniformMagField(
    G4ThreeVector(0., 0., 2.0 * tesla)
);
```

### Dipole Field (deflecting in X)

```cpp
G4UniformMagField* dipoleField = new G4UniformMagField(
    G4ThreeVector(0., 1.5 * tesla, 0.)
);
```

### Quadrupole Field (focusing)

```cpp
class QuadrupoleField : public G4MagneticField
{
public:
    QuadrupoleField(G4double gradient) : fGradient(gradient) {}

    void GetFieldValue(const G4double point[4], G4double* field) const override
    {
        G4double x = point[0], y = point[1];

        // 四极场：Bx = k*y, By = k*x
        field[0] = fGradient * y;  // Bx
        field[1] = fGradient * x;  // By
        field[2] = 0.;             // Bz
    }

private:
    G4double fGradient;  // 场梯度 (T/m)
};
```

## Disabling Field in Specific Volumes

```cpp
// 在某些体积中禁用场
G4FieldManager* nullFieldManager = new G4FieldManager();
logicShield->SetFieldManager(nullFieldManager, true);
```

## Common Issues

| Problem | Solution |
|---------|----------|
| Particle trajectory not curved | Check field strength and volume field manager |
| Particle spiraling infinitely | Increase minimum step size or use helix stepper |
| Simulation very slow | Reduce field precision or use simpler stepper |
| Field not active in volume | Ensure field manager is set to correct volume |

## Example: Complete Field Setup

```cpp
// 完整的场设置示例
G4VPhysicalVolume* DetectorConstruction::Construct()
{
    // ... 创建几何体 ...

    // 1. 创建磁场
    G4UniformMagField* magField = new G4UniformMagField(
        G4ThreeVector(0., 0., 1.0 * tesla)
    );

    // 2. 创建场管理器
    G4FieldManager* fieldManager = new G4FieldManager(magField);

    // 3. 配置步进器
    G4EquationOfMotion* equation = new G4Mag_EqRhs(magField);
    G4MagIntegratorStepper* stepper = new G4HelixSimpleRunge(equation);

    // 4. 配置弦查找器
    G4ChordFinder* chordFinder = new G4ChordFinder(magField, 0.01*mm, stepper);
    fieldManager->SetChordFinder(chordFinder);

    // 5. 设置精度
    fieldManager->SetDeltaOneStep(0.01 * mm);
    fieldManager->SetDeltaIntersection(0.001 * mm);

    // 6. 应用到场体积
    logicWorld->SetFieldManager(fieldManager, true);

    return physWorld;
}
```
