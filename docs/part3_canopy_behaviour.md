## 3.3 Interface Contract

The Canopy subsystem is exposed internally through the `GROWTH` Fortran subroutine and externally through the `CROPMOD` executable. `GROWTH` receives daily environmental and soil-water data, updates crop growth state over the simulation period, and returns daily biomass, daily leaf area index, and final grain yield.

### 3.3.1 `GROWTH` Subroutine Interface

The crop-growth routine is declared as:

```fortran
subroutine growth(n, idoy, tmax, tmin, srad, sw, awc, xlat, biom, xlai, yield)
```

Its contract is:

| Parameter | Direction | Meaning |
|---|---|---|
| `n` | Input | Number of daily simulation records |
| `idoy(n)` | Input | Day-of-year for each record |
| `tmax(n)` | Input | Daily maximum air temperature |
| `tmin(n)` | Input | Daily minimum air temperature |
| `srad(n)` | Input | Daily incoming solar radiation |
| `sw(n)` | Input | Daily profile soil water supplied by `WATBAL` |
| `awc` | Input | Available water capacity used in the crop water-stress calculation |
| `xlat` | Input | Latitude used in the photoperiod calculation |
| `biom(n)` | Output | Daily total above-ground biomass |
| `xlai(n)` | Output | Daily green leaf area index |
| `yield` | Output | Final grain yield in t/ha |

`RAIN` and `RDEPTH` are not direct inputs to `GROWTH`. They are consumed by `WATBAL`, which calculates the `SW` array before `CROPMOD` calls `GROWTH`.

### 3.3.2 Behavioural Invariants

The following conditions define important behavioural assumptions of the crop-growth subsystem:

- Crop development moves forward through the runtime states `PRESOW → VEG → GRAINFIL → MATURE`; it does not transition backwards.
- Phenological progress is driven by photoperiod-adjusted thermal time.
- The grain-fill fraction `gf_frac` is constrained to the range `0.0` to `1.0`.
- Water stress is ultimately constrained to the range `0.0` to `1.0`.
- Canopy LAI cannot become negative and vegetative expansion is capped at `LAIMAX = 5.2`.
- Total biomass is calculated from the accumulated leaf, stem, and grain dry-matter pools.
- Once the crop reaches `STAGE_MATURE`, no further dry-matter partitioning occurs.
- The final reported yield is calculated from the grain pool rather than from a fixed harvest index.

An important historical exception is the water-stress calculation. `GROWTH` calculates its initial soil-water fraction using:

```text
SW / (AWC × 100)
```

while `WATBAL` uses its own soil-water capacity calculation. This known divergence is documented as `MRD-204` and is intentionally preserved in the current implementation.

### 3.3.3 External `CROPMOD` Input Contract

The surrounding executable reads a fixed-width file named:

```text
MERIDIAN.DAT
```

from the current working directory.

The expected records are:

| Record | Fields | Fortran Format |
|---|---|---|
| Header 1 | `STNID`, `IYEAR`, `XLAT` | `A8, I4, F8.3` |
| Header 2 | `AWC`, `RDEPTH` | `F8.2, F8.2` |
| Daily record | `IDOY`, `TMAX`, `TMIN`, `RAIN`, `SRAD` | `I3, F6.1, F6.1, F6.1, F6.2` |

Daily records terminate when a negative value is encountered in the `IDOY` field, when input ends, or when the fixed capacity of 366 daily records is reached.

The driver assumes that day-of-year records are supplied in ascending order. This ordering is not validated before the simulation runs.

### 3.3.4 External `CROPMOD` Output Contract

After `WATBAL` and `GROWTH` complete, `CROPMOD` writes:

```text
MERIDIAN.OUT
```

The output contains:

1. A banner containing station ID, year, and number of processed days.
2. A fixed column heading line.
3. One fixed-width daily record containing:
   - `DOY`
   - `SW`
   - `ET`
   - `DRAIN`
   - `BIOM`
   - `LAI`
4. A final `YIELD` record.

Daily records use the format:

```fortran
FORMAT (I4, 1X, F7.2, 1X, F7.3, 1X, F7.3, 1X, F7.1, 1X, F7.3)
```

and final yield uses:

```fortran
FORMAT ('YIELD ', F10.3)
```

The fixed-width representation is part of the interface contract and must be preserved by downstream parsers. The existing documentation already identifies this fixed-column dependency as important to avoiding parsing failures.

### 3.3.5 Error Behaviour

`GROWTH` itself does not perform file I/O or terminate the program with explicit error codes. File-related errors are handled by the surrounding `CROPMOD` driver.

The verified driver-level failure cases are:

| Condition | Behaviour |
|---|---|
| `MERIDIAN.DAT` cannot be opened | Prints `CROPMOD: cannot open MERIDIAN.DAT` and executes `STOP 2` |
| First header record cannot be parsed | Prints `CROPMOD: bad header record` and executes `STOP 3` |
| No daily weather records are accepted | Prints `CROPMOD: no weather records` and executes `STOP 4` |
| Successful execution | Writes `MERIDIAN.OUT`, prints `CROPMOD: ok, N days`, and stops normally |

Only the first header read (`STNID`, `IYEAR`, `XLAT`) is explicitly guarded using `IOSTAT`. The second header read containing `AWC` and `RDEPTH` does not use the same explicit error check, so it should not be described as guaranteed to produce `STOP 3` on malformed input.