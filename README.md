# v2_normalize
A performance analysis of v2 normalization.

## Introduction

Here, v2 is the name of a two-dimensional double precision vector implemented as follows:

```c++
struct v2 {
    double x;
    double y;
};
```

Normalization is the act of altering `x` and `y` so that the length of the vector, `sqrt(x*x + y*y)`, becomes 1.0. This is achieved by dividing both `x` and `y` by the current length.

## Multiplication vs division

It is commonly known that multiplication is faster than division for floating point values. Turning to Agner Fog's instruction tables[fog] we read that on Intel Skylake MULSD, MULtiply Single Double precision, has a dependency latency of 4 cycles, an issue latency of 1 cycle (or less) and can go to one of two execution ports. DIVSD, on the other hand, has 13-14 cycles of dependency latency, 4 cycles of issue latency and only one compatible execution port.

Since a v2 normalizaion requires two divisions by the same value, the vector length, it is tempting to compute the inverse of the length and perform two multiplications instead. Compare the following two implementations of a normalization kernel:


## Kernel implementations

```c++
void normalize_mul_inv(int N, v2* d) {
    for (int i = 0; i < N; ++i) {
        v2* v = d + i;
        double length = sqrt(v->x * v->x + v->y * v->y);
        if (length > 0.0) {
            double inv = 1.0 / length;
            v->x *= inv;
            v->y *= inv;
        }
    }
}


void normalize_div_length(int N, v2* d) {
    for (int i = 0; i < N; ++i) {
        v2_dd* v = d + i;
        double length = sqrt(v->x * v->x + v->y * v->y);
        if (length > 0.0) {
            v->x /= length;
            v->y /= length;
        }
    }
}
```

Which one of these can we expect to be faster? For now we assume that the loop overhead and the length calculation are identical in both versions, and that the vetor length is never zero.

## Cycle estimation

The interesting part performed by the `mul_inv` version is then a divide and two multiplications that depend on the divide but not on each other. As stated in the introduction, the divide has a dependency latency of 13-14 cycles and the multiplication as two execution ports and can therefore be done in parallel, with an issue latency of 1 cycle. The total becomes 14-15 cycles.

The interesting part performed by the `div_length` version is simply two divides. There is no dependency between them, but there is only one execution port for divides, so we must wait 4 cycles of issue latency before the second can start. The total becomes 8 cyles since there are here no dependencies on the result of the divides.

## Intel architecture code analyzer

The Intel architecture code analyser, iaca for short, is a tool that visualizes the mapping from instructions to execution ports and port utilization. When run on the two kernel above the following is produced:

```
mul_inv

Block Throughput: 9.95 Cycles       Throughput Bottleneck: Backend
Loop Count:  22
Port Binding In Cycles Per Iteration:
--------------------------------------------------------------------------------------------------
|  Port  |   0   -  DV   |   1   |   2   -  D    |   3   -  D    |   4   |   5   |   6   |   7   |
--------------------------------------------------------------------------------------------------
| Cycles |  4.5    10.0  |  4.5  |  2.0     2.0  |  2.0     2.0  |  2.0  |  1.0  |  2.0  |  2.0  |
--------------------------------------------------------------------------------------------------
```

```
div_length

Block Throughput: 14.00 Cycles       Throughput Bottleneck: Backend
Loop Count:  22
Port Binding In Cycles Per Iteration:
--------------------------------------------------------------------------------------------------
|  Port  |   0   -  DV   |   1   |   2   -  D    |   3   -  D    |   4   |   5   |   6   |   7   |
--------------------------------------------------------------------------------------------------
| Cycles |  5.0    14.0  |  3.0  |  2.0     2.0  |  2.0     2.0  |  2.0  |  1.0  |  2.0  |  2.0  |
--------------------------------------------------------------------------------------------------
```

This tells us that multiplying by the inverse of the length is faster than dividing by the length directly, and that it is Port 0, the divider pipeline, that is the bottleneck in the `div_length` case since the block throughput is the same as the Port 0 utilization.

## References

[fog] http://www.agner.org/optimize/instruction_tables.pdf
