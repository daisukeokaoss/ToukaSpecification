# Touka Specification
Specitication of Testing framework that ensure if two C language function input is same then output must be same.Touka means equivalent by Japanese.And this is Japanese ordinary woman name.

### Motivation - To Port Intel intrinsic function to OpenPOWER
 Motivation to this project is trial to port Intel intrinsic function to OpenPOWER.This is one of CPU platform dependency problem.
 Original document is this.

 https://openpowerfoundation.org/specifications/vectorintrinsicportingguide/

 Porting process descrived in this document uses specific wrapper structure.Like shown below.

 ```c
extern __inline __m128d __attribute__((__gnu_inline__, __always_inline__,__artificial__))
_mm_add_pd (__m128d __A, __m128d __B)
{
   return (__m128d) ((__v2df)__A + (__v2df)__B);
}
```

another example of wrapper structure of porting is like below.

```c
extern __inline  __m128d  __attribute__((__gnu_inline__, __always_inline__, __artificial__))
_mm_cmpeq_sd(__m128d  __A, __m128d  __B)
{
  __v2df __a, __b, __c;
  /* PowerISA VSX does not allow partial (for just lower double)
     results. So to insure we don't generate spurious exceptions
     (from the upper double values) we splat the lower double
     before we do the operation. */
  __a = vec_splats (__A[0]);
  __b = vec_splats (__B[0]);
  __c = (__v2df) vec_cmpeq(__a, __b);
  /* Then we merge the lower double result with the original upper
     double from __A.  */
  return (__m128d) _mm_setr_pd (__c[0], __A[1]);
}
```

To make wrapper structure,we have to have detail understanding of Intel x86 ISA and OpenPOWER ISA.But one question arises.  

This wrapper structure is truly correct?  

To make sure wrapper structure is correct.We have to make testing framework Touka.

### Touka mascot  
  
<img width="256" height="256" alt="Touka" src="Touka.png" />  
  
This is the image of mascot of Testing Framwework Touka.Imaging Japanese Kimono(Japanese traditional dress) and Japanese animal musasabi.
