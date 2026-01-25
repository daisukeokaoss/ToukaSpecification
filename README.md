# ToukaSpecification
Specitication of Testing framework that ensure if two C language function input is same then output must be same.

### Motivation - To Port Intel intrinsic function to OpenPOWER
 Motivation to this project is trial to port Intel intrinsic function to OpenPOWER.
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
