# CodeChecker Review Issues

In this document we shall document some issues discovered during the review process.
This may be limited to Quality-of-life Suggestions for developers of code review tools.

## Duplicate Reports

A number of the reports raised were in fact duplicates that were not filtered by CodeChecker's own duplication detection system. The tool uses hashing of the report to determine duplication. It does not account for cases when the same check raises multiple warnings on the same location with small differences.

From a review perspective, many of the reports may then point to the same underlying issue, but inflate the number of warnings.

### Known Causes
Template implementations in C++ will generate different versions of code constructs (structs, classes, methods ...). 

As an example, the following extract from ```opencv/modules/core/test/test_intrin_utils.hpp``` generated one report per instantiated type of the Data struct.
While it is very possible that some of the cases are real problems, some ability to merge reports or review them in bulk without removing them completely would be preferable.

```C++
template <typename R> struct Data
{
    typedef typename R::lane_type LaneType;
    typedef typename V_TypeTraits<LaneType>::int_type int_type;

    Data()
    {
        for (int i = 0; i < R::nlanes; ++i)
            d[i] = (LaneType)(i + 1); /// <-- at time of writing yields 28 variations of the following warning: either cast from 'int' to 'opencv_test::hal::intrin128::cpu_baseline::Data<cv::hal_baseline::v_uint64x2>::LaneType' (aka 'unsigned long') is ineffective, or there is loss of precision before the conversion 
    }
```

Similarly, CppCheck has checks for non-initialized member variables. If it is a template class, an individual warning will be created for each potential type encountered. 
```C++
/// llvm-project/llvm/include/llvm/ADT/SmallVector.h:1185
SmallVector() : SmallVectorImpl<T>(N) {} /// <-- Member variable 'SmallVectorStorage < char , 10 >::InlineElts' is not initialized in the constructor. Maybe it should be initialized directly in the class SmallVectorStorage < char , 10 >?

```
The same issue happens with macros, which expands to some bad code. This is a cause of a lot of potential duplication of warnings.

In the realm of software testing, this becomes an issue as test frameworks that need to be compliant with both C and C++ are typically forced to use some macro-based approach for templating. As in the example below, it will trigger a separate warning on each usage of ```TEST``` or ```TEST_SCN``` macros.

```C++
/// cmake/Utilities/KWIML/test/test_int_format.h:65
#define TEST_SCN_(SCN, T, U, STR) TEST_SCN2_(SCN, SCN, T, U, STR)
#define TEST_SCN2_(PRI, SCN, T, U, STR)                                 \
  {                                                                     \
  T const x = VALUE(T, U);                                              \
  T y;                                                                  \
  char const* str = STR;                                                \
  /// MY NOTE: sscanf will trigger unsafe buffer warning
  if(sscanf(str, "%" KWIML_INT_SCN##SCN, &y) != 1)                      \
    {                                                                   \
    y = 0;                                                              \
    }                                                                   \
  printf(LANG "KWIML_INT_SCN" #SCN ":"                                  \
         " expected [%" KWIML_INT_PRI##PRI "],"                         \
         " got [%" KWIML_INT_PRI##PRI "]", x, y);                       \
  if(x == y)                                                            \
    {                                                                   \
    printf(", PASSED\n");                                               \
    }                                                                   \
  else                                                                  \
    {                                                                   \
    printf(", FAILED\n");                                               \
    result = 0;                                                         \
    }                                                                   \
  }

#define TEST_(FMT, T, U, STR) TEST2_(FMT, FMT, T, U, STR)
#define TEST2_(PRI, SCN, T, U, STR)                                     \
  TEST_PRI_(PRI, T, U, STR)                                             \
  TEST_SCN2_(PRI, SCN, T, U, STR)

/* Concatenate T and U now to avoid expanding them.  */
#define TEST(FMT, T, U, STR) \
        TEST_(FMT, KWIML_INT_##T, KWIML_INT_##U, STR)
#define TEST2(PRI, SCN, T, U, STR) \
        TEST2_(PRI, SCN, KWIML_INT_##T, KWIML_INT_##U, STR)
#define TEST_C(C, V, PRI, T, U) \
        TEST_C_(KWIML_INT_##C, V, PRI, KWIML_INT_##T, KWIML_INT_##U)
#define TEST_PRI(PRI, T, U, STR) \
        TEST_PRI_(PRI, KWIML_INT_##T, KWIML_INT_##U, STR)
#define TEST_SCN(SCN, T, U, STR) \
        TEST_SCN_(SCN, KWIML_INT_##T, KWIML_INT_##U, STR)
#define TEST_SCN2(PRI, SCN, T, U, STR) \
        TEST_SCN2_(PRI, SCN, KWIML_INT_##T, KWIML_INT_##U, STR)
```

## Capturing Non-code build targets

The built-in compile command database logging may in some versions of CodeChecker (happened on 6.15) capture compilation actions of non-code files. This interfered with the CTU collection step, as the compilation database was reported to be "invalid".

Through the process of binary search and rerunning the analyzer, the problem was narrowed down to inclusion of *.so.1.1 files in the compile command database.

### Examples

```JSON
{
        "directory": "cmake/<build_dir>/Utilities/cmcurl",
        "command": "/usr/bin/cc -w -O3 -DNDEBUG CMakeFiles/curltest.dir/curltest.c.o -o curltest lib/libcmcurl.a -ldl /usr/lib/x86_64-linux-gnu/libssl.so /usr/lib/x86_64-linux-gnu/libcrypto.so ../cmnghttp2/libcmnghttp2.a ../cmzlib/libcmzlib.a",
        "file": "/usr/lib/x86_64-linux-gnu/libssl.so.1.1"
}
```

### Mitigation

Whenever possible, the files were simply removed from the compilation database. As the CodeChecker analysis tool analyses C/C++ code such files do not yield much. Logging the faulty DB entry would allow for faster root cause analysis.

```Python
    ## Temporarily copying each entry into _entry as they're being processed

    except (ValueError, KeyError, TypeError) as ex:
        if not compilation_database:
            LOG.error('The compile database is empty.')
        else:
            LOG.error('The compile database is not valid.')
            LOG.error('Invalid entry found: ' + str(_entry['file']))
        LOG.debug(traceback.format_exc())
        LOG.debug(ex)
        sys.exit(1)

        # Result:
        #[ERROR]log_parser.py:1350 parse_unique_log() - The compile database is not valid.
        #[ERROR]log_parser.py:1351 parse_unique_log() - Invalid entry found: /usr/lib/x86_64-linux-gnu/libssl.so.1.1
```

## Duplicate Checker aliases

Some checkers exist as aliases of other checkers. For instance, there are two checker aliases for <https://clang.llvm.org/extra/clang-tidy/checks/misc-throw-by-value-catch-by-reference.html>. Checker configurations do not account for this when running more "extreme" modes:

Depending on the configuration, the CodeChecker analyser will simply run both of them, and generate duplicate reports. This can be filtered in the tool itself, but it seems somewhat wasteful. Should the checker aliases be known, perhaps a postprocess of the selected checkers to remove duplicates before the execution would be possible?

```JSON
clang-tidy: [checks omitted]
bugprone-unhandled-self-assignment, bugprone-unused-raii, bugprone-unused-return-value, bugprone-use-after-move, bugprone-virtual-near-miss, cert-con36-c, cert-con54-cpp, cert-dcl03-c, cert-dcl16-c, cert-dcl21-cpp, cert-dcl37-c, cert-dcl50-cpp, cert-dcl51-cpp, cert-dcl54-cpp, cert-dcl58-cpp, cert-dcl59-cpp, cert-env33-c, **cert-err09-cpp**, cert-err33-c, cert-err34-c, cert-err52-cpp, cert-err58-cpp, cert-err60-cpp, **cert-err61-cpp**, cert-fio38-c, cert-flp30-c, cert-mem57-cpp, cert-msc30-c, cert-msc32-c, cert-msc50-cpp, cert-msc51-cpp, cert-oop11-cpp, cert-oop54-cpp, cert-oop57-cpp, cert-oop58-cpp, cert-pos44-c, cert-pos47-c, cert-sig30-c, cert-str34-c, [checks omitted]
```

## Assumption after Assertion Macros

A lot of test frameworks use assertion macros that expand to (typically) a do-assertion-while(0) loop, and depending on the additional boilerplate it is difficult to know what text such as "assuming the condition is true" is referring to.

### Example
The following shows warning from a potential null dereference, that is happening somewhere inside a complicated assertion macro. It is quite difficult at a glance to understand what condition the assumption is referring to.

![Messy assertion macro with assumption](/Data/Assumption_on_macro_expansion.png)

Based on similar assertions, the condition is in fact the inner if-statement that shall be true if the assertion should pass.

![Clearer example of assertion macro with assumption](/Data/Assumption_on_macro_expansion_2.png)

## Enable Bugpath Refutation in CodeChecker

A number of the false positives are related to assumptions that go against assertions. Another issue is cases where loops are assumed to never execute.

This causes a number of issues that are high-severity. Hence they are likely seen as quite important.

### Mitigation
CodeChecker documentation suggests adding static assertions to the code that force the analyser to e.g., execute loops or stop exploring paths past failed assertions.

However, this requires direct changes to the source code, until then it will cause noise. For reviewing purposes such cases may be better suited to cluster and deal with at later times, e.g., with dedicated markings.

We argue that some assumptions may be done for test code that is known to pass, and can be accounted for to post-process analysis results.

* If an assertion fails, continuing will not lead to a stable analysis result.

* Loops are likely part of the test to be run, hence we may assume they are executed at least once.

**Note: We do not remove reports outright, this is merely to cluster some common refutation issues based on information, and mark it clearly**.

## Report Conversion
Reports are converted to the .plist format in order to work with CodeChecker. CodeChecker provides a set of ready-made converters that can be invoked as follows `report-converter -t <toolname> -o <output_dir>`. In some cases the script fails to find the source files.

### Affected Files
#### Cmake
cmake/<build_dir>/Tests/CMakeLib/PseudoMemcheck/ret0.cxx
cmake/<build_dir>/Tests/CMakeLib/PseudoMemcheck/ret1.cxx
cmake/<build_dir>/Source/cmsys/RegularExpression.hxx
cmake/<build_dir>/Source/cmsys/Status.hxx
### Mitigation
The issue was traced to the usage of relative paths during the invocation of `report-converter`.

