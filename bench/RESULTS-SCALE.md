# Compile-time scaling

Recorded 2026-11-14 03:48 UTC; CPU: Intel(R) Xeon(R) Processor @ 2.10GHz.
Regenerate with `python3 bench/scale.py --markdown bench/RESULTS-SCALE.md`.

Wall time of one build (best of three), in milliseconds. `front-end` is
`rustc --emit=metadata`, the type-checking floor every rustc pipeline pays.

| lines | rmc | cb | cranelift | llvm-O0 | llvm-O3 | front-end |
| ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 973 | 14 | 210 | 67 | 61 | 139 | 42 |
| 9253 | 127 | 1063 | 412 | 391 | 1469 | 248 |
| 46053 | 1231 | 6108 | 2029 | 1826 | 14301 | 1121 |
