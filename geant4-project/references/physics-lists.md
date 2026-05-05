# Physics Lists Reference

Physics lists determine which particles and interactions are simulated in Geant4.
Choosing the right physics list is critical for accurate results.

## How to Choose

Ask these questions:
1. What particles are involved?
2. What energy range?
3. Do you need high precision for specific processes?

## Built-in Reference Physics Lists

### High Energy Physics (HEP)

| Physics List | Description | Best For |
|--------------|-------------|----------|
| QGSP_BERT | Quark Gluon String Precompound + Bertini cascade | General HEP, >1 GeV |
| QGSP_BERT_HP | QGSP_BERT + High Precision neutron | Neutron transport, reactor physics |
| QGSP_BIC | QGSP + Binary Cascade | Better for <10 GeV hadronic |
| QGSP_BIC_HP | QGSP_BIC + High Precision neutron | Low energy neutrons with ions |
| FTFP_BERT | Fritiof + Bertini | Alternative to QGSP, good for muons |
| FTFP_BERT_HP | FTFP_BERT + HP neutron | Neutron applications with FTFP |

### Electromagnetic Physics

| Physics List | Description | Best For |
|--------------|-------------|----------|
| EmStandard | Standard EM | General purpose |
| EmLivermore | Low energy EM (Livermore data) | <1 MeV gamma/electron |
| EmPenelope | Low energy EM (Penelope data) | Medical physics, <1 MeV |
| EmStandard_opt4 | Optimized for LHC | High energy colliders |

### Specialized

| Physics List | Description | Best For |
|--------------|-------------|----------|
| Shielding | Shielding + HP neutron | Radiation shielding |
| ShieldingLEND | Shielding + LEND neutron | Thermal neutron shielding |
| NuBeam | Neutrino beam | Neutrino experiments |
| RadioactiveDecay | Radioactive decay | Nuclear decays |

## Energy Ranges

- **Very low energy (<1 keV)**: Use EmLivermore or EmPenelope
- **Low energy (1 keV - 100 MeV)**: Use QGSP_BIC or FTFP_BERT
- **High energy (>100 MeV)**: Use QGSP_BERT or FTFP_BERT
- **Ultra high energy (>10 TeV)**: Use QGSP_BERT with appropriate hadronic model

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

1. **Using EmStandard for low energy**: If simulating <1 MeV particles, EmLivermore or EmPenelope
   will give more accurate results.
2. **Forgetting HP for neutrons**: Without "_HP", neutron interactions below 20 MeV use
   parameterized models which are much less accurate.
3. **Mixing physics constructors**: Don't mix electromagnetic physics from different lists.
   Use one complete list or build a custom one properly.
