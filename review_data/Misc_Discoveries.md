# Random Discoveries in OSS

In this document we gather some discoveries found along the way.

They are not necessarily related to the work done, but may be of interest for future studies.

## Unused Local variables

Seems to be taken care of in a number of ways. Note that we do not refer to function parameters (which may instead be unnamed to show they're not used).

The examples below are some cases seen in open source solutions. Something to investigate might be performance/suitability issues between the different examples.
```C++
#define UNUSED_VOID(expr) ((void)(expr))
#define UNUSED(expr) do { (void)(expr); } while (0)
#define UNUSED_IF(x) { if(x!=0){} }
```

## Incomplete Type checking

The sizeof operator is used to check whether a type is complete or not, by checking if they have a non-zero size. However, this may not work as intended if the type itself is a pointer, as pointers to incomplete types still have a size. Thus, if the T below is a pointer type, the check will not work as intended.

```C++
  void reset(T* p = NULL) {
    if (p != ptr_) {
      if (IsTrue(sizeof(T) > 0)) {  // Makes sure T is a complete type.
        delete ptr_;
      }
      ptr_ = p;
    }
  }
```

```C++
#include <iostream>
using namespace std;
struct incom;
int main()
{
    printf("Hello World");
    struct incom * ptr;
    // compilation error as struct incom is incomplete
    //cout << "incomplete type size: " << sizeof(struct incom) << "\n";
    cout << "incomplete pointer size: " << sizeof(struct incom*) << "\n";
    
    return 0;
}
```

