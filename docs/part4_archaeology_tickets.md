# Part 4 — Subsystem Biography

Subsystem Part 4 — Biographical Information

Historical Background

This subsystem has had more than three decades of historical development. The first commit of relevance to this subsystem is commit 1f21be5, made on March 14, 1994 by R. Halvorsen that implemented the first single layer water balance crop model and driver. Later, in 1996, Halvorsen extended the GROWTH model in commit 5929f6e through the use of radiation-use efficiency for dry matter accrual.

This model has since been updated and improved. According to comments in the source code in growth.f90, the GROWTH model was updated to Fortran 90 in 1998 and the radiation-use-efficiency part was re-fitted using the 1971-1996 trial data. Additionally, Git shows that on July 22, 1998, there is a commit 695e7c5 that implemented the Fortran 90 thermal-time phenology stages. Finally, in 2009, M. Chen organized the important model constants into the shared crop_params module through commit 4f9fc3f.

This is one of the most significant changes, dating back to 2013. In commit 9d6f6e1, named Canopy expansion and dry-matter partitioning, off 1971-1996 trials, K. Osei included canopy.f90 and partition.f90 files. Until then, GROWTH model used one sine curve to represent Leaf Area Index between Emergence and Maturity phases. Canopy module now saves LAI state for each day. LAI grows during vegetative stage and senesces during grain filling. Additionally, water stress on one day is no longer automatically recovered the next but affects further canopy development.

## Load-Bearing / Puzzling Code

Historical functionality of special interest can be found in the calculation of water-stress fraction in file growth.f90. GROWTH models calculate their initial water-stress fraction by the formula SW / (AWC * 100), while WATBAL uses another approach – SW / SWMAX. On first sight, correcting GROWTH so that both approaches would be the same seems like an obvious decision.

Nevertheless, commit 208d017 of February 11, 2021 by K. Osei clearly outlines this discrepancy as per MRD-204. The source stresses that the historical GROWTH formula should not just be “fixed” to agree with WATBAL. In the comments, the reason for this discrepancy between the two formulas is explained, stating that they differ at the rooting depth which is not equal to 1000 mm, but the historical behaviour was deliberately kept until it is verified with the right validation harness (MRD-231).

These facts illustrate the significance of Git archaeology in this particular subsystem. Code which seems to be erroneous can be the one representing some behaviour relied upon by historical model results.


# Part 5 — Open Tickets

## MRD-118 — Ascending DOY Assumption

## Location

MRD-118 is found in model/cropmod.f, particularly on lines 43–55, where daily weather data are read. The code actually assumes the weather data records are in ascending day-of-year (DOY) and that this is not verified.

## Root Cause

The cropmod.f module reads a weather record and saves the DOY and weather data directly into arrays in the same order they are in the input deck. There is no verification that the current DOY is greater than the last one. There is also no sorting done before WATBAL and GROWTH are called.

Both modules read the weather data sequentially; therefore, any misordered input deck can make the model process the weather data on the wrong timeline. MRD-118 suggests that such an input deck will create an incorrect water balance but will not give any error message.


## What a Fix Would Have to Cover

A fix in the future would involve changes to the input-validation part in cropmod.f. Changes will have to be made to validate whether the DOY values are in the right order before passing the weather data arrays to WATBAL and GROWTH. What action should be taken on finding an out-of-order record needs to be decided – such as rejection or explicit handling of the order. The proposed solution will need to be verified to ensure correctness of previously valid input decks.

## MRD-204 — Water Stress Calculation Formulas Diverge

## Location

MRD-204 concerns itself with model/growth.f90 and model/waterbal.f. Growth.f90 calculates the water stress fraction using SW / (AWC * 100) on lines 60-74. Comments in waterbal.f indicate that WATBAL calculates the stress fraction by using SWCUR / SWMAX.

## Root Cause

The two components of the model use different representations for the available soil water in the calculation of water stress. GROWTH uses AWC as a per unit depth parameter and calculates stress fraction using SW / (AWC * 100). WATBAL calculates stress using the total profile water relative to its maximum as discussed in MRD-204.
-

This is a well-known behavior of historical calculations rather than an unexpected programming error in the code. The comments in growth.f90 clearly state that historical calculations must never be just modified to fit WATBAL.

## What a Fix Would Involve

It would involve choosing a validated definition for water stress and then modifying GROWTH, WATBAL, or their interface, so that both functions use the same values. Since modifications could affect historical calculations of crop growth and yields, comparison to the trusted historical calculations would be required before making any changes. Thus, the ticket is related to MRD-231 and Missing Validation Harness.

## MRD-231 — Missing Validation Harness

## Location

MRD-231 is a subsystem-wide issue rather than an error in a single line. ISSUES.md mentions the request from Agronomy to port cropmod from Fortran, but states that there is no validation harness available for checking whether a new implementation gives "the same answer." MRD-231 can also be found in files such as growth.f90, params.f90, and pet.f90.

## Root Cause

Crop model has gathered calibrated equations, constants, and historical behaviour through many years; however, there is no validation suite which would execute a set of known inputs and check that the full set of expected model outputs is produced. As the result, developers cannot reliably distinguish between intended improvement and a change which breaks some behaviour used by existing forecasts.

For instance, params.f90 discourages changes to fitted parameters without re-validation, while pet.f90 has another method for computing evapotranspiration which is not used in production since changing it affects all forecasts without a validation system to prove the changed behaviour is good.

## What a Fix Would Need to Touch

Fixing of MRD-231 would need creation of a validation system for cropmod prior to any significant rewrite attempt. Reference input decks with known outputs would have to be collected as reference cases and a new implementation would be run against them with comparison of the results against the Fortran implementation. It would only be possible to define what are acceptable differences in order to determine whether the port keeps the required model behaviour.

## Part 6 – Here Be Dragons

There are some historical assumptions and dependencies in the crop-growth subsystem that any new developer needs to know about prior to making any modifications.

1. Fixed-width and Column-exact formats

Evidence: cropmod.f uses the model data in fixed Fortran format and the workflow in Meridian relies on exactly defined input and output formats of the model.

Risk: Any small format change could result in reading data incorrectly or breaking down the interaction between the model and other subsystems of Meridian.

2. Weather Data Must Be Sorted by DOY

Evidence: cropmod.f does not validate or sort weather data which is assumed to be provided in the ascending order of day of year (MRD-118).

Risk: Incorrectly provided data will result in the wrong results from the model without any failure occurring.

3. Different Calculation of Water Stress

Evidence: in growth.f90 there is historical SW/(AWC*100) calculation kept, while WATBAL uses profile-based calculations. It is documented in MRD-204 and commit 208d017.

Risk: A developer can consider these formulas as wrong and fix GROWTH unintentionally changing historical behaviour of the model.

4. Calibrated Constants

Evidence: Model parameters are calibrated via historical agronomy trials; includes trials from 1971–1996.

Risk: What appears to be arbitrary values may have been calibrated. Modifying such values without validating the change could modify crop-growth and yield predictions.

5. Lack of Validation Harness

Evidence: MRD-231 specifies that there is not a validation harness capable of checking if a replacement model implementation gives the same model output.

Risk: Replacing or refactoring the Fortran model code is risky because developers have difficulty proving that the historical behavior has been maintained.

6. Stateful Canopy Growth

Evidence: Following the Canopy change in 2013 from commit 9d6f6e1, LAI is carried forward from day-to-day as opposed to recalculating each day.

Risk: Changes to crop growth or water stress on any one day impacts future canopy growth since what is thought to be a local change impacts future days.

7. Legacy Fixed-File Assumptions

Evidence: The model uses fixed files like MERIDIAN.DAT as part of its workflow per MRD-201.

Risk: Running multiple model instances using the same working directory risks interacting with one another in unintended ways.
