# Geometry Reference

## Table of Contents

- [Coordinate System](#coordinate-system)
- [Solids (Shapes)](#solids-shapes)
- [Logical Volume](#logical-volume)
- [Physical Volume (Placement)](#physical-volume-placement)
- [Rotation](#rotation)
- [Materials](#materials)
- [Volume Hierarchy](#volume-hierarchy)
- [Checking Overlaps](#checking-overlaps)
- [Common Patterns](#common-patterns)

Geant4 uses a hierarchical geometry where volumes are placed inside other volumes.
The top-level volume is always the "World" volume.

## Coordinate System

Geant4 uses a right-handed coordinate system:
- Units: mm (default), cm, m, km
- Angles: radian (default), degree
- Always specify units: `10 * cm`, `45 * degree`

## Solids (Shapes)

### G4Box — Box/Rectangle
```cpp
G4Box("name", halfX, halfY, halfZ)
// Example: 20cm x 50cm x 50cm box
new G4Box("Target", 10*cm, 25*cm, 25*cm)
```

### G4Tubs — Cylinder/Tube
```cpp
G4Tubs("name", rMin, rMax, halfZ, startAngle, spanAngle)
// Example: solid cylinder, radius 5cm, height 10cm
new G4Tubs("Cylinder", 0, 5*cm, 5*cm, 0, 360*degree)
// Example: hollow tube (pipe)
new G4Tubs("Pipe", 2*cm, 5*cm, 10*cm, 0, 360*degree)
```

### G4Sphere — Sphere
```cpp
G4Sphere("name", rMin, rMax, startPhi, spanPhi, startTheta, spanTheta)
// Example: solid sphere, radius 10cm
new G4Sphere("Sphere", 0, 10*cm, 0, 360*degree, 0, 180*degree)
// Example: hollow sphere (shell)
new G4Sphere("Shell", 8*cm, 10*cm, 0, 360*degree, 0, 180*degree)
```

### G4Cons — Cone
```cpp
G4Cons("name", rMin1, rMax1, rMin2, rMax2, halfZ, startAngle, spanAngle)
// Example: cone from radius 0-5cm (bottom) to 0-10cm (top), height 20cm
new G4Cons("Cone", 0, 5*cm, 0, 10*cm, 10*cm, 0, 360*degree)
```

### G4Torus — Torus (Donut)
```cpp
G4Torus("name", rMin, rMax, rtor, startPhi, spanAngle)
// rMin/rMax: tube radius, rtor: torus radius
new G4Torus("Torus", 0, 2*cm, 10*cm, 0, 360*degree)
```

### G4Polyhedra — Polygon Prism
```cpp
G4Polyhedra("name", startPhi, spanPhi, numSides, numZPlanes,
            zPlane[], rInner[], rOuter[])
```

### G4EllipticalTube — Elliptical Cylinder
```cpp
G4EllipticalTube("name", dx, dy, dz)
```

## Logical Volume

A logical volume connects a solid to material and sensitivity:
```cpp
G4LogicalVolume* logic = new G4LogicalVolume(
    solid,      // the shape
    material,   // material
    "name"      // name
);
```

## Physical Volume (Placement)

Place a logical volume inside another:
```cpp
new G4PVPlacement(
    rotation,       // G4RotationMatrix* (nullptr = no rotation)
    position,       // G4ThreeVector
    logicVolume,    // the volume to place
    "name",         // name
    motherVolume,   // parent logical volume
    false,          // boolean operation
    copyNumber,     // unique ID
    true            // check overlaps
);
```

## Rotation

```cpp
// No rotation
G4RotationMatrix* rot = nullptr;

// Rotation around Z axis by 45 degrees
G4RotationMatrix* rot = new G4RotationMatrix();
rot->rotateZ(45 * degree);

// Multiple rotations (order matters!)
rot->rotateX(30 * degree);
rot->rotateY(45 * degree);
```

## Materials

### Using NIST Database (Recommended)
```cpp
G4NistManager* nist = G4NistManager::Instance();

// Elements
nist->FindOrBuildMaterial("G4_H");
nist->FindOrBuildMaterial("G4_He");
nist->FindOrBuildMaterial("G4_Li");
nist->FindOrBuildMaterial("G4_Be");
nist->FindOrBuildMaterial("G4_C");
nist->FindOrBuildMaterial("G4_N");
nist->FindOrBuildMaterial("G4_O");
nist->FindOrBuildMaterial("G4_Al");
nist->FindOrBuildMaterial("G4_Si");
nist->FindOrBuildMaterial("G4_Fe");
nist->FindOrBuildMaterial("G4_Cu");
nist->FindOrBuildMaterial("G4_Ge");
nist->FindOrBuildMaterial("G4_Ag");
nist->FindOrBuildMaterial("G4_Au");
nist->FindOrBuildMaterial("G4_Pb");

// Compounds
nist->FindOrBuildMaterial("G4_AIR");
nist->FindOrBuildMaterial("G4_WATER");
nist->FindOrBuildMaterial("G4_CONCRETE");
nist->FindOrBuildMaterial("G4_POLYETHYLENE");
nist->FindOrBuildMaterial("G4_PLASTIC_VINYLTOLUENE");
```

### Custom Material
```cpp
// Define elements
G4double z, a;
G4Element* elH = new G4Element("Hydrogen", "H", z=1., a=1.008*g/mole);
G4Element* elO = new G4Element("Oxygen", "O", z=8., a=16.00*g/mole);

// Define material
G4double density = 1.0*g/cm3;
G4Material* water = new G4Material("Water", density, 2);
water->AddElement(elH, 2);
water->AddElement(elO, 1);
```

### Custom Alloy
```cpp
// Stainless steel example
G4Element* elFe = new G4Element("Iron", "Fe", 26., 55.85*g/mole);
G4Element* elCr = new G4Element("Chromium", "Cr", 24., 52.00*g/mole);
G4Element* elNi = new G4Element("Nickel", "Ni", 28., 58.69*g/mole);

G4double density = 8.0*g/cm3;
G4Material* steel = new G4Material("Steel", density, 3);
steel->AddElement(elFe, 0.70);  // 70% iron
steel->AddElement(elCr, 0.18);  // 18% chromium
steel->AddElement(elNi, 0.12);  // 12% nickel
```

## Volume Hierarchy

```
World (G4Box, AIR)
├── Target (G4Box, Aluminum)
├── Detector (G4Tubs, Silicon)
└── Shield (G4Box, Lead)
    └── Window (G4Box, Air)
```

## Checking Overlaps

Always enable overlap checking during development:
```cpp
new G4PVPlacement(..., true);  // last parameter = checkOverlaps
```

If Geant4 reports overlaps, adjust positions or sizes until no warnings appear.

## Common Patterns

### Layered Target
```cpp
// Multiple thin layers
for (G4int i = 0; i < nLayers; i++) {
    G4double xPos = i * layerSpacing;
    new G4PVPlacement(0, G4ThreeVector(xPos, 0, 0),
                      logicLayer, "Layer", logicWorld, false, i, true);
}
```

### Concentric Cylinders
```cpp
// Inner cylinder
new G4Tubs("Inner", 0, r1, halfZ, 0, 360*deg);
// Outer shell
new G4Tubs("Outer", r1, r2, halfZ, 0, 360*deg);
```
