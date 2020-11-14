+++
title = "OpenFOAM - Multiphase Heater Simulation"
date = "2020-11-14"
mathjax = true
jsxgraph = true
author = "Thomas Enzinger"
cover = "img/OpenFOAM-logo.png"
description = "Simulation of an water cooker with 3,000 W of power usage externalWallHeatFluxTemperature boundary condidition."
+++


# Introduction

Starting an project with an simple example will be some times lead into an
problem with more questions marks as bevor. A simulation of an simple water
cooker with OpenFOAM was an project for me with this behavior.

First of all I'm starting to setup an case with fluid conditions for an
multiphase simulation with three phases (liquid, vapor, air), volume fraction
method to modeling the phases and an solid region for the heater parts.<br />
The problem's begins to rise with current version of openfoam (v2006). I could
not find an solver that supports all conditions. Multiple region
solver leaks in multiphase supports as well as multiphase solvers leaks in ...


# The case

Bevor I get thinking about implementing an new solver or couple OpenFOAM with an
second solver for the solid parts, I decide to setup an simulation with
only one region (fluid) and an fixed uniform spaced heat flux on the solid
walls. As boundary condition I decide to use the
**externalWallHeatFluxTemperature** class and
the **icoReactingMultiphaseInterFoam** as solver.

The future will be show us how big the modeling error for the heat transfer
between solid and fluid region would be. With an constant heat flux on the walls
(without the respect of contacting phase) personaly I expect it will be leading
into some glitch phenomenons (overheating of vapor) near the walls in the fluid
region.

After successfull run I decide to rewrite the **CHT** module of the preCICIE
OpenFOAM adapter. Since it will supports all OpenFOAM solvers and it is possible
to couple an **icoReactingMultiphaseInterFoam** solver with an solver for the
solid heat transfer part. I decide to use the **laplacianFoam** solver.

All case files are stored on [my github repository][1].

## Geometry of the case

The geometry of the case is defined on figure 1 as below.

{{<figure src="schematic_multiphase_case.png" caption="Figure 1: Schematic representation">}}



## Simulation model

#### Fluid Part

- Equation of motion for fuild flow: **Reynolds-averaged Navier–Stokes equations**
- Turbulence model: [k-epsilon][2]
- Gas and liquid phases
- Thermophysical properties:
  - **Gas**
    - [Multiple component mixture][3]
    - Thermophysical model for fixed composition, based on density (heRhoThermo).
    - Dynamic viscosity and Prandtl numer are constant (transport model: const)
    - Assumes a constant \\(c_p\\) and a heat of fusion \\(H_f\\) (hconst).
    - Incompressible perfect gas
    - Phases: air and vapor
    - ... more, see [case files][1]
  - **Liquid**
    - [Pure mixture][3]
    - Thermophysical model for fixed composition, based on density (heRhoThermo).
    - Dynamic viscosity and Prandtl numer are constant (transport model: const)
    - Assumes a constant \\(c_p\\) and a heat of fusion \\(H_f\\) (hconst).
    - Constant density
    - ... more, see [case files][1]
- Surface tension modeling by tabled values. Based on [IAPWS R1-76(2014), The International Association for the Properties of Water and Steam, Moscow, Russia, June 2014][4]
- Mass transfer model \\(liquid \rightarrow gas\\):
  - Model: [kineticGasEvaporation][5]
  - Accommodation coefficient \\(C\\): 0.001
  - Saturation temperature \\(T_{activate}\\): 372 K
- Apply of gravity: -9.81 m/s²


#### Solid Part

- Simplify the temperature field to an laplacian equation
- Material: Steel, 1.4301, **X5CrNi18-10**
  - Density: 7,900 kg/m³
  - Thermal conductivity: 16.2 \\(\frac{W}{m \cdot K}\\)
  - Diffusion coefficient \\(D_T\\): \\(4.1 \cdot 10^{-6}\\) m²/s
  - Specific heat capacity (constant pressure) \\(c_p\\): 480 \\(\frac{J}{kg \cdot K}\\)


### Boundary conditions

#### Opening

- Type: InletOutlet (Opening)
- Pressure: 100.000 Pa
- Inflow
  - Temperature: 295 K
  - Phases: Air

#### Heaters

- Initial Temperature: 295 K
- Material:
  - Steel, X5CrNi18-10
  - Density: 7,900 kg/m³
  - Specific heat capacity: \\(c_p = 480\\) \\(\frac{J}{kg \cdot K}\\)
- Heat source with a power of 3,000 W:
  - All source values are linear ramped on the first ten seconds.
  - Wall heat flux for **externalWallHeatFluxTemperature**: 105,263 W/m²
  - Volumetric heat source:
    - Reference volume for heating power: \\(60.6 \cdot 10^{-6}\\) m³
    - Volume specific heat power: \\(49.54 \cdot 10^{6}\\) W/m³

#### Walls

- Adiabatic
- Usage of wall functions


# The meshs

See to the [case files][1] under CAD directory for more information. Both case
files are using the same mesh.

Important notes on the wall fluid \\(\longleftrightarrow\\) solid:

  - Radius discretisation with local length: 0.15 mm
  - Total layer thickness: 2 mm
  - Number of layers: 8
  - Ratio factor: 1.15


# The results

Summary, comparison of case 1 (externalWallHeatFluxTemperature) and case 2 (
heater region modeled with laplacianFoam):

  - Sooner and much higher vapor generation in case 1
  - Average water temperature rise near equal, little higher in case 2. That
    properly depends on the less power losses due to vapor generation.
  - Very diverse temperature and velocity field in water region.

Video of both cases with temperature field and liquid volume fraction view.

{{< video muted="true" autoplay="false" loop="false" preload="auto" src="external_vs_heater_region" >}}


[1]: https://github.com/TEFEdotCC/OpenFOAM-Water-Cooker-Example
[2]: https://www.openfoam.com/documentation/guides/latest/doc/guide-turbulence-ras-k-epsilon.html
[3]: https://cfd.direct/openfoam/user-guide/v6-thermophysical/
[4]: http://www.iapws.org/relguide/Surf-H2O-2014.pdf
[5]: https://www.openfoam.com/documentation/guides/latest/api/classFoam_1_1meltingEvaporationModels_1_1kineticGasEvaporation.html#details
