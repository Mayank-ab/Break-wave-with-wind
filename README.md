# Wind Effects on Wave Overtopping using OpenFOAM/olaFlow

This case setup is based on the study of wind effects on wave overtopping at a vertical sea defence, inspired by De Chowdhury et al. (2023). The goal is to simulate wave overtopping in OpenFOAM/olaFlow and approximate wind action by adding a momentum source in a selected air/wind region.

## 1. Objective

The main objective is to study how onshore wind changes wave overtopping at a vertical wall or caisson.

The simulation approach is:

- generate waves using `waveDict`
- initialize the water phase using `setFields`
- select a wind/forcing region using `topoSet`
- apply a momentum source using `fvOptions`
- run the olaFlow solver in parallel
- reconstruct and visualize the results in ParaView

The physical idea is that wind can change the shape and travel distance of the overshooting water jet. Depending on the wave and jet shape, wind can increase overtopping, reduce overtopping, or have only a small effect.

## 2. Important physical background

The referenced study observed three types of wind response:

| Type | Wind effect | Meaning |
|---|---|---|
| Type A | Small wind effect | The wave impact already creates strong air motion, so added wind changes the jet only slightly. |
| Type B | Negative wind effect | Wind breaks the jet into smaller fragments, so more water falls back and overtopping reduces. |
| Type C | Strong positive wind effect | Wind pushes the jet in a way that increases overtopping, sometimes strongly near the wall. |

The key lesson is that wind effect depends not only on wind speed, but also on wave steepness, freeboard, jet thickness, and jet instability.

## 3. Simulation workflow

Recommended workflow:



```bash
chmod +x runCase
./runCase
```
For a clean run, remove old time folders and processor folders before restarting.
```bash
chmod +x cleanCase
./cleanCase
```

### From here on these are already provided in the case I am just going through them one by one


## 4. `topoSetDict`: creating the wind zone

Create `system/topoSetDict`.

Example:

```cpp
/*--------------------------------*- C++ -*----------------------------------*\
| =========                 |                                                 |
| \\      /  Field          | OpenFOAM                                        |
\*---------------------------------------------------------------------------*/

FoamFile
{
    version     2.0;
    format      ascii;
    class       dictionary;
    object      topoSetDict;
}

actions
(
    {
        name    windCells;
        type    cellSet;
        action  new;

        source  boxToCell;

        sourceInfo
        {
            // Wind/forcing zone
            // box (xmin ymin zmin) (xmax ymax zmax)
            box (3 -1 0.45) (5 1 0.9);
        }
    }

    {
        name    windZone;
        type    cellZoneSet;
        action  new;

        source  setToCellZone;

        sourceInfo
        {
            set windCells;
        }
    }
);
```

### Meaning of key entries

| Entry | Meaning |
|---|---|
| `windCells` | Temporary cell set selected by the box. |
| `boxToCell` | Selects all mesh cells inside the given box. |
| `box (3 -1 0.45) (5 1 0.9)` | Defines the wind forcing region. Change this according to your geometry. |
| `windZone` | Final cell zone used by `fvOptions`. |
| `setToCellZone` | Converts the selected cell set into a cell zone. |

Important: the `cellZone` name used in `fvOptions` must match the zone created here. If `topoSetDict` creates `windZone`, then `fvOptions` must also use `cellZone windZone;`.

## 5. `fvOptions`: applying the wind/momentum source

Create `constant/fvOptions`.

### Option A: `meanVelocityForce`

This tries to maintain a target mean velocity inside the selected cell zone.

```cpp
/*--------------------------------*- C++ -*----------------------------------*\
| =========                 |                                                 |
\*---------------------------------------------------------------------------*/

FoamFile
{
    version     2.0;
    format      ascii;
    class       dictionary;
    object      fvOptions;
}

momentumSource
{
    type            meanVelocityForce;
    active          yes;

    selectionMode   cellZone;
    cellZone        windZone;

    fields          (U);

    Ubar            (2 0 0);
    relaxation      0.2;

    timeStart       0;
    duration        10000;
}
```

### Meaning of key entries

| Entry | Meaning |
|---|---|
| `type meanVelocityForce` | Applies a force to make the selected region approach a target mean velocity. |
| `active yes` | Turns the source on. Use `active no` to disable it. |
| `selectionMode cellZone` | Applies the source only inside a selected cell zone. |
| `cellZone windZone` | The source is applied inside the cell zone named `windZone`. |
| `fields (U)` | Applies the source to velocity field `U`. |
| `Ubar (2 0 0)` | Target mean velocity is 2 m/s in the x-direction. |
| `relaxation 0.2` | Controls how strongly/smoothly the correction is applied. |
| `timeStart 0` | Source starts at simulation time 0 s. |
| `duration 10000` | Source remains active for 10000 s after `timeStart`. |

Warning: `meanVelocityForce` can become unstable if it is used near very small, distorted, or highly refined cells. It acts like a velocity controller and may apply a very large correction if the target velocity is difficult to reach.

### Option B: `semiImplicitSource` for OpenFOAM 10

For a gentler fan/gust-style source in OpenFOAM 10, use `semiImplicitSource`.

```cpp
/*--------------------------------*- C++ -*----------------------------------*\
| =========                 |                                                 |
\*---------------------------------------------------------------------------*/

FoamFile
{
    version     2.0;
    format      ascii;
    class       dictionary;
    object      fvOptions;
}

windSource
{
    type            semiImplicitSource;
    active          yes;

    selectionMode   cellZone;
    cellZone        windZone;

    volumeMode      specific;

    sources
    {
        U
        {
            explicit    (0.0001 0 0);
            implicit    0;
        }
    }

    timeStart       1;
    duration        0.05;
}
```

Mathematically, this adds a source term of the form:

```text
S(U) = Su + Sp * U
```

For the example above:

```text
Su = (0.0001, 0, 0)
Sp = 0
```

So it adds a small constant push in the positive x-direction inside the selected wind zone. It does not try to maintain an exact target velocity.

## 6. Mesh warning

A major issue observed during testing was instability when the forcing zone overlapped with small triangular or highly refined cells created by `snappyHexMesh`.

Avoid placing the `topoSetDict` forcing box directly over:

- triangular/snappy transition cells
- very small cells
- highly skewed cells
- caisson corners
- wave impact/splash regions
- mixed air-water interface cells

A safer approach is to place the wind zone in a clean upper-air region with mostly uniform cells.

Example safer box:

```cpp
box (2.0 -1 0.65) (3.0 1 0.9);
```

If using a vertical wall/caisson, a cleaner alternative is to build the geometry directly using multiple `blockMesh` blocks instead of using `snappyHexMesh`. This avoids distorted triangular cells and gives better numerical stability.

## 7. Debugging common errors

### Error: cannot find cellZone

Example:

```text
Cannot find cellZone momentumZone
Valid cellZones are 1(windZone)
```

Reason: `fvOptions` is asking for `momentumZone`, but `topoSetDict` created `windZone`.

Fix:

```cpp
cellZone windZone;
```

or rename the zone in `topoSetDict` to match `fvOptions`.

### Error: cannot find patchField entry

Example:

```text
Cannot find patchField entry for caisson
```

Reason: the mesh has a patch called `caisson`, but the files in `0.org` do not contain boundary conditions for this patch.

Fix: add `caisson` to all relevant files in `0.org`, such as:

- `0.org/U`
- `0.org/p_rgh`
- `0.org/alpha.water`

Example for `U`:

```cpp
caisson
{
    type noSlip;
}
```

Example for `alpha.water`:

```cpp
caisson
{
    type zeroGradient;
}
```

Example for `p_rgh`:

```cpp
caisson
{
    type fixedFluxPressure;
    value uniform 0;
}
```

### Error: simulation says complete after fatal error

Add this near the top of the run script:

```bash
set -e
```

This stops the script immediately if any command fails.

## 8. Recommended run script

```bash
#!/bin/bash
set -e

echo "========================================"
echo "Starting olaFlow simulation"
echo "========================================"

echo "Cleaning old files..."
rm -rf processor*
rm -rf constant/polyMesh
rm -rf [1-9]* 0.*
find . -name '*:Zone.Identifier' -delete
find . -name 'Zone.Identifier' -delete

echo "Running blockMesh..."
blockMesh > blockMesh.log

echo "Preparing 0 folder..."
rm -rf 0
cp -r 0.org 0

echo "Running setFields..."
setFields > setFields.log

echo "Running topoSet..."
topoSet > topoSet.log

echo "Running decomposePar..."
decomposePar > decomposePar.log

echo "Running olaFlow in parallel..."
mpirun -np 8 olaFlow -parallel > olaFlow.log

echo "Reconstructing case..."
reconstructPar > reconstructPar.log

echo "========================================"
echo "Simulation complete."
echo "========================================"
```

## 9. Monitoring the run(Optional)

Use:

```bash
tail -f olaFlow.log
```

Watch these values:

```text
Time =
deltaT =
Courant Number max
Interface Courant Number max
```

Stop the run if:

```text
deltaT becomes extremely small, such as 1e-20 or lower
Time stops increasing
pressure gradient becomes huge
Ubar becomes extremely large
Floating point exception occurs
```

A stable run should have:

- time increasing normally
- bounded `alpha.water` between 0 and 1
- Courant number below the chosen limit
- no exploding pressure-gradient or velocity values

## 10. Practical recommendation

For early testing:

1. Run without wind first.
2. Keep adjusting the wave height.
3. Confirm the wave case is stable.
4. Add the wind source only in a clean upper-air region.
5. Start with a very small source value.
6. Increase gradually.
7. Avoid placing the source near snappyHexMesh transition cells or caisson corners.

For the most physical wind setup, split the inlet into two patches:

- lower patch for wave generation
- upper patch for air wind inlet

Then apply fixed air velocity at the upper inlet. This is more physical than an internal `fvOptions` forcing region, but requires changing the mesh boundary patches.
