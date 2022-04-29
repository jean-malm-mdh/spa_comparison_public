# Public Version of SPA Evaluation on Test Code

## CodeChecker Analysis
Default: `CodeChecker analyze -o codechecker_results`
Extreme: `CodeChecker analyze --enable extreme --ctu-all -o codechecker_ctu_extreme`

### CodeChecker Skipfile
```
-/tmp/*
+*/*\[tT\]est*
+*/*\[tT\]est*.c(c|pp|xx)?
+*/*\[tT\]est*.hpp
-*
```
## CppCheck
Default: `cppcheck -j8 --plist-output=$(PWD)/cppcheck_results --file-filter=*test* --file-filter=*Test* -i /tmp/` 
Extreme: `cppcheck -j8 --plist-output=$(PWD)/cppcheck_results --file-filter=*test* --file-filter=*Test* -i /tmp/ --enable=warning --enable=style --cppcheck-build-dir=<project_build_dir> --max-ctu-depth=200`
