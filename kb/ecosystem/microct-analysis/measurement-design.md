# MicroCT Measurement Design

Femoral surface measurements and tibial slice measurements are different domains; treating both as Euclidean distances produced reported errors of 11–58% against published values. IIOC height is a slice count, and frontal-projected femoral width depends on the orientation plane. Amira display lower bounds and SCANCO quantitative thresholds are not interchangeable.

The published method uses manual tibial orientation. PCA alignment is an approximation and needs independent validation: asymmetric anatomy can rotate its axes. Label-mask resampling requires nearest-neighbor interpolation to avoid mixed labels.

An earlier OA6-1RK run reported all six geometric indices within a ±10% acceptance gate, with a largest deviation of 2.89%. This does not establish detector correctness: subsequent live review found femoral landmarks visibly misplaced when a tibia-aligned frame left the femur about 23.8° skewed. The package's `docs/findings/2026-06-04-femoral-orientation-and-visual-landmarking.md` records the evidence and unresolved alternatives.
