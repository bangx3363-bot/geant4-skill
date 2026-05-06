# Physics Lists Reference

## Table of Contents

- [How to Choose](#how-to-choose)
- [Built-in Reference Physics Lists](#built-in-reference-physics-lists)
- [Energy Ranges](#energy-ranges)
- [Adding HP (High Precision) for Neutrons](#adding-hp-high-precision-for-neutrons)
- [Custom Physics Lists](#custom-physics-lists)
- [Common Mistakes](#common-mistakes)

Physics lists determine which particles and interactions are simulated in Geant4.
Choosing the right physics list is critical for accurate results.

## How to Choose

Ask these questions:
1. What particles are involved?
2. What energy range?
3. Do you need high precision for specific processes?

## Built-in Reference Physics Lists

**Geant4 11.4+ recommends `FTFP_BERT` as the default general-purpose physics list.**
`QGSP_BERT_HP` is still under validation and not recommended for production physics studies.

### High Energy Physics (HEP)

| Physics List | Description | Best For |
|--------------|-------------|----------|
| FTFP_BERT | FTF (Fritiof) + Bertini cascade | **Default** for general purpose |
| FTFP_BERT_HP | FTFP_BERT + High Precision neutron | Neutron transport, reactor physics |
| FTFP_INCLXX | FTF + INCL++ (Liege model) | Spallation, ADS |
| FTFP_INCLXX_HP | FTFP_INCLXX + High Precision neutron | Spallation with neutrons |
| FTF_BIC | FTF + Binary cascade | Alternative to FTFP_BERT |
| QGSP_BERT | QGS (Quark-Gluon String) + Bertini | Legacy general HEP, >1 GeV |
| QGSP_BERT_HP | QGSP_BERT + High Precision neutron | Neutron transport (legacy) |
| QGSP_BIC | QGS + Binary cascade | Better for <10 GeV hadronic |
| QGSP_BIC_HP | QGSP_BIC + High Precision neutron | Low energy neutrons with ions |
| QBBC | Binary cascade based | Medical physics, mixed particles |

### Electromagnetic Physics Constructors

These are **not standalone physics lists** — they are EM constructors to register in a custom physics list.
All reference hadronic physics lists (FTFP_BERT, QGSP_BERT, etc.) already include standard EM physics.

| Constructor Header | Description | Best For |
|-------------------|-------------|----------|
| G4EmStandardPhysics | Standard EM | General purpose (default in FTFP_BERT) |
| G4EmLivermorePhysics | Low energy EM (Livermore data) | <1 MeV gamma/electron |
| G4EmPenelopePhysics | Low energy EM (Penelope data) | Medical physics, <1 MeV |
| G4EmStandardPhysics_option4 | Optimized for LHC | High energy colliders |

Example: replacing standard EM with Livermore in a custom physics list:
```cpp
#include "G4VModularPhysicsList.hh"
#include "G4EmLivermorePhysics.hh"
#include "G4HadronPhysicsFTFP_BERT.hh"

class MyPhysicsList : public G4VModularPhysicsList {
public:
    MyPhysicsList() {
        RegisterPhysics(new G4EmLivermorePhysics);
        RegisterPhysics(new G4HadronPhysicsFTFP_BERT);
    }
};
```

### Specialized

| Physics List | Description | Best For |
|--------------|-------------|----------|
| Shielding | Shielding + HP neutron | Radiation shielding |
| ShieldingLEND | Shielding + LEND neutron | Thermal neutron shielding |
| NuBeam | Neutrino beam | Neutrino experiments |

## Energy Ranges

- **Very low energy (<1 keV)**: Use G4EmLivermorePhysics or G4EmPenelopePhysics
- **Low energy (1 keV - 100 MeV)**: Use FTFP_BERT or QGSP_BIC
- **High energy (>100 MeV)**: Use FTFP_BERT (default) or QGSP_BERT
- **Ultra high energy (>10 TeV)**: Use FTFP_BERT with appropriate hadronic model

## Adding HP (High Precision) for Neutrons

The "_HP" suffix adds high-precision neutron cross sections from the JEFF-3.3 evaluated nuclear
data library. This is essential for:
- Reactor simulations
- Shielding calculations
- Any simulation where neutron interactions below 20 MeV matter

The HP models are slower but much more accurate for low-energy neutrons.

## Custom Physics Lists

For advanced users, you can create custom physics lists by:
1. Inheriting from `G4VModularPhysicsList`
2. Registering specific physics constructors
3. Setting production cuts

This is rarely needed for beginners — suggest built-in lists first.

## Common Mistakes

1. **Using standard EM for low energy**: If simulating <1 MeV particles, G4EmLivermorePhysics or
   G4EmPenelopePhysics will give more accurate results.
2. **Forgetting HP for neutrons**: Without "_HP", neutron interactions below 20 MeV use
   parameterized models which are much less accurate.
3. **Mixing physics constructors**: Don't mix electromagnetic physics from different lists.
   Use one complete list or build a custom one properly.
