# To record exception
When creating the test framework Touka, we need a strategy to record exceptions in the output of the test framework when an intrinsic operation throws an exception.
I will discuss it here.  
"Intrinsic operations" involve throwing values ​​into Intel intrinsics and their ports to OpenPOWER and RISC-V, primarily performing SIMD operations.
The core of this is called the "intrinsic operations unit".  
The intrinsic calculation unit is written so that Intel Intrinsics are executed on the Intel side, and the following wrapper structure is written on the OpenPOWER side.
```c
extern __inline __m128d __attribute__((__gnu_inline__, __always_inline__,__artificial__))
_mm_add_pd (__m128d __A, __m128d __B)
{
  return (__m128d) ((__v2df)__A + (__v2df)__B);
}
```
There is no syntax equivalent to try catch in C. If the test framework is created monolithically with the intrinsic operations part, the application will stop at the point where an exception is thrown.
As a software, one option is to separate the intrinsic operations part from the other parts of the Touka test framework.
An interface between the intrinsic operations part and the other parts of Touka is required.
