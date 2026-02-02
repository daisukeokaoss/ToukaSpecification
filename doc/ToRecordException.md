# To record exception
When creating the test framework Touka, we need a strategy to record exceptions in the output of the test framework when an intrinsic operation throws an exception.
I will discuss it here.  
"Intrinsic operations" involve throwing values ​​into Intel intrinsics and their ports to OpenPOWER and RISC-V, primarily performing SIMD operations.
