# CppCheck Issues
## Preprocessor error combinations
Despite CppCheck setting the preprocessor definitions based on the compile command, there are still issues when bad combinations of preprocessor defines are used. 

In the example below, -D__STDC_CONSTANT_MACROS is set, however the error is still triggered at every inclusion of the file.
```C++
// https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm-c/DataTypes.h:37
#if !defined(UINT32_C)

# error "The standard header <cstdint> is not C++11 compliant. Must #define "\

"__STDC_CONSTANT_MACROS before #including llvm-c/DataTypes.h"

#endif
```
### Mitigation
Approaches that have been tried are to extend the ctu depth, setting the -DUINT32_C flag manually, with no effect. 

As this check does not seem to provide anything useful, the entire preprocessor check was temporarily disabled for the purpose of the analysis. This is somewhat of a "nuclear" option however, and does not scale well given it would potentially have to be done to every "#error" preprocessor.
[https://cppcheck.sourceforge.io/manual.pdf] 

## Unknown Macros
In some cases CppCheck cannot find headers that are given to define macros.

## System Version Mismatch
A somewhat context-dependent example. CppCheck has a number of common configurations for well-known packages. In some cases that information can cause noisy warnings if the versions do not match. Consider the case below:
```C++
///subsurface/tests/testgitstorage.cpp:100
#if QT_VERSION >= QT_VERSION_CHECK(5, 10, 0)
		email = QString("gitstorage%1@hohndel.org").arg(QRandomGenerator::global()->bounded(10));
#else
		// on Qt 5.9 we go back to using qsrand()/qrand()
		qsrand(time(NULL));
		email = QString("gitstorage%1@hohndel.org").arg(qrand() % 10); /// <-- Obsolete function 'qrand' called. It is recommended to use 'QRandomGenerator' instead.
#endif
```
The warning is given due to a version mismatch between the configuration analysed and the configuration file. Based on context, QT Version 5.9 did not yet have `QRandomGenerator` and this is `#ifdef'd` out. Hence there is an implicit dependency between the `#define QT_VERSION` value and this particular check.

### Mitigation
This is somewhat specific to our setup. In a real system the software will likely provide a well-defined set of defines to run the tools with. 

This may occur in testing of systems with large amounts of variants. It could be handled by introducing a dedicated dependency check between parts of the configuration, similar to how Clang checks language versions before determining if the check should be run. As a concrete example, the case above could be avoided if the check required QT_VERSION to be higher than some value.

## Inability to Recognize Test Fixtures
CppCheck in general has good support for different test frameworks. In some cases however it throws warnings that are clear false positives.
* uninitDerivedMemberVar is raised on GTest class fields when the variable is initialized in the SetUp function. In some cases the SetUp is done in a base class, and the test fixture is of a derived class.
```C++
class TableTest : public ::testing::Test {
 protected:

  virtual void SetUp() {
    ots::FontFile *file = new ots::FontFile();
    file->context = new ots::OTSContext();
    font = new ots::Font(file);
  }

  virtual void TearDown() {
    delete font->file->context;
    delete font->file;
    delete font;
  }

  TestStream out;
  ots::Font *font;
};
class ScriptListTableTest : public TableTest { };
/// All the test with fixtures below cause a warning for ots::Font *font not being defined.
TEST_F(ScriptListTableTest, TestScriptRecordOffsetUnderflow) {
  BuildFakeScriptListTable(&out, 1, 1, 1);
  // Set bad offset to ScriptRecord[0].
  out.Seek(6);
  out.WriteU16(0);
  EXPECT_FALSE(ots::ParseScriptListTable(font, out.data(), out.size(), 1));
}

TEST_F(ScriptListTableTest, TestScriptRecordOffsetOverflow) {
  BuildFakeScriptListTable(&out, 1, 1, 1);
  // Set bad offset to ScriptRecord[0].
  out.Seek(6);
  out.WriteU16(out.size());
  EXPECT_FALSE(ots::ParseScriptListTable(font, out.data(), out.size(), 1));
}

TEST_F(ScriptListTableTest, TestBadLangSysCount) {
  BuildFakeScriptListTable(&out, 1, 1, 1);
  // Set too large langsys count.
  out.Seek(10);
  out.WriteU16(2);
  EXPECT_FALSE(ots::ParseScriptListTable(font, out.data(), out.size(), 1));
}

TEST_F(ScriptListTableTest, TestLangSysRecordOffsetUnderflow) {
  BuildFakeScriptListTable(&out, 1, 1, 1);
  // Set bad offset to LangSysRecord[0].
  out.Seek(16);
  out.WriteU16(0);
  EXPECT_FALSE(ots::ParseScriptListTable(font, out.data(), out.size(), 1));
}
```

## Misc Issues
The following seems to be blatantly false ...
```C++
std::vector<int*> v1{ new int(1), new int(2), new int(3) };
    std::vector<int*> v1_ref = v1; ///<-- Assignment v1_ref=v1, assigned value is size=0
```
Cppcheck is ignoring posix::Abort():
```C++
const TestPartResult& TestResult::GetTestPartResult(int i) const {
  if (i < 0 || i >= total_part_count()) ///<-- Assuming i<0 is not redundant
    internal::posix::Abort();
  return test_part_results_.at(i); /// <-- invalid argument. Cppcheck assumes it can be -1 ...
}
```
mismatchingContainerExpression should likely not trigger on equality check. E.g., if checking preconditions
```C++
/// Check pointer identity (not value) of identifier and data.
  friend bool operator==(const MemoryBufferRef &LHS,
                         const MemoryBufferRef &RHS) {
    return LHS.Buffer.begin() == RHS.Buffer.begin() && // <-- Seems reasonable way to do it.
```
Constructor that calls only superclass constructor (empty body) still gives warning for noReturn
=> If function body is completely empty, this is probably intended and can be surpressed. If unsure, restrict it to constructor specifically.

Memory/Resource leaks warned about:
```C++
FILE* f = fopen("street.mp4", "rb");
  ASSERT_TRUE(f != nullptr); ///<-- resource leak f (False, since it is closed later)
  // Read just the moov header to work around the parser
  // treating mid-box eof as an error.
  // read_vector reader = read_vector(f, 1061);
  struct stat s;
  ASSERT_EQ(0, fstat(fileno(f), &s));
  read_vector reader = read_vector(f, s.st_size);
  fclose(f);
```
## Knowledge Database Improvements
* Ability to tell if functions have sideffects. LLVM coding guideline will cause a number of these warnings. Should be able to tell that back(), empty() or other iterator functions are side-effect free.
```C++
  assert(Constraints.empty() || R.size() == Constraints.back().size());
  /// < Assert statement calls a function which may have desired side effects: 'back'.
```
* Ability to set conditional restrictions on \#define flags so that some warnings are not raised, e.g., deprecated versions of functions when building a backwards-compatible version.
	```C++
///subsurface/tests/testgitstorage.cpp:100
#if QT_VERSION >= QT_VERSION_CHECK(5, 10, 0)
		email = QString("gitstorage%1@hohndel.org").arg(QRandomGenerator::global()->bounded(10));
#else
		// on Qt 5.9 we go back to using qsrand()/qrand()
		qsrand(time(NULL));
		email = QString("gitstorage%1@hohndel.org").arg(qrand() % 10); /// <-- Obsolete function 'qrand' called. It is recommended to use 'QRandomGenerator' instead.
#endif
```
* LLVM Support
	* llvm_unreachable => mark as noreturn - it throws a number of missingReturn warnings currently
* Mozilla
	* MOZ_CRASH causes several issues - Should be marked with noreturn and ignore blatant dereference of nullptr
* Test Improvements
	* Cluster warnings raised within asserts. The following raises a lot of "unnecessary string comparison" but it is obviously test-related
	```C++
	  // Test cmSystemTools::strverscmp
  cmAssert(cmSystemTools::strverscmp("", "") == 0, "strverscmp empty string");
  cmAssert(cmSystemTools::strverscmp("abc", "") > 0,
           "strverscmp string vs empty string");
  cmAssert(cmSystemTools::strverscmp("abc", "abc") == 0,
           "strverscmp same string");
  cmAssert(cmSystemTools::strverscmp("abd", "abc") > 0,
           "strverscmp character string");
  cmAssert(cmSystemTools::strverscmp("abc", "abd") < 0,
           "strverscmp symmetric");
  cmAssert(cmSystemTools::strverscmp("12345", "12344") > 0,
           "strverscmp natural numbers");
```