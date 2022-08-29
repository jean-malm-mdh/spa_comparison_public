# Clang Tidy

## False Positives
### Infinite Loop
```C++
already_AddRefed<nsIThread> CreateTestThread(const char* aName,
                                             Monitor& aMonitor, bool& aDone);
/// Note the bool& of aDone
Monitor monitor("TestCrashThreadAnnotation");
  bool threadNameSet = false;
  nsCOMPtr<nsIThread> thread =
      CreateTestThread("Thread1", monitor, threadNameSet); /// <-- threadNameSet passed by reference
  ASSERT_TRUE(!!thread);

  {
    MonitorAutoLock lock(monitor);
    while (!threadNameSet) {   /// Would be good if we could detect such cases. mark with volatile for now?
      monitor.Wait();
    }
  }
```

## Improvement Potential
### Incorrect Roundings
For test code, symbolic constants are typically used. Saw a lot of integer divisions of image height/width. A bit expensive solution but we could do some constant propagation and then check if result is "safe".

### Multiple-Statement-Macro
Difficult to determine validity without the macro context. Clang-Tidy should add it as a note(?)

### Signed-Char-Misuse
Lots of occurences in templates - check if the warning is triggered on a template implementation?

### cert-err33-c
To make it easier to filter, output what function is being called in the actual warning. Some functions are more critical to check for potential error off, in my opinion.

### clang-diagnostic-unused-parameter
To my knowledge, this can be hidden by simply commenting out the name of the variable in the argument list.
Some heuristic would be good to have to detect cases where this is intended, e.g., if the function is not implemented or a test mock.

### clang-diagnostic-init-variables
Basic values have fixes that suggest setting it to e.g., (0, NAN, nullptr). They can probably be used immediately. One simple case is variables that are first declared and then initialized immediately in the next statement. Fix-it would be to simply combine it.