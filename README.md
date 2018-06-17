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

The Intel architecture code analyser, iaca for short, is a tool that visualizes the mapping from instructions to execution ports and port utilization[iaca]. When run on the two kernel above the following is produced:

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


## Benchmark

The Intel architecture code analyzer is a static tool that analyzes the binary, but doesn't actually run it. To get actual performance numbers we should run the code. Here I'm using the Google benchmarking suite[bench], running on an i7-7820X CPU @ 4.0 GHz.

```
mul_inv

---------------------------------------------------------
Benchmark                  Time           CPU Iterations
---------------------------------------------------------
BM_normalize/8            21 ns         21 ns   32638163
BM_normalize/16           44 ns         44 ns   15942537
BM_normalize/32          101 ns        101 ns    6913534
BM_normalize/64          202 ns        202 ns    3464931
BM_normalize/128         407 ns        407 ns    1718113
BM_normalize/256         811 ns        811 ns     863501
BM_normalize/512        1616 ns       1616 ns     432976
BM_normalize/1024       3225 ns       3225 ns     217184
BM_normalize/2048       6492 ns       6492 ns     107797
```

```
div_length

---------------------------------------------------------
Benchmark                  Time           CPU Iterations
---------------------------------------------------------
BM_normalize/8            28 ns         28 ns   24740613
BM_normalize/16           57 ns         57 ns   12407197
BM_normalize/32          137 ns        137 ns    5118732
BM_normalize/64          259 ns        259 ns    2702066
BM_normalize/128         521 ns        521 ns    1344022
BM_normalize/256        1037 ns       1037 ns     675255
BM_normalize/512        2071 ns       2071 ns     337887
BM_normalize/1024       4132 ns       4133 ns     169377
BM_normalize/2048       8336 ns       8336 ns      83988
```

Divinding once and multiplying twice is indeed faster. Dividing the times by the number of elements gives about 3 ns for the `mul_inv` version and about 4 ns for the `div_length` version. At 4.0 GHz that corresponds to about 13 cycles for the `mul_inv` version and 16 cycles for the `div_length` version. That's fairly close to the cycle estimate given by the iaca tool.


## References

[fog] http://www.agner.org/optimize/instruction_tables.pdf  
[iaca] https://software.intel.com/en-us/articles/intel-architecture-code-analyzer
[bench] https://github.com/google/benchmark
