# Misc. Review/Analysis issues

## Match Checker Configuration to C/C++ Version

A number of reports were raised due to checkers being run on code that was intended to support older versions of C/C++. While modernizing is good, such reports will essentially be additional noise if the project is held back to work with older C++ compilers.

### Known Causes

One cause is due to the analyzer framework not recognizing which C++ version should be supported. While gcc/g++ supports passing ```-std=<lang_version>``` this is then dependent on the compile command database having the correct information.

This was not the case in e.g., the implementation of the bullet3 physics simulator library. Upon further inspection it is clearly stated that the project is meant to support C++03 compilers. As such issues/recommendations related to e.g., move semantics etc (which were not added until C++11) are not needed.

### Fix Suggestion

In a real setting, the mismatch would likely not be a problem as the assumption is that the used version of the analyzer/compiler will match the project setup.

A tool-specific approach is to group the checkers based on their minimum language version. Then unsuitable versions can be disabled in bulk instead of having to do it one-by-one, as is currently the case.

Some checkers seem to account for this, as indicated in <https://clang.llvm.org/extra/clang-tidy/checks/modernize-use-using.html>.

#### Checkers affected

google-explicit-constructor -- requires explicit keyword
