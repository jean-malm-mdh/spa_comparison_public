# Report Overview

In this companion piece we shall discuss some notable reports found during the work.

Where relevant we include examples and suggest applicable "fix-it" solutions with the aim to reduce the manual effort spent to fix issues.

Examples are annotated with inline comments to show locations in the source file, as well as clarificating notes.

## Legend/Abbreviations

FP - False Positive

Clang SA - Clang Static Analyzer

CFG - Control-flow Graph

## Reports

In this section we present the reports.

## C++ Special Member Functions (Destructor, Assignment, Copy Constructor and Move Semantics)

Severity - Low

Finds so-called "rule of three/rule of five" violations. These rules state that if one of these member functions have been explicitly defined, all of them likely are needed.
The rule of three encapsulates destructor, copy constructor and assignment operator. The rule of five additionally includes the move constructor and move assignment.

As noted in <https://en.cppreference.com/w/cpp/language/rule_of_three> this is both a lost performance opportunity (for the move semantics) and also a potential source of errors. Notably, it may cause erroneously creating of shallow copies in cases where deep copying is preferable. For instance, a general copy constructor will simply copy e.g., a pointer field, and may then cause unintended aliasing or worse mutations of shared states.

### C++ Special Member Functions - Examples

CMake parses a number of text files such as test logs, for which classes violate this design rule. In such cases their lifetime is quite short (see examples below), and not likely to be copied which may be a reason for the lack of implemented assignment operator or copy constructors.

```C++
/// 
bool cmParseBlanketJSCoverage::ReadJSONFile(std::string const& file)
{
  cmParseBlanketJSCoverage::JSONParser parser(this->Coverage);
  cmCTestOptionalLog(this->CTest, HANDLER_VERBOSE_OUTPUT,
                     "Parsing " << file << std::endl, this->Coverage.Quiet);
  parser.ParseFile(file);
  return true;
}
```

```C++
  LogParser out(this, "log-out> ");
  out.Process("<?xml version=\"1.0\"?>\n"
              "<log>\n");
  OutputLogger err(this->Log, "log-err> ");
  this->RunChild(hg_log, &out, &err);
  out.Process("</log>\n");
  return true;
```

Some of CMake's CTest framework classes cover the rule of three. However they do not support move semantics. Given that they appear to be objects that are seldom created, the performance loss is not *that big*. For instance, cmCTestBuildAndTestCaptureRAII is created once every invocation of CTest's --build-and-test command (for details, see <https://cmake.org/cmake/help/book/mastering-cmake/chapter/Testing%20With%20CMake%20and%20CTest.html>)

```C++
int cmCTestCoverageHandler::RunBullseyeCommand(
// ...
// Note: 
cmCTestRunProcess runCoverageSrc;
runCoverageSrc.SetCommand(program.c_str());
runCoverageSrc.AddArgument(arg);
std::string stdoutFile =
  cmStrCat(cont->BinaryDir, "/Testing/Temporary/",
           this->GetCTestInstance()->GetCurrentTag(), '-', cmd);
std::string stderrFile = stdoutFile;
stdoutFile += ".stdout";
stderrFile += ".stderr";
runCoverageSrc.SetStdoutFile(stdoutFile.c_str());
runCoverageSrc.SetStderrFile(stderrFile.c_str());
if (!runCoverageSrc.StartProcess()) {
  cmCTestLog(this->CTest, ERROR_MESSAGE,
             "Could not run : " << program << " " << arg << "\n"
                                << "kwsys process state : "
                                << runCoverageSrc.GetProcessState());
  return 0;
}
  // since we set the output file names wait for it to end
  runCoverageSrc.WaitForExit();
  outputFile = stdoutFile;
```

```C++
int cmCTestBuildAndTestHandler::RunCMakeAndTest(std::string* outstring) {
// ...
cmCTestBuildAndTestCaptureRAII captureRAII(cm, cmakeOutString);
static_cast<void>(captureRAII);
/// Note: captureRAII is not accessed directly after this point. It is simply used to hold some callback functionality.
```

If there is a mismatch between analyzer and project target language, noisy issues will be reported.

The example below is a generated class used by an older version of GTest. 
As seem, it violates the rule-of-three, as the assignment operator is explicitly declared.

The bullet project explicitly targets the C++03 version, which predates the 
addition of move semantics. The generated report when running Clang directly on the generated compile database did not 
account for this versioning, hence it raised a number of 

```C++
/// bullet3/test/gtest-1.7.0/include/gtest/internal/gtest-param-util-generated.h
// Used in the Values() function to provide polymorphic capabilities.
template <typename T1>
class ValueArray1
{
public:
 explicit ValueArray1(T1 v1) : v1_(v1) {}

 template <typename T>
 operator ParamGenerator<T>() const
 {
  return ValuesIn(&v1_, &v1_ + 1);
 }

private:
 // No implementation - assignment is unsupported.
 void operator=(const ValueArray1& other);

 const T1 v1_;
};
```

### C++ Special Member Functions - Suppression

Clang-tidy provides flags to suppress combinations of these warnings, see <https://clang.llvm.org/extra/clang-tidy/checks/cppcoreguidelines-special-member-functions.html#options> for details.

### C++ Special Member Functions - Fixing

It is difficult to fix it in the general case without expert knowledge of the system in question. However, explicitly declaring the methods using (```= default``` or ```= delete```) would make the potential issue more visible in the code itself.

## Avoid Magic Numbers

Severity - Style

This issue is mostly one of making understandability of code difficult. Magic numbers are typically numerical constants that occur in code. The issue is that they say nothing about the developer's selection of the number in question.

### Avoid Magic Numbers - Examples

In the example below, it is very difficult to know whether there is any meaning behind the value 1477, or if it is some random number used.

```C++
/// cmake/Tests/FindPackageModeMakefileTest/foo.cpp
#include "foo.h"

int foo()
{
  return 1477;
}
```

While it has a comment, the choice of sleeping for *specifically 6 seconds* in the code below is not motivated. This specific case also exposes the test to some amount of non-determinism, as there may be cases where 6 seconds is not enough.

```C
/// cmake/Source/kwsys/testProcess.c:358
static int test10_grandchild(int argc, const char* argv[])
{
  /* The grandchild just sleeps for a few seconds and handles signals.  */
  (void)argc;
  (void)argv;
  fprintf(stdout, "Output on stdout from grandchild before sleep.\n");
  fprintf(stderr, "Output on stderr from grandchild before sleep.\n");
  fflush(stdout);
  fflush(stderr);
  /* Sleep for 6 seconds.  */
  testProcess_sleep(6);
  fprintf(stdout, "Output on stdout from grandchild after sleep.\n");
  fprintf(stderr, "Output on stderr from grandchild after sleep.\n");
  fflush(stdout);
  fflush(stderr);
  return 0;
}
```

Here we shall note another case of magic numbers. In addition to the array size of 10 (the number of different tests) the values in the test parameters are generating warnings (especially *timeouts*). The issue comes from a mixing of integer values used as flags (shares, otputs, delays) or values. At the least, passing the flags as symbolic values (ON/OFF) would make the code more understandable.

```C
    /// cmake/Source/kwsys/testProcess.c:656
    int values[10] = { 0, 123, 1, 1, 0, 0, 0, 0, 1, 1 };
    int shares[10] = { 0, 0, 0, 0, 0, 0, 0, 0, 1, 1 };
    int outputs[10] = { 1, 1, 1, 1, 1, 0, 1, 1, 1, 1 };
    int delays[10] = { 0, 0, 0, 0, 0, 1, 0, 0, 0, 0 };
    double timeouts[10] = { 10, 10, 10, 30, 30, 10, -1, 10, 6, 4 };
    int polls[10] = { 0, 0, 0, 0, 0, 0, 1, 0, 0, 0 };
    int repeat[10] = { 257, 1, 1, 1, 1, 1, 1, 1, 1, 1 };
    int createNewGroups[10] = { 0, 0, 0, 0, 0, 0, 0, 0, 1, 1 };
    unsigned int interruptDelays[10] = { 0, 0, 0, 0, 0, 0, 0, 0, 3, 2 };
    
    /// [...]
    r = runChild(cmd, states[n - 1], exceptions[n - 1], values[n - 1],
                 shares[n - 1], outputs[n - 1], delays[n - 1], timeouts[n - 1],
                 polls[n - 1], repeat[n - 1], 0, createNewGroups[n - 1],
                 interruptDelays[n - 1]);
```

The example below shows a rather common pattern, where a magic number (4000) is used to set the size of some buffer. When using it, some number related to the size (e.g., the maximum number of allowed elements) is then used. This connection between 4000 and 3999, and the choice behind why 4000 is relevant is not visible.

```C++
/// xerces-c/tests/src/DOM/DOMTest/DTest.cpp
//temp XMLCh String Buffer
XMLCh tempStr[4000];
XMLCh tempStr2[4000];
XMLCh tempStr3[4000];
XMLCh tempStr4[4000];
XMLCh tempStr5[4000];
/// [...]
bool DOMTest::docBuilder(DOMDocument* document, XMLCh* nameIn)
/// [...]
    //name + "FirstElement", name + "firstElement"
    XMLString::catString(tempStr, name);
    XMLString::transcode("FirstElement", tempStr2, 3999);
    XMLString::catString(tempStr, tempStr2);
    XMLString::catString(tempStr2, name);
    XMLString::transcode("firstElement", tempStr3, 3999);
    XMLString::catString(tempStr2, tempStr3);
    docFirstElement->setAttribute(tempStr, tempStr2);
    DOMAttr* docFirstElementAttr = docFirstElement->getAttributeNode(tempStr);
```

Magic numbers may also be hexadecimal. That makes it even more difficult to understand connections between values and meanings at a glance.

```C++
  /// warzone2100/3rdparty/utf8proc/test/iterate.c:42
    // Check valid sequences that were considered valid erroneously before
    buf[0] = 0xef;
    buf[1] = 0xb7;
    for (byt = 0x90; byt < 0xa0; byt++) {
        CHECKVALID(2, byt, 3);
    }
    // Check 0xfffe and 0xffff
    buf[1] = 0xbf;
    CHECKVALID(2, 0xbe, 3);
    CHECKVALID(2, 0xbf, 3);
    // Check 0x??fffe & 0x??ffff
    for (byt = 0x1fffe; byt < 0x110000; byt += 0x10000) {
        buf[0] = 0xf0 | (byt >> 18);
        buf[1] = 0x80 | ((byt >> 12) & 0x3f);
        CHECKVALID(3, 0xbe, 4);
        CHECKVALID(3, 0xbf, 4);
    }
```

Some utility functions fo testing of UTF8 uses hexcodes to compare against. In the example below it seems to compare c with some specific unicode characters, but why they are chosen specifically is not shown <https://www.utf8-chartable.de/unicode-utf8-table.pl>.

```C++
/// warzone2100/3rdparty/utf8proc/test/charwidth.c
static int my_isprint(int c) {
    int cat = utf8proc_get_property(c)->category;
    return (UTF8PROC_CATEGORY_LU <= cat && cat <= UTF8PROC_CATEGORY_ZS) ||
           (c == 0x0601 || c == 0x0602 || c == 0x0603 || c == 0x06dd || c == 0x00ad) ||
           (cat == UTF8PROC_CATEGORY_CN) || (cat == UTF8PROC_CATEGORY_CO);
}

```

### Avoid Magic Numbers - Fixing

The general consensus [<a href="https://refactoring.guru/replace-magic-number-with-symbolic-constant">1</a>][<a href="https://www.c-sharpcorner.com/uploadfile/GemingLeader/refactoring-magic-numbers/">2</a>][<a href="https://www.codereadability.com/magic-numbers/">3</a>][<a href="https://refactoring.com/catalog/replaceMagicLiteral.html">4</a>] is to replace such numbers with aptly named symbolical constants. That way the developer's intent can be made clear to all who read the code.

While such refactorings are not difficult to a developer, they require some domain/expert knowledge to give good and understandable names.


## Use after Move

Severity - High
<https://clang.llvm.org/extra/clang-tidy/checks/bugprone-use-after-move.html>

The check looks for potential usages of data that has been set to be moved explicitly by developers, through the usage of std::move().

In general, the reports from this check have been on code where the violation appears to be made intentionally, for testing of move semantic implementation. While this is not a false positive, it will generate high-severity reports that are in fact not a problem.

### Use after Move - Examples

In the example below, both pipe.get() and stream.get() throw warnings. As is indicated by the surrounding context, this is a test case for checking that the move semantics are implemented as intended, and similar tests would likely throw such warnings.

```C++
    /// cmake/Tests/CMakeLib/testUVRAII.cxx:125
    cm::uv_pipe_ptr pipe;
    pipe.init(Loop, 0);

    cm::uv_stream_ptr stream = std::move(pipe);
    if (pipe.get()) {
      std::cerr << "Move should be sure to invalidate the previous ptr"
                << std::endl;
      return false;
    }
    cm::uv_handle_ptr handle = std::move(stream);
    if (stream.get()) {
      std::cerr << "Move should be sure to invalidate the previous ptr"
                << std::endl;
      return false;
    }
```

```C++
static bool testMoveConstruct(std::vector<Event>& expected)
{
  cm::optional<EventLogger> o1{ 4 };
  const cm::optional<EventLogger> o2{ std::move(o1) };
  cm::optional<EventLogger> o3{};
  const cm::optional<EventLogger> o4{ std::move(o3) };

  expected = {
    { Event::VALUE_CONSTRUCT, &*o1, nullptr, 4 },
    { Event::MOVE_CONSTRUCT, &*o2, &*o1, 4 },
    { Event::DESTRUCT, &*o2, nullptr, 4 },
    { Event::DESTRUCT, &*o1, nullptr, 4 },
  };
  return true;
}
```

In the example below, it is quite clear that the move semantics are being tested, as s1.data() is asserted against nullptr despite being initialized to the string "abc".

```C++
/// cmake/Tests/CMakeLib/testString.cxx
static bool testConstructMove()
{
  std::cout << "testConstructMove()\n";
  cm::String s1 = std::string("abc");
  cm::String s2 = std::move(s1);
  ASSERT_TRUE(s1.data() == nullptr);
  ASSERT_TRUE(s1.size() == 0);
  ASSERT_TRUE(s2.size() == 3);
  ASSERT_TRUE(std::strncmp(s2.data(), "abc", 3) == 0);
  ASSERT_TRUE(s1.is_stable());
  ASSERT_TRUE(s2.is_stable());
  return true;
}

```

### Use after Move - Fixing

Some heuristic pattern-based automated suppression method. For instance, if the usage occurs as part of a condition or an assert statement, it may be possible to pre-classify it as intentional.

## CallAndMessage

Severity - High

This core checker is responsible for modelling the function calls, and for raising issues such as warnings about uninitialized values.

To recap, Clang Static Analyzer (SA) explores the exploded graph of the program, which can be likened to a "unfolded" control-flow graph. During this exploration it will generate paths by making assumptions that may not reflect the actual execution, and this can lead to FPs.

### CallAndMessage - Examples

From a test-code perspective, collecting test inputs in an array or similar that is then traversed in a loop is problematic, as Clang SA will most likely try to explore the path where e.g., loops are skipped (e.g. by falsifying the loop condition in the program point before the loop). In the example below the analyser has skipped the execution of the first loop in testUTF8, which causes issues

```C++
/// Tests/CMakeLib/testUTF8.cxx:165
struct test_utf8_entry
{
  int n;
  test_utf8_char str;
  unsigned int chr;
};
static test_utf8_entry const good_entry[] = {
  { 1, "\x20\x00\x00\x00", 0x0020 },   /* Space.  */
  { 2, "\xC2\xA9\x00\x00", 0x00A9 },   /* Copyright.  */
  { 3, "\xE2\x80\x98\x00", 0x2018 },   /* Open-single-quote.  */
  { 3, "\xE2\x80\x99\x00", 0x2019 },   /* Close-single-quote.  */
  { 4, "\xF0\xA3\x8E\xB4", 0x233B4 },  /* Example from RFC 3629.  */
  { 3, "\xED\x80\x80\x00", 0xD000 },   /* Valid 0xED prefixed codepoint.  */
  { 4, "\xF4\x8F\xBF\xBF", 0x10FFFF }, /* Highest valid RFC codepoint. */
  { 0, { 0, 0, 0, 0, 0 }, 0 }
};

/// Note: Code omitted for brevity

int testUTF8(int /*unused*/, char* /*unused*/ [])
{
  int result = 0;
  for (test_utf8_entry const* e = good_entry; e->n; ++e) {
/// ...
```

In the examples below, TASSERT is an example of a "soft" assertion, that just notes that a failure has occured and continues. While this can be good to check if there are multiple problems in a test, it results in analysers raising high-severity issues that are technically possible. If the test continues it will be in a unstable state anyway. As such, it would be good to either bypass the code related to the failed null-check or simply use a more strict assert.

```C++
/// xerces-c/tests/src/DOM/DOMMemTest/DOMMemTest.cpp:38
bool errorOccurred = false;

#define UNUSED(x) { if(x!=0){} }

#define TASSERT(c) tassert((c), __FILE__, __LINE__)

void tassert(bool c, const char *file, int line)
{
    if (!c) {
        printf("Failure.  Line %d,   file %s\n", line, file);
        errorOccurred = true;
    }
}
/// [...]
/// xerces-c/tests/src/DOM/DOMMemTest/DOMMemTest.cpp:522
a = cloned->getAttributeNode(X("CTestAttr"));
TASSERT(a != 0);
s = a->getValue();  <-- Warns about potential null pointer dereference.

/// [...]
/// xerces-c/tests/src/DOM/DOMMemTest/DOMMemTest.cpp:755
DOMDocumentType* dt = impl->createDocumentType(X("foo:docName"), X("pubId"), X("http://sysId"));
TASSERT(dt != 0);
TASSERT(dt->getNodeType() == DOMNode::DOCUMENT_TYPE_NODE);  <-- Warns about potential null pointer dereference.
```

## Null Dereference

**NOTE: This is a core checker. It should not be turned off!**

**Severity: High!**

This check is similar to "call and message", however deals specifically with checking for null pointers being dereferenced.

It suffers from the same type of issues in test cases, specifically when assertions would normally stop the test from moving on.

It is quite common to check if some pointer is not null. If the analyser does not recognise it as a non-returning assertion it will additionally try the path where the check failed and then the test continues. In such cases, the warning would be considered a false positive, as the assertion is otherwise stopping the dereference from being reached.

Another possible explanation is that the assertion is meant to force a crash, and it does so by creating a segmentation error. In such cases, the issue is of course intentional, but the approach is somewhat unfortunately chosen from an analysis point of view.

### Null Dereference - Examples

In the destructor, the checker gives a warning that the MOZ_RELEASE_ASSERT dereferences a null pointer. In fact this happens for every such assertion when exploring the path where the analyser assumes that the assert fails. Based on the documentation provided in `mozilla-central/mfbt/Assertions.h` this is intended in order to force a crash.

```C++
///mozilla-central/mozglue/tests/TestPrintf.cpp:37
class TestPrintfTarget : public mozilla::PrintfTarget {
 public:
  static const char* test_string;

  TestPrintfTarget() : mOut(0) { memset(mBuffer, '\0', sizeof(mBuffer)); }

  ~TestPrintfTarget() {
    MOZ_RELEASE_ASSERT(mOut == strlen(test_string));
    MOZ_RELEASE_ASSERT(strncmp(mBuffer, test_string, strlen(test_string)) == 0);
  }

  bool append(const char* sp, size_t len) override {
    if (mOut + len < sizeof(mBuffer)) {
      memcpy(&mBuffer[mOut], sp, len);
    }
    mOut += len;
    return true;
  }

 private:
  char mBuffer[100];
  size_t mOut;
};

const char* TestPrintfTarget::test_string = "test string";
```

The following code is the macro expansion for the example above.

```C++
do {
  static_assert(mozilla ::detail ::AssertionConditionType<
                    decltype(mOut == strlen(test_string))>::isValid,
                "invalid assertion condition");
  if ((__builtin_expect(!!(!(!!(mOut == strlen(test_string)))), 0))) 
  {
    do {} 
    while (false);
    AnnotateMozCrashReason(
        "MOZ_RELEASE_ASSERT"
        "("
        "mOut == strlen(test_string)"
        ")");
    do 
    {
      *((volatile int*)__null) = 44;  ///REVIEW NOTE: #define __null NULL exists in another part of the program
      ::abort();
    } 
    while (false);
  }
} while (false)
```

### Null Dereference - Fixing

As this checker is of high severity, and the error is quite noticable, chances are quite big that actual issues are weeded out by sanity checking and running the tests. Hence remaining examples are likely false positives, or even intentional. However, as it is a *core* checker, just surpressing it is not an option.

#### Null Dereference - False Positives

It is possible to annotate the source code with suppression information on a line-by-line basis. But unless some common representation is used, this approach will be tool-specific and add noise to the final source code. For this particular problem it will mean many assertions. Tests assert the state of pointers quite regularily.

The deeper issue is that by not recognizing the assertions, the analysis engine is exploring paths that are infeasible, which in turn may pollute the exploded graph.

The recognition of general assertions is done by the engine. It depends on specific language features being detected. Adding some configuration option to clang-SA would allow end users to provide additional function/macro/method signatures to be interpreted as assertions. This would however make some invasive changes to the analysis core. Adding such checks would also decrease performance for the general case, as this issue *should* be test-specific. In other cases the standard assertions are likely used.

The issue can be mitigated by a dedicated checker to model the assertion specifically. Since checkers are modular and may be run on an opt-in basis, such a checker could then be run as part of analysing test code. This has been done in practice, specifically for GoogleTest. This shifts the effort towards the test framework providers.

Post-processing the analysis to remove such faulty assumptions may also be done. These cases can be detected easily as the framework already states its assumptions. Consider the case where it assumes the condition of a "hard" (halting) assertion is false. If the bug path continues in the same function that contains the assertion, there is a false positive.

#### Null Dereference - Intentional Cases

The intentional case observed is when test asserts force a crash. This commonly triggers a prompt to send a "crash report". This approach may be appropriate for end-user testing, as there may be a vast difference in software and hardware configurations between users. The downside from the point of this work is that this approach will trigger one warning per such assert.

Given that this is intentional behaviour such reports will not provide anything useful, and should be silenced. Doing it manually by e.g., adding a surpression comment would make it more clear that it is intentional. But as previously noted, this will be specific to the analysis tool [https://codechecker.readthedocs.io/en/latest/analyzer/user_guide/#source-code-comments].

Alternatively, we may make the (potentially dangerous) assumption that issues raised as part of an assertion are **intentional**. This may in fact help us avoid a number of similar issues related to e.g., negative testing practices. The reason is that such cases will likely intentionally put the system in a "bad" state, and then verify that this happened inside some assertion.

However it is potentially dangerous in the sense that it may mask a real issue, and should therefore be specific to test assertions.

To specify, assume we have some classification of assertions such that we can retrieve the start and end location (in whichever format is desired). Note that test assertions are commonly defined using macros. Hence the locations in the source code may be most appropriate for the general cases. As an aside, for fixing we should also point the user to the definition of the macro such that it can be updated.

Assume that a bugpath is defined as a list of (location, status) pairs detailing the steps that are taken to reach the bug. Then the final element is the entry that holds the warning report.

To do the pre-filter/marking, we then extract reports where the last entry in the bugpath has a location that is inside an assertion "call". This would allow developers to quickly do targeted investigations of such warnings.

## Clone Checker

Cloning reduces the maintainability of the code. This checker finds potential clones.

While the issue of code clones is not test-specific, there are test-specific use cases.

* Automated software testing frameworks supply common setup and teardown actions that are invoked before/after each test or test suite. Clone detection could be used to find and extract common setup and teardown actions. They could then be moved into their dedicated places.

### Clone Checker - Examples

The example below is an outtake from DOMBasicTests. The clone checker detects duplicated code on the initialization of the doc variable, and it shall also be noted that most of the blocks end with the release method call. A minimal example can be seen in the first test scope that creates a new empty document.

It shall be noted that Xerces DOM-tests uses CMake's test framework. Hence each test suite provides their own entry function. It does some general setup and then calls the (in this case) three test suite functions. Each test suite is then defined as a set of bracketed scopes.

```C++
    /// xerces-c/tests/src/DOM/DOMMemTest/DOMMemTest.cpp:122
    //  Test Doc01      Create a new empty document
    //
    {
        //  First precondition, so that lazily created strings do not appear
        //  as memory leaks.
        DOMDocument*   doc;
        doc = DOMImplementationRegistry::getDOMImplementation(X("Core"))->createDocument();
        doc->release();
    }

    //
    //  Test Doc02      Create one of each kind of node using the
    //                  document createXXX methods.
    //                  Watch for memory leaks.
    //
    {
        //  Do all operations in a preconditioning step, to force the
        //  creation of implementation objects that are set up on first use.
        //  Don't watch for leaks in this block (no  / )
        DOMDocument* doc = DOMImplementationRegistry::getDOMImplementation(X("Core"))->createDocument();
        /// Code Omitted for brevity
        doc->release();
    }



    //
    //  Doc03 - Create a small document tree
    //

    {
        DOMDocument*   doc = DOMImplementationRegistry::getDOMImplementation(X("Core"))->createDocument();
        /// Code Omitted for brevity
        doc->release();
    };
```

## Macro Argument without Paranthesis

Given the popularity of macro-based test frameworks, it is worth repeating that macros expansion may cause subtle bugs if arguments are not properly enclosed with parentheses. If this is not done, assertion macros may result in faulty evaluations due to e.g., operator precedence mismatches.

### Macro Argument without Paranthesis - Examples

As an example, bitwise AND as used to compare myposition and position below has lower precedence than e.g., addition, so care must be taken if such expressions would be used.

```C++
/// xerces-c/tests/src/DOM/DOMTest/DTest.cpp:136
#define COMPARETREEPOSITIONTEST(thisNode, otherNode, position, line) \
    myposition = thisNode->compareDocumentPosition(otherNode); \
    if ((myposition & position) == 0) {\
        fprintf(stderr, "DOMNode::compareDocumentPosition does not work in line %i\n", line); \
        OK = false; \
    }
```

```C++
#define CHECK_NORM(NRM, norm, src) {                                 \
    unsigned char *src_norm = (unsigned char*) utf8proc_ ## NRM((utf8proc_uint8_t*) src);      \
    check(!strcmp((char *) norm, (char *) src_norm),                                  \
          "normalization failed for %s -> %s", src, norm);          \
    free(src_norm);                                                 \
}
```

There are cases where such fixit solutions would not work however. The macros below creates some boilerplate for GTest test cases through macro expansion. In this case wrapping usages of ```test_case_name``` and ```test_name``` in the macro below in parentheses would cause a compilation error. As such, for such cases, some additional context checking in the macro text may be necessary. In general it should be checked if it is part of some typename .

```C++
////opencv/modules/ts/include/opencv2/ts/ts_gtest.h
// Expands to the name of the class that implements the given test.
#define GTEST_TEST_CLASS_NAME_(test_case_name, test_name) \
  test_case_name##_##test_name##_Test
// Helper macro for defining tests.
#define GTEST_TEST_(test_case_name, test_name, parent_class, parent_id)\
class GTEST_TEST_CLASS_NAME_(test_case_name, test_name) : public parent_class {\
 public:\
  GTEST_TEST_CLASS_NAME_(test_case_name, test_name)() {}\
 private:\
  virtual void TestBody();\
  static ::testing::TestInfo* const test_info_ GTEST_ATTRIBUTE_UNUSED_;\
  GTEST_DISALLOW_COPY_AND_ASSIGN_(\
      GTEST_TEST_CLASS_NAME_(test_case_name, test_name));\
};\
\
::testing::TestInfo* const GTEST_TEST_CLASS_NAME_(test_case_name, test_name)\
  ::test_info_ =\
    ::testing::internal::MakeAndRegisterTestInfo(\
        #test_case_name, #test_name, NULL, NULL, \
        ::testing::internal::CodeLocation(__FILE__, __LINE__), \
        (parent_id), \
        parent_class::SetUpTestCase, \
        parent_class::TearDownTestCase, \
        new ::testing::internal::TestFactoryImpl<\
            GTEST_TEST_CLASS_NAME_(test_case_name, test_name)>);\
void GTEST_TEST_CLASS_NAME_(test_case_name, test_name)::TestBody()

#define GTEST_TEST(test_case_name, test_name)\
  GTEST_TEST_(test_case_name, test_name, \
              ::testing::Test, ::testing::internal::GetTestTypeId())

// Define this macro to 1 to omit the definition of TEST(), which
// is a generic name and clashes with some other libraries.
#if !GTEST_DONT_DEFINE_TEST
# define TEST(test_case_name, test_name) GTEST_TEST(test_case_name, test_name)
#endif
```

### Macro Argument without Paranthesis - Fixing

This checker has automatic fix-it option to enclose usages of arguments with parentheses.

Note that as seen below, running fixits as is will cause issues in googleTest usecases.

## Sign/Size Differences

As for-loops that iterate over some C-array are very commonly written with a loop variable using auto or int type, this mismatch happens when some function returns e.g., `size_t` or some other size.

### Sign/Size Differences - Fixing

In general, an explicit typecasting makes it more obvious what is going on. Depending on the usage, rewriting it to use STL iteration using begin and end iterator may help as well. However, it is dependent on how the value is actually used.

### Sign/Size Differences - Examples

In the example below taken from the surge project, k is some class representing a Synthesizer Keyboard, and the test checks that each key maps to its corresponding numerical value, starting from 0. It uses the std::vector from C++'s STL library to store keys. The size method returns an unsigned long int. As the constant 0 is implicitly an integer there is a mismatch between the signs and sizes. Note that in this particular case, rewriting it to use STL's iteration with .begin() and .end() iterators would be counterproductive.

```C++
  auto k = Tunings::KeyboardMapping();
  REQUIRE( k.count == 0 );
  REQUIRE( k.firstMidi == 0 );
  REQUIRE( k.lastMidi == 127 );
  REQUIRE( k.middleNote == 60 );
  REQUIRE( k.tuningConstantNote == 60 );
  REQUIRE( k.tuningFrequency == Approx( 261.62558 ) );
  REQUIRE( k.octaveDegrees == 0 );
  for( auto i=0; i<k.keys.size(); ++i )
      REQUIRE( k.keys[i] == i );

  // REQUIRE( k.rawText == "" );
   }
```

## Invalidated iterator Accessed

**Note: Checker is currently in alpha stage**.

**Note: Not test-specific. May be good to be aware in case iterators are used a lot in tests, e.g., when adding removing elements.**

The checker catches occurrences of STL iterators that have been invalidated, and then accessed.

The issue seems to be that it may lead to them pointing to invalid memory. If the elements are not moved/updated accordingly such that the iterator position still points to a valid element. This seems to be STL-type specific, as some (such as std::vector) are specified to ensure continuity. std::vector for instance will move the elements such that the iterator simply points to the element after the last erased one in the array.

### Invalidated iterator Accessed - Examples

In the example below, the checker reports that the result of an iterator that has been erased using std::vector::erase in the previous iteration has been invalidated. Based on <https://www.cplusplus.com/reference/vector/vector/erase/> "*Iterators, pointers and references pointing to position (or first) and beyond are invalidated*" the iterator is indeed invalidated as a result of the erase call. Based on a quick test it does not seem to crash (which is supported by std::vector documentation that subsequent elements are moved), but it might be problematic when running low on memory etc.

```C++
////mozilla-central/dom/media/webrtc/transport/test/ice_unittest.cpp:737
std::vector<std::string> attributes = remote->GetAttributes(i);

        for (auto it = attributes.begin(); it != attributes.end();) {
          if (trickle_mode == TRICKLE_SIMULATE &&
              it->find("candidate:") != std::string::npos) {
            std::cerr << name_ << " Deferring remote candidate: " << *it
                      << std::endl;
            attributes.erase(it);
          } else {
            std::cerr << name_ << " Adding remote attribute: " + *it
                      << std::endl;
            ++it;
          }
        }
```

### Invalidated iterator Accessed - Fixing

**Investigate**: erase for instance is specified to return the iterator to the next element, or .end() if it was the last to be removed. Can we simply reassign the iterator to the result of erase? Erase works differently depending on which STL type it is, how is the invalidation modelled?
