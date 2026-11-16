# STEP model

STEP ($s_T$ with enhanced propagation) model is an extended G-equation model with a modified formulation of the turbulent flame speed to consider the effects of thermodiffusive instabilities in hydrogen flames and their interactions with turbulence. The model is developed in the commercial software CONVERGE using user-defined functions (UDF). The compiled library of the UDF, containing the STEP model, is provided in this repository to facilitate the use of this model. 


## Citations

Please cite the following publication when using this model

H. Chu, V. Varatharajan, B. Pehlivanlar, T. Lehmann, L. Berger, D. Golc, S. Esposito, T. L. Howarth, E. Porcelli, D. Laera, M. Günther, M. Gauding, J. Beeckmann, S. Pischinger, and H. Pitsch. An extended G-equation formulation for simulating thermodiffusively unstable hydrogen flames. Proceedings of the Combustion Institute, 2025. https://doi.org/10.1016/j.proci.2025.105945

D. Golc, H. Chu, B. Brand, S. Esposito, T. Lehmann, T. L. Howarth, J. Beeckmann, and H. Pitsch. Characterization of thermodiffusive instabilities across various hydrogen engine conditions based on numerical and experimental investigations. Proceedings of the Combustion Institute, 2026. https://doi.org/10.1016/j.proci.2026.106867

## Files

This repository contains two files:
1. `libconverge_udf.so` is the compiled CONVERGE UDF library.
2. `tlf_table.h5` is the table containing the parameter $\omega_2$ and the required laminar-flame properties.

Experimental engine data accompanying the 2026 paper are available on [Zenodo](https://doi.org/10.5281/zenodo.23660662). Use these measurements as reference data when testing the STEP model and comparing simulation results with the engine experiments described in the paper.


# STEP model: installation and case setup


**This UDF has been checked with CONVERGE 5.0.2.**

The UDF is supplied as the ready-to-use shared library `libconverge_udf.so`. The simulation uses an HDF5 laminar-flame table containing the instability parameter **ω₂**. **The previous `flame_data.csv` file is no longer required.**

## 1. Install the supplied library

Create a directory for the UDF library and copy `libconverge_udf.so` into it:

```bash
mkdir -p /path/to/STEP-udf
cp /path/to/download/libconverge_udf.so /path/to/STEP-udf/
```

The directory can be located alongside the CONVERGE case or elsewhere on a filesystem accessible to the solver.

Add the following line to your shell environment or job-submission script **before starting CONVERGE**:

```bash
export LD_LIBRARY_PATH="/path/to/STEP-udf${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}"
```

Replace `/path/to/STEP-udf` with the actual absolute directory containing the library. Do not add a space after `LD_LIBRARY_PATH=`. For batch or MPI runs, ensure that the solver processes inherit this environment and can access the library.

## 2. Enable UDF usage in the case

In the existing `feature_control` section of `inputs.in`, set:

```yaml
feature_control:
  udf_flag: 1
```

In `udf.in`, use:

```yaml
user_laminar_flamespeed_flag: 1
user_g_eqn_flag: 1
user_out_flag: 1
user_turbulent_flamespeed_flag: 0
user_input_flag: 0
```

| Setting | Purpose |
| --- | --- |
| `user_laminar_flamespeed_flag: 1` | Enables the custom laminar-flame-speed calculation. |
| `user_g_eqn_flag: 1` | Enables the custom G-equation model. |
| `user_out_flag: 1` | Enables automatic tabulated flame-front diagnostics. |
| `user_turbulent_flamespeed_flag: 0` | Leaves the separate turbulent-flame-speed hook disabled; the STEP treatment is implemented through the G-equation UDF. |
| `user_input_flag: 0` | Leaves the separate user-input hook disabled. |

These switches can also be set in CONVERGE Studio. Merge the settings into your existing case files and retain other case-specific UDF switches where needed.

## 3. Provide the HDF5 flame table

Either:

- Generate a laminar-flame table using the CONVERGE table-generation tool, including **ω₂** and the required laminar-flame properties; or
- Use the provided `tlf_table.h5`, after checking that its chemistry, mixture and dilution definitions, and parameter ranges are suitable for your case.

Place `tlf_table.h5` in the case directory alongside the input files. The table must cover the pressure, unburned temperature, equivalence ratio, and dilution conditions encountered in the simulation. A conventional flame-speed table without ω₂ is not sufficient for this UDF.

## 4. Configure combustion and flame-speed corrections

In **`combust.in`**, merge the following settings into the existing sections:

```yaml
g_eqn_model:
  active: 1

sl_model:
  active: 5
  thermodiffusive_instabilities_correction:
    model: OFF
    howarth_aspden_model:
      coefficient_at_high_pressure: 1
      coefficient_at_low_pressure: 2
    berger_model:
      temp_ref: 298
      pres_ref: 100000
      equiv_ratio_ref: 0.5
      sc_sl_ratio_ref: 2.69
      gamma_equiv_ratio: -1.62
      gamma_pres: 0.4
      gamma_temp: -2.56
  sl_option:
    tlf_table_filename: tlf_table.h5

st_model:
  active: 1
  st_option:
    peters_correlation:
      a4: 0.78
      b1: 2.2
      b3: 1
```

| Setting | Meaning |
| --- | --- |
| `g_eqn_model.active: 1` | Enables the G-equation combustion model. |
| `sl_model.active: 5` | Selects the HDF5 laminar-flame table. |
| `sl_option.tlf_table_filename` | Specifies the flame table used by the case. |
| `thermodiffusive_instabilities_correction.model: OFF` | Disables CONVERGE's separate built-in TDI correction. The custom STEP correction remains active through the UDF. |
| `st_model.active: 1` | Selects the Peters turbulent flame-speed model without the G-prime model. |
| `a4`, `b1`, `b3` | Sets the Peters constants; the example uses `0.78`, `2.2`, and `1`. |

Keep the remaining chemistry, ignition, boundary-condition, turbulence, and numerical settings appropriate to your case. In particular, `b1: 2.2` is the value used in the supplied STEP setup, rather than a universal calibration for every engine.

### Critical-pressure fit and smooth transition

The UDF reuses **two Howarth–Aspden input fields** as model selectors. Both are located under:

```text
sl_model.thermodiffusive_instabilities_correction.howarth_aspden_model
```

For the revised EGR-dependent critical-pressure fit and sigmoid regime transition, set:

```yaml
coefficient_at_high_pressure: 1
coefficient_at_low_pressure: 2
```

| Setting | Value | Behavior in the UDF |
| --- | --- | --- |
| `coefficient_at_high_pressure` | `1` | Revised EGR-dependent critical-pressure fit. |
| `coefficient_at_high_pressure` | `0` | Alternative fitted critical-pressure correlation. |
| `coefficient_at_low_pressure` | `2` | Sigmoid transition between low- and high-pressure regimes. |
| `coefficient_at_low_pressure` | `1` | De Bellis-type exponential transition below the critical pressure. |
| `coefficient_at_low_pressure` | `0` | Sharp transition between low- and high-pressure regimes. |

Use the integer-valued settings shown above. These entries act as selectors, not as adjustable physical propagation coefficients. The Berger settings are not the two smooth-transition controls.

Keep the built-in correction at `model: OFF` for this setup. Do not enable the built-in Howarth–Aspden or Berger correction in addition to the UDF.

## 5. Select spatial diagnostic output in `post.in`

Add the desired variables to the existing `cells.user` list:

```yaml
cells:
  user:
    - sf_out
    - da_f_out
    - st_out
    - regime_out
    - omega_2_out
    - ka_f_out
    - ss_out
    - da_s_out
    - sl_out
    - da_out
    - ll_out
    - lf_out
    - egr_converge_out
    - s_ka_out
    - lt_out
    - uprime_out
```

The first 14 entries are selected in the supplied `post.in`; `lt_out` and `uprime_out` are additional available diagnostics.

| Variable | Meaning |
| --- | --- |
| `sl_out` | Laminar flame speed before the laminar TDI correction, s_L (m/s). |
| `sf_out` | Local propagation speed after the laminar TDI correction, s_L* (m/s). |
| `ss_out` | Local propagation speed including the turbulence–TDI contribution, s_L,t* (m/s). |
| `st_out` | Resulting turbulent flame speed (m/s). |
| `ll_out` | Original UDF laminar length scale, $\ell_L$ (m), defined below. |
| `lf_out` | UDF length scale after the laminar TDI correction, $\ell_F^*$ (m), defined below. |
| `lt_out` | Turbulent length scale $\ell_t$ supplied by CONVERGE (m). |
| `da_out` | Damköhler number based on the original laminar properties. |
| `da_f_out` | Damköhler number based on the laminar-TDI-adjusted properties. |
| `da_s_out` | Damköhler quantity used in the turbulence–TDI flame-speed calculation. |
| `omega_2_out` | Instability parameter ω₂ used in the correction. |
| `regime_out` | Low-/high-pressure blending factor: values near `0` represent the low-pressure branch and values near `1` the high-pressure branch. |
| `ka_f_out` | Effective Karlovitz number based on the laminar-TDI-adjusted properties. |
| `s_ka_out` | Karlovitz-dependent blending function. |
| `egr_converge_out` | EGR value used by the UDF. |
| `uprime_out` | Turbulence velocity scale $u'$ supplied by CONVERGE (m/s). |



### Built-in CONVERGE fields

The following built-in fields provide context for the custom STEP diagnostics:

```yaml
cells:
  combustion:
    - damkohler
    - lambda
    - lam_flamethickness
    - lam_flamespeed
    - mix_frac
    - tur_flamespeed
  turbulence:
    - eps
    - tke
    - turb_length
  passive:
    - G_EQN_TRANSPORT
    - G_EQN_PROGRESS
    - UNBURNED_TEMP
```

| Field | Meaning | Unit |
| --- | --- | --- |
| `damkohler` | CONVERGE's internal Damköhler diagnostic; distinguish it from the explicitly defined UDF diagnostics `da_out`, `da_f_out`, and `da_s_out`. | Dimensionless |
| `lambda` | Relative air–fuel ratio, conventionally $\lambda=1/\phi$. | Dimensionless |
| `lam_flamethickness` | CONVERGE's internal laminar-flame-thickness field. It is not automatically identical to the UDF's `ll_out` or `lf_out`. | m |
| `lam_flamespeed` | CONVERGE's internal laminar-flame-speed field; distinguish it from the separate UDF speed outputs. | m/s |
| `mix_frac` | Mixture fraction calculated by CONVERGE for the configured mixture. | Dimensionless |
| `tur_flamespeed` | CONVERGE's internal turbulent-flame-speed field. | m/s |
| `eps` | Turbulent kinetic-energy dissipation rate, $\varepsilon$. | m²/s³ |
| `tke` | Turbulent kinetic energy, $k$. | m²/s² |
| `turb_length` | Turbulence length scale used by CONVERGE; the UDF exports its local value as `lt_out`. | m |
| `G_EQN_TRANSPORT` | Signed G-equation field describing flame-front position; $G=0$ denotes the mean flame front. | Length, for the signed-distance formulation |
| `G_EQN_PROGRESS` | G-equation progress field used to select the automatic diagnostic bands in Section 6; distinct from the signed G field. | Dimensionless |
| `UNBURNED_TEMP` | Unburned-gas temperature field. | K |

Merge both excerpts under **one `cells:` key** in the actual `post.in`, alongside your existing geometry, general, boundary, and species selections. Select species and passive fields only where they exist in the case.

Other case-specific fields from the supplied example, such as `react_lambda`, `react_ratio`, and the `NOX` passive, may be retained where needed. They are not STEP-specific outputs and do not control the progress-band selection.

## 6. Automatic flame-front diagnostics

With `user_out_flag: 1`, the UDF automatically selects cells and writes volume-weighted diagnostic averages for three inclusive **`G_EQN_PROGRESS`** bands:

| Output suffix | Progress band |
| --- | --- |
| `low_range` | `0.1 <= G_EQN_PROGRESS <= 0.2` |
| `mid_range` | `0.4 <= G_EQN_PROGRESS <= 0.6` |
| `high_range` | `0.8 <= G_EQN_PROGRESS <= 0.9` |

These are progress-variable bands, not signed-G distance intervals or lean/stoichiometric/rich mixture classes. The output represents volume averages over selected cells, rather than surface-area averages on exact isosurfaces.

### Meaning of the `_FF` suffix

For each selected band $B$, regional or whole-mesh averages are formed as:

$$
\langle q\rangle_{\mathrm{FF},B}
= \frac{\sum_{i\in B}q_i V_i}{\sum_{i\in B}V_i},
$$

where $V_i$ is cell volume and $q_i$ is the local diagnostic. The `_FF` suffix identifies this band average, not a separate local variable. Cells with volume at or below $10^{-20}$ are excluded.

The flame-length and dimensionless-number equations in Section 5 apply **cellwise, before averaging**. In general, an average of a ratio differs from a ratio of averages. For example:

$$
\left\langle\frac{\alpha_u}{s_L}\right\rangle_{\mathrm{FF},B}
\ne
\frac{\langle\alpha_u\rangle_{\mathrm{FF},B}}
     {\langle s_L\rangle_{\mathrm{FF},B}}.
$$

### Output files and columns

For each band, the UDF writes regional and whole-mesh output using these filename stems:

```text
user_outfile_low_range_region<region_id>
user_outfile_mid_range_region<region_id>
user_outfile_high_range_region<region_id>

user_outfile_low_range_<mesh_name>
user_outfile_mid_range_<mesh_name>
user_outfile_high_range_<mesh_name>
```

The CONVERGE output interface determines the final file extension. All three bands contain the same 24 diagnostic columns, in addition to the timestep column:

```text
DAMKOHLER_FF          DA_F_OUT_FF          DA_OUT_FF
DA_S_OUT_FF           EGR_CONVERGE_OUT_FF  EQUIV_RATIO_FF
KA_F_OUT_FF           LAMBDA_FF            LAM_FLAMESPEED_FF
LAM_FLAMETHICKNESS_FF LL_OUT_FF            LF_OUT_FF
LT_OUT_FF             OMEGA_2_OUT_FF       PRESSURE_FF
REGIME_OUT_FF         SF_OUT_FF            SL_OUT_FF
SS_OUT_FF             ST_OUT_FF            TEMPERATURE_FF
UNBURNED_TEMP_FF      S_KA_OUT_FF          UPRIME_OUT_FF
```

The following columns require particular care because their names describe different quantities:

| Column | Quantity averaged within the selected band | Unit |
| --- | --- | --- |
| `LAM_FLAMETHICKNESS_FF` | CONVERGE's internal `LAM_FLAMETHICKNESS` field. | m |
| `LL_OUT_FF` | UDF original thermal-diffusion length, $\ell_L=\alpha_u/s_L$. | m |
| `LF_OUT_FF` | UDF laminar-TDI-adjusted length, $\ell_F^*=\ell_L s_L/s_L^*$. | m |
| `LT_OUT_FF` | Turbulence length scale $\ell_t$ supplied by CONVERGE. | m |
| `UPRIME_OUT_FF` | Turbulence velocity scale $u'$ supplied by CONVERGE. | m/s |
| `LAM_FLAMESPEED_FF` | CONVERGE's internal `LAM_FLAMESPEED` field. | m/s |
| `SL_OUT_FF` | UDF original laminar speed $s_L$. | m/s |
| `SF_OUT_FF` | UDF laminar-TDI-adjusted speed $s_L^*$. | m/s |
| `SS_OUT_FF` | UDF local speed including turbulence–TDI effects, $s_{L,t}^*$. | m/s |
| `ST_OUT_FF` | UDF resulting turbulent flame speed. | m/s |
| `DA_OUT_FF`, `DA_F_OUT_FF`, `KA_F_OUT_FF` | Cellwise $Da$, $Da_F$, and $Ka_F$ from Section 5, averaged after evaluation. | Dimensionless |

In particular, `LAM_FLAMETHICKNESS_FF` and `LL_OUT_FF` are **not interchangeable**: the output routine reads separate underlying fields. The UDF does not enforce equality between them. For length-scale comparisons involving the STEP corrections, use the explicitly defined `LL_OUT_FF` and `LF_OUT_FF` pair.

**`post.in` controls spatial output only.** The three progress bands and tabulated column selection are fixed in the supplied library and cannot be changed through `post.in`.

Rows with no cells in a selected band contain initialized zero values; do not interpret them as physical zero-valued averages. `LAMBDA_FF` averages the cellwise reciprocal of equivalence ratio and is not generally equal to `1 / EQUIV_RATIO_FF`.

**Known output issue in the reviewed implementation:** the conversion used for `PRESSURE_FF` makes values labelled MPa too small by a factor of 10^6. Use the regular pressure field for pressure analysis until a corrected library is provided.

## 7. Case-setup checklist

Before starting the simulation, confirm that:

- `libconverge_udf.so` is accessible through `LD_LIBRARY_PATH` on the solver nodes.
- `feature_control.udf_flag` is `1`.
- `user_laminar_flamespeed_flag`, `user_g_eqn_flag`, and `user_out_flag` are `1`.
- The G-equation model is enabled and `st_model.active` is `1`.
- `sl_model.active` is `5`, with a suitable `tlf_table.h5` containing ω₂ in the case directory.
- The built-in TDI correction is `OFF`.
- The Howarth–Aspden selector values are `1` and `2` for the revised fit and sigmoid transition.
- The desired spatial variables are selected in `post.in`.
