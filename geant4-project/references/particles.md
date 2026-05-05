# Particle Source Reference

Geant4 provides two main ways to define particle sources:
1. **G4ParticleGun** — simple single-particle source (recommended for beginners)
2. **G4GeneralParticleSource (GPS)** — complex source with distributions

## G4ParticleGun

### Basic Setup
```cpp
#include "G4ParticleGun.hh"
#include "G4ParticleTable.hh"
#include "G4SystemOfUnits.hh"

// In constructor
fParticleGun = new G4ParticleGun(1);  // 1 particle per event

// Set particle type
G4ParticleTable* table = G4ParticleTable::GetParticleTable();
fParticleGun->SetParticleDefinition(table->FindParticle("neutron"));

// Set properties
fParticleGun->SetParticleEnergy(10 * MeV);
fParticleGun->SetParticlePosition(G4ThreeVector(-20*cm, 0, 0));
fParticleGun->SetParticleMomentumDirection(G4ThreeVector(1, 0, 0));
```

### Common Particle Names

| Particle | Name string | Notes |
|----------|-------------|-------|
| Neutron | "neutron" | |
| Photon (gamma) | "gamma" | |
| Electron | "e-" | |
| Positron | "e+" | |
| Proton | "proton" | |
| Alpha | "alpha" | He-4 nucleus |
| Deuteron | "deuteron" | |
| Triton | "triton" | |
| Muon- | "mu-" | |
| Muon+ | "mu+" | |
| Pion+ | "pi+" | |
| Pion- | "pi-" | |
| Pion0 | "pi0" | |
| Kaon+ | "kaon+" | |
| Kaon- | "kaon-" | |

### Multiple Particles Per Event

```cpp
// Fire 3 particles per event
fParticleGun = new G4ParticleGun(3);
```

Note: All particles in one event will have the same properties unless you modify them
in GeneratePrimaries().

### Random Energy

```cpp
#include "Randomize.hh"

void MyPrimaryGeneratorAction::GeneratePrimaries(G4Event* event)
{
    // Random energy between 1 and 10 MeV
    G4double energy = 1*MeV + G4UniformRand() * 9*MeV;
    fParticleGun->SetParticleEnergy(energy);
    fParticleGun->GeneratePrimaryVertex(event);
}
```

### Random Direction (Isotropic)

```cpp
void MyPrimaryGeneratorAction::GeneratePrimaries(G4Event* event)
{
    // Random direction on a sphere
    G4double cosTheta = 2.*G4UniformRand() - 1.;
    G4double phi = 2.*CLHEP::pi*G4UniformRand();
    G4double sinTheta = std::sqrt(1. - cosTheta*cosTheta);

    G4double ux = sinTheta*std::cos(phi);
    G4double uy = sinTheta*std::sin(phi);
    G4double uz = cosTheta;

    fParticleGun->SetParticleMomentumDirection(G4ThreeVector(ux, uy, uz));
    fParticleGun->GeneratePrimaryVertex(event);
}
```

### Random Position (Point Source with Spread)

```cpp
void MyPrimaryGeneratorAction::GeneratePrimaries(G4Event* event)
{
    // Random position in a circle
    G4double r = 1*cm * std::sqrt(G4UniformRand());
    G4double theta = 2.*CLHEP::pi*G4UniformRand();
    G4double x = r*std::cos(theta);
    G4double y = r*std::sin(theta);

    fParticleGun->SetParticlePosition(G4ThreeVector(x, y, -20*cm));
    fParticleGun->GeneratePrimaryVertex(event);
}
```

### Beam with Divergence

```cpp
void MyPrimaryGeneratorAction::GeneratePrimaries(G4Event* event)
{
    // Beam along z with small angular spread
    G4double sigmaAngle = 2.*degree;
    G4double ux = G4RandGauss::shoot(0, sigmaAngle);
    G4double uy = G4RandGauss::shoot(0, sigmaAngle);

    fParticleGun->SetParticleMomentumDirection(
        G4ThreeVector(ux, uy, 1.).unit()
    );
    fParticleGun->GeneratePrimaryVertex(event);
}
```

## G4GeneralParticleSource (GPS)

GPS is more powerful but more complex. It supports:
- Multiple source shapes (point, box, sphere, cylinder, surface)
- Angular distributions
- Energy distributions
- Time distributions
- Biasing

### Basic GPS Setup (in main or macro)

```cpp
#include "G4GeneralParticleSource.hh"

// In your primary generator action constructor
fGPS = new G4GeneralParticleSource();
```

### GPS via Macro File

```
# gps.mac — 粒子源配置
/gps/particle neutron
/gps/pos/type Plane
/gps/pos/shape Circle
/gps/pos/centre 0 0 -20 cm
/gps/pos/radius 1 cm
/gps/ang/type iso
/gps/ene/type Mono
/gps/ene/mono 10 MeV
```

### GPS Source Shapes

| Type | Command | Parameters |
|------|---------|------------|
| Point | `/gps/pos/type Point` | `/gps/pos/centre` |
| Circle | `/gps/pos/type Plane` + `/gps/pos/shape Circle` | centre, radius |
| Square | `/gps/pos/type Plane` + `/gps/pos/shape Square` | centre, halfX, halfY |
| Sphere | `/gps/pos/type Volume` + `/gps/pos/shape Sphere` | centre, radius |
| Cylinder | `/gps/pos/type Volume` + `/gps/pos/shape Cylinder` | centre, radius, halfZ |

### GPS Angular Distributions

| Type | Command | Description |
|------|---------|-------------|
| Isotropic | `/gps/ang/type iso` | Uniform in 4pi |
| Direction | `/gps/ang/type planar` | Fixed direction |
| Cosine | `/gps/ang/type cos` | Cosine distribution |

### GPS Energy Distributions

| Type | Command | Parameters |
|------|---------|------------|
| Mono | `/gps/ene/type Mono` | `/gps/ene/mono` |
| Linear | `/gps/ene/type Lin` | min, max |
| Power law | `/gps/ene/type Pow` | min, max, alpha |
| Exponential | `/gps/ene/type Exp` | min, max, alpha |
| Gaussian | `/gps/ene/type Gauss` | mean, sigma |
| Blackbody | `/gps/ene/type BBody` | temperature |
| Cosmic ray | `/gps/ene/type Cdg` | Emin, Emax |

## When to Use Which

- **G4ParticleGun**: Simple sources, single particle type, basic distributions
  - Best for beginners
  - Good for testing
  - Easy to understand and modify

- **G4GeneralParticleSource**: Complex sources, need macro control, multiple distributions
  - When you need spatial distributions
  - When you want to change source without recompiling
  - When simulating realistic sources

## Tips

1. Always check units — forgetting `* MeV` or `* cm` is a common mistake
2. For radioactive sources, consider using GPS with the `/gps/ion` command
3. For cosmic ray simulations, GPS has built-in cosmic ray spectra
4. Test your source with a simple geometry first before building complex detectors
