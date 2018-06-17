# v2_normalize
A performance analysis of v2 normalization.

Here, v2 is the name of a two-dimensional double precision vector implemented as follows:

```c++
struct v2 {
    double x;
    double y;
};
```

Normalization is the act of altering `x` and `y` so that the length of the vector, `sqrt(x*x + y*y)`, equals 1.0. This is achieved by dividing both `x` and `y` by that length.
