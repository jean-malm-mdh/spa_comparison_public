# Review Process Documentation
1) In CodeChecker server Report tab for the product, 
	1) Filtered to unique reports
	2) Filtered to Medium, High, Unknown severity

## Manually removed reports
* llvm/ADT/SmallVector.h
	Not technically test code, caused 700 unspecified warnings related to template class with non-initialized member variable.

* mozilla-central/toolkit/crashreporter/google-breakpad/src/common/scoped_ptr.h
	Not test code, 12 cases of non-const parameters that may be declared const