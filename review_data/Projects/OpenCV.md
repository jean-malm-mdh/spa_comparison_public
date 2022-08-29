
# OpenCV
Git: [https://github.com/opencv/opencv.git]
Commit: 9d16612a9b028f03747037c74d9678ef72e95e85
## Issues
### CppCheck
#### Missing Include Files
CppCheck needs to be run with `--library=opencv2`.

Based on the results, the current compile command does not include certain system libraries, giving a number of issues such as 
`~/opencv/modules/core/include/opencv2/core/cv_cpu_dispatch.h:71:0: information: Include file: <immintrin.h> not found. Please note: Cppcheck does not need standard library headers to get proper results. [missingIncludeSystem]`

