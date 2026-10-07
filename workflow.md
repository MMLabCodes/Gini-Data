# Simles to gini: End-to-End Workflow Guide

## ORCA Calculation from SMILES

The ORCA calculations are reproduced from the provided `.inp` files. These input files are retained in the repository because they contain the computational settings required to reproduce the quantum-chemical calculations, while the substantially larger ORCA output and cube files are generated locally and are not stored in the repository.

Starting from a SMILES string, the workflow is:

1. Convert the SMILES representation into an initial 3D molecular geometry.
2. Generate an ORCA `.inp` file containing the molecular geometry and required computational settings.
3. Perform a geometry optimisation in ORCA using **B3LYP/6-31+G\*\***.
4. Using the optimised geometry, calculate the electronic structure at the **B3LYP/6-31+G\*\*** level of theory.
5. Obtain the Kohn-Sham HOMO and LUMO.
6. Generate cube files containing:
   - the total electron density, $\rho(\mathbf{r})$,
   - the HOMO,
   - the LUMO.
7. Pass these three cube files to CRITIC2 for calculation of the condensed dual descriptor.

The repository therefore retains the ORCA input files required to reproduce the calculations, for example:

```text
orca_inputs/
├── ethanal.inp
├── acetone.inp
├── acetophenone.inp
├── benzaldehyde.inp
└── ...
```

## Condensed Dual Descriptor Workflow

Condensed dual descriptor ((f\^{(2)})) is the per-atom integral of the
dual Fukui function over CRITIC2 atomic basins. The used workflow was:

(Note: Use the ORCA provided cube files for the total electron density, HOMO, and LUMO.)

1.  CRITIC2 loads the density as the reference field, constructs the raw
    dual Fukui field as
    `f2(r) = rho_LUMO(r) - rho_HOMO(r) = LUMO(r)^2 - HOMO(r)^2`,
2.  Finds the density attractors/basins with `auto`,
3.  Marks the dual field as `integrable`, and writes the basin
    integrations to JSON.
4.  The condensed value for each atom is then taken directly from that
    atom's attractor integrals, specifically the third integral stored
    as `integrals[2]`, and saved as `dual`; no additional normalisation
    is applied in the code.

**Example CRITIC2 input for water:**

``` plaintext
molecule water_rho.cube 
load water_rho.cube  id rho
load water_lumo.cube  id lumo
load water_homo.cube  id homo
load as "($lumo)^2 - ($homo)^2" id dual

cube grid FILE water.dual.cube field dual
reference rho
auto
cpreport water.cp.json
integrable dual

yt JSON water.int.json
```

## Gini reactivity descriptor

Following the paper, the condensed dual-descriptor values are first separated into their positive and negative components, excluding zero-valued contributions:

$$
x_p = \{ f_k^{(2)} \mid f_k^{(2)} > 0 \}
$$

$$
x_n = \{ |f_k^{(2)}| \mid f_k^{(2)} < 0 \}
$$

The Gini coefficient is then calculated independently for each set. For a set \(x\) containing \(n\) atomic contributions with mean \(\bar{x}\), the uncorrected Gini coefficient is

$$
G(x) =
\frac{1}{2n^2\bar{x}}
\sum_{i=1}^{n}
\sum_{j=1}^{n}
|x_i - x_j|
$$

Because the number of contributing atomic basins can differ between molecules, the finite-sample correction is applied:

$$
G_{\mathrm{corr}}(x) =
\frac{n}{n-1}G(x)
$$

The final Gini reactivity descriptors are therefore calculated separately for the positive and negative dual-descriptor components:

$$
G^p = G_{\mathrm{corr}}(x_p)
$$

$$
G^n = G_{\mathrm{corr}}(x_n)
$$

Here, \(n\) is the number of non-zero atomic contributions in the relevant positive or negative set, not the total number of atoms in the molecule. A larger Gini value indicates that the corresponding dual-descriptor response is concentrated on a smaller subset of atomic basins, whereas a smaller value indicates a more evenly distributed response.
