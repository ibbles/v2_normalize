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

It is commonly known that multiplication is faster than division for floating point values. Turning to Agner Fog's instruction tables[fog] we read that on Intel Skylake MULSD, MULtiply Single Double precision, has a dependency latency of 4 cycles, an issue latency of 1 cycle (or less) and can go to one of two execution ports. DIVSD, on the other hand, has 13-14 cycles of dependency latency, 4 cycles of issue latency and only one compatible execution port.


[fog] http://www.agner.org/optimize/instruction_tables.pdf
