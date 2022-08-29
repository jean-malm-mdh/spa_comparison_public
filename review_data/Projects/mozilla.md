# Mozilla
Version:
Test Code lines analyzed:
Issues:


## Misc Info

During the build process, the build logger captured a number of "/tmp/conftest.[a-z0-9]+.c(pp)?" files that seems to have been removed during the build process itself. CodeChecker can simply skip them, but cppcheck had some trouble even when using the `-i /tmp/` flag

## Static Tools Used
Mozilla has their own version of clang with some custom checkers. Info can be found at [https://firefox-source-docs.mozilla.org/code-quality/index.html]. They've provided also the configuration that is run, along with motivations when checks are disabled etc.

To some extent, the findings during this work reflect the information listed.

For instance according to [https://firefox-source-docs.mozilla.org/code-quality/static-analysis/existing.html#false-positives] "A lot of \[false positives\] are due to the analyzer having difficulties following the relatively complicated error handling in various preprocessor macros." 