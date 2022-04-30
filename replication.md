# Replication
In this document, we document the steps taken to perform the study.
## Prerequisites
A reasonably new version of Clang & Clang-Tidy. The version of clang used during this work was clang 15.0.0

A reasonably new version of CppCheck: During the process of the work, the version was uplifted from 2.4 to 2.7

A version of CodeChecker. The newer the better, as it is continuously being improved. The version was uplifted from 6.15 to 6.20 during the process of the work.

https://pypi.org/project/PyGithub/ - to interact with the github API for project selection and filtering

https://github.com/AlDanial/cloc - To count code lines, for filtering 

CMake

### Convenience Commands
`alias spacomp_setenv='source ~/codechecker/venv/bin/activate'`

## Build Process
`export CC_LOGGER_GCC_LIKE="gcc:g++:clang:clang++:cc:c++"` - Configures which compiler calls CodeChecker log will listen to

`cat CMakeLists.txt | grep "TEST"` - Quick way to find configuration options to enable for testing

`~/<proj_folder>cmake -B cmakebuild -DCMAKE_EXPORT_COMPILE_COMMANDS=ON [-DCMAKE_BUILD_TYPE=Release] .`

## Analysis Process
`source $HOME/codechecker/venv/bin/activate` - Run CodeChecker using the installed virtualenv.

`cd cmakebuild && CodeChecker log -b "make -j4 && make test"`
