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

| Num Of   |                    Ports pressure in cycles                         |      |
|  Uops    |  0  - DV    |  1   |  2  -  D    |  3  -  D    |  4   |  5   |  6   |  7   |
-----------------------------------------------------------------------------------------
|   1      |             |      | 1.0     1.0 |             |      |      |      |      | movsd xmm0, qword ptr [rsi]
|   1      |             |      |             | 1.0     1.0 |      |      |      |      | movsd xmm1, qword ptr [rsi+0x8]
|   1      |             | 1.0  |             |             |      |      |      |      | mulsd xmm0, xmm0
|   1      |             | 1.0  |             |             |      |      |      |      | mulsd xmm1, xmm1
|   1      | 0.5         | 0.5  |             |             |      |      |      |      | addsd xmm0, xmm1
|   1      | 1.0         |      |             |             |      |      |      |      | ucomisd xmm2, xmm0
|   1      | 1.0     6.0 |      |             |             |      |      |      |      | sqrtsd xmm1, xmm0
|   1      |             |      |             |             |      |      | 1.0  |      | jnbe 0x3d
|   1      | 1.0         |      |             |             |      |      |      |      | ucomisd xmm1, xmm2
|   1      |             |      |             |             |      |      | 1.0  |      | jbe 0x20
|   1*     |             |      |             |             |      |      |      |      | movapd xmm0, xmm3
|   1      | 1.0     4.0 |      |             |             |      |      |      |      | divsd xmm0, xmm1
|   1      |             |      | 1.0     1.0 |             |      |      |      |      | movsd xmm1, qword ptr [rsi]
|   1      |             | 1.0  |             |             |      |      |      |      | mulsd xmm1, xmm0
|   2^     |             | 1.0  |             | 1.0     1.0 |      |      |      |      | mulsd xmm0, qword ptr [rsi+0x8]
|   2^     |             |      |             |             | 1.0  |      |      | 1.0  | movsd qword ptr [rsi], xmm1
|   2^     |             |      |             |             | 1.0  |      |      | 1.0  | movsd qword ptr [rsi+0x8], xmm0
|   1      |             |      |             |             |      | 1.0  |      |      | add rsi, 0x10
|   1*     |             |      |             |             |      |      |      |      | cmp rsi, rbp
|   0*F    |             |      |             |             |      |      |      |      | jnz 0xffffffffffffffae
```


```
div_length

| Num Of   |                    Ports pressure in cycles                         |      |
|  Uops    |  0  - DV    |  1   |  2  -  D    |  3  -  D    |  4   |  5   |  6   |  7   |
-----------------------------------------------------------------------------------------
|   1      |             |      | 1.0     1.0 |             |      |      |      |      | movsd xmm0, qword ptr [rsi]
|   1      |             |      |             | 1.0     1.0 |      |      |      |      | movsd xmm1, qword ptr [rsi+0x8]
|   1      |             | 1.0  |             |             |      |      |      |      | mulsd xmm0, xmm0
|   1      |             | 1.0  |             |             |      |      |      |      | mulsd xmm1, xmm1
|   1      |             | 1.0  |             |             |      |      |      |      | addsd xmm0, xmm1
|   1      | 1.0         |      |             |             |      |      |      |      | ucomisd xmm2, xmm0
|   1      | 1.0     6.0 |      |             |             |      |      |      |      | sqrtsd xmm1, xmm0
|   1      |             |      |             |             |      |      | 1.0  |      | jnbe 0x39
|   1      | 1.0         |      |             |             |      |      |      |      | ucomisd xmm1, xmm2
|   1      |             |      |             |             |      |      | 1.0  |      | jbe 0x1c
|   1      |             |      | 1.0     1.0 |             |      |      |      |      | movsd xmm0, qword ptr [rsi]
|   1      | 1.0     4.0 |      |             |             |      |      |      |      | divsd xmm0, xmm1
|   2^     |             |      |             |             | 1.0  |      |      | 1.0  | movsd qword ptr [rsi], xmm0
|   1      |             |      |             | 1.0     1.0 |      |      |      |      | movsd xmm0, qword ptr [rsi+0x8]
|   1      | 1.0     4.0 |      |             |             |      |      |      |      | divsd xmm0, xmm1
|   2^     |             |      |             |             | 1.0  |      |      | 1.0  | movsd qword ptr [rsi+0x8], xmm0
|   1      |             |      |             |             |      | 1.0  |      |      | add rsi, 0x10
|   1*     |             |      |             |             |      |      |      |      | cmp rsi, rbp
|   0*F    |             |      |             |             |      |      |      |      | jnz 0xffffffffffffffb2
```
[fog] http://www.agner.org/optimize/instruction_tables.pdf
