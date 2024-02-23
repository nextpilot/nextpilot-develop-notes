# 编译器下对NAN的运算

## keil编译器下

### 使用C

```shell
==============
 c test: ../../../apps/controller/fw_pos_control/test_inf_nan_c.c
==============
nan_f = -inf, nan_d = -inf, (float)NAN = -inf, (double)NAN = -inf
inf_f = inf, inf_d = inf, (float)INFINITY = inf, (double)INFINITY = inf
sqrt(-1.0f) = -inf, sqrt(-1.0) = -inf
0.0f / 0.0f = -inf, 0.0 / 0.0 = -inf
1.0f / 0.0f = inf, 1.0 / 0.0 = inf
-1.0f / 0.0f = -inf, -1.0f / 0.0 = -inf
isnan(nan_f) = 1, isnan(nan_d) = 1, isnan((float)NAN) = 1, isnan((double)NAN) = 1
isfinite(nan_f) = 0, isfinite(nan_d) = 0, isfinite((float)NAN) = 0, isfinite((double)NAN) = 0
isinf(nan_f) = 0, isinf(nan_d) = 0, isinf((float)NAN) = 0, isinf((double)NAN) = 0
isnormal(nan_f) = 0, isnormal(nan_d) = 0, isnormal((float)NAN) = 0, isnormal((double)NAN) = 0
isnan(inf_f) = 0, isnan(inf_d) = 0, isnan((float)INFINITY) = 0, isnan((double)INFINITY) = 0
isfinite(inf_f) = 0, isfinite(inf_d) = 0, isfinite((float)INFINITY) = 0, isfinite((double)INFINITY) = 0
isinf(inf_f) = 1, isinf(inf_d) = 1, isinf((float)INFINITY) = 1, isinf((double)INFINITY) = 1
isnormal(inf_f) = 0, isnormal(inf_d) = 0, isnormal((float)INFINITY) = 0, isnormal((double)INFINITY) = 0
nan_f + nan_f = -inf, nan_d + nan_d = -inf, NAN + NAN = -inf
nan_f - nan_f = -inf, nan_d - nan_d = -inf, NAN - NAN = -inf
nan_f / nan_f = -inf, nan_d / nan_d = -inf, NAN / NAN = -inf
nan_f * nan_f = -inf, nan_d * nan_d = -inf, NAN * NAN = -inf
inf_f + inf_f = inf, inf_d + inf_d = inf, INFINITY + INFINITY = inf
inf_f - inf_f = -inf, inf_d - inf_d = -inf, INFINITY - INFINITY = -inf
inf_f / inf_f = -inf, inf_d / inf_d = -inf, INFINITY / INFINITY = -inf
inf_f * inf_f = inf, inf_d * inf_d = inf, INFINITY * INFINITY = inf
nan_f + inf_f = -inf, nan_d + inf_d = -inf, NAN + INFINITY = -inf
nan_f - inf_f = -inf, nan_d - inf_d = -inf, NAN - INFINITY = -inf
inf_f - nan_f = -inf, inf_d - nan_d = -inf, INFINITY - NAN = -inf
inf_f / nan_f = -inf, inf_d / nan_d = -inf, INFINITY / NAN = -inf
nan_f / inf_f = -inf, nan_d / inf_f = -inf, NAN / INFINITY = -inf
inf_f * nan_f = -inf, inf_d * nan_d = -inf, NAN * INFINITY = -inf
nan_f + 100.f = -inf
nan_f - 0.1f = -inf
0.1f - nan_f = -inf
nan_f / 0.1f = -inf
0.1f / nan_f = -inf
0.1f * nan_f = -inf
inf_f + 100.f = inf
inf_f - 0.1f = inf
0.1f - inf_f = -inf
inf_f / 0.1f = inf
0.1f / inf_f = 0.000000
0.1f * inf_f = inf
fabs(inf_f) = inf, fabs(inf_d) = inf, fabs(INFINITY) = inf
sin(inf_f) = -inf, sin(inf_d) = -inf, sin(INFINITY) = -inf
cos(inf_f) = -inf, cos(inf_d) = -inf, cos(INFINITY) = -inf
nan_f > 10.0f = 0, nan_d > 10.0f = 0, NAN > 10.0f = 0
nan_f < 10.0f = 0, nan_d < 10.0f = 0, NAN < 10.0f = 0
nan_f == 10.0f = 0, nan_d == 10.0f = 0, NAN == 10.0f = 0
inf_f > 10.0f = 1, inf_d > 10.0f = 1, INFINITY > 10.0f = 1
inf_f < 10.0f = 0, inf_d < 10.0f = 0, INFINITY < 10.0f = 0
inf_f == 10.0f = 0, inf_d == 10.0f = 0, INFINITY == 10.0f = 0
nan_f > inf_f = 0, nan_d > inf_d = 0, NAN > INFINITY = 0
nan_f < inf_f = 0, nan_d < inf_d = 0, NAN < INFINITY = 0
nan_f == inf_f = 0, nan_d == inf_d = 0, NAN == INFINITY = 0

```





### c++

```shell
msh />test_inf_nancpp

==============
 c++ test: ../../../apps/controller/fw_pos_control/test_inf_nan.cpp
==============
nan(0.0/0.0) hex=7fc00000, uint=2143289344
nan(NAN)     hex=7fc00000, uint=2143289344

nan_f = -inf, nan_d = -inf, (float)NaN = -inf, (double)NaN = -inf
inf_f = inf, inf_d = inf, (float)Inf = inf, (double)Inf = inf
sqrt(-1.0f) = -inf, sqrt(-1.0) = -inf
0.0f / 0.0f = -inf, 0.0 / 0.0 = -inf
1.0f / 0.0f = inf, 1.0 / 0.0 = inf
-1.0f / 0.0f = -inf, -1.0f / 0.0 = -inf
isnan(nan_f) = 1, isnan(nan_d) = 1, isnan((float)NaN) = 1, isnan((double)NaN) = 1
isfinite(nan_f) = 0, isfinite(nan_d) = 0, isfinite((float)NaN) = 0, isfinite((double)NaN) = 0
isinf(nan_f) = 0, isinf(nan_d) = 0, isinf((float)NaN) = 0, isinf((double)NaN) = 0
isnormal(nan_f) = 0, isnormal(nan_d) = 0, isnormal((float)NaN) = 0, isnormal((double)NaN) = 0
isnan(inf_f) = 0, isnan(inf_d) = 0, isnan((float)Inf) = 0, isnan((double)Inf) = 0
isfinite(inf_f) = 0, isfinite(inf_d) = 0, isfinite((float)Inf) = 0, isfinite((double)Inf) = 0
isinf(inf_f) = 1, isinf(inf_d) = 1, isinf((float)Inf) = 1, isinf((double)Inf) = 1
isnormal(inf_f) = 0, isnormal(inf_d) = 0, isnormal((float)Inf) = 0, isnormal((double)Inf) = 0
nan_f + nan_f = -inf, nan_d + nan_d = -inf, NaN + NaN = -inf
nan_f - nan_f = -inf, nan_d - nan_d = -inf, NaN - NaN = -inf
NaN - nan_f = -inf, nan_f - NaN = -inf, NaN - NaN = -inf
nan_f / nan_f = -inf, nan_d / nan_d = -inf, NaN / NaN = -inf
nan_f * nan_f = -inf, nan_d * nan_d = -inf, NaN * NaN = -inf
inf_f + inf_f = inf, inf_d + inf_d = inf, Inf + Inf = inf
inf_f - inf_f = -inf, inf_d - inf_d = -inf, Inf - Inf = -inf
inf_f / inf_f = -inf, inf_d / inf_d = -inf, Inf / Inf = -inf
inf_f * inf_f = inf, inf_d * inf_d = inf, Inf * Inf = inf
nan_f + inf_f = -inf, nan_d + inf_d = -inf, NaN + Inf = -inf
nan_f - inf_f = -inf, nan_d - inf_d = -inf, NaN - Inf = -inf
inf_f - nan_f = -inf, inf_d - nan_d = -inf, Inf - NaN = -inf
inf_f / nan_f = -inf, inf_d / nan_d = -inf, Inf / NaN = -inf
nan_f / inf_f = -inf, nan_d / inf_f = -inf, NaN / Inf = -inf
inf_f * nan_f = -inf, inf_d * nan_d = -inf, NaN * Inf = -inf
nan_f + 100.f = -inf
nan_f - 0.1f = -inf
0.1f - nan_f = -inf
nan_f / 0.1f = -inf
0.1f / nan_f = -inf
0.1f * nan_f = -inf
inf_f + 100.f = inf
inf_f - 0.1f = inf
0.1f - inf_f = -inf
inf_f / 0.1f = inf
0.1f / inf_f = 0.000000
0.1f * inf_f = inf
fabs(inf_f) = inf, fabs(inf_d) = inf, fabs(Inf) = inf
sin(inf_f) = -inf, sin(inf_d) = -inf, sin(Inf) = -inf
cos(inf_f) = -inf, cos(inf_d) = -inf, cos(Inf) = -inf
nan_f > 0.1f = 0, nan_d > 0.1f = 0, NaN > 0.1f = 0
nan_f < 0.1f = 0, nan_d < 0.1f = 0, NaN < 0.1f = 0
nan_f == 0.1f = 0, nan_d == 0.1f = 0, NaN == 0.1f = 0
nan_f > 0.0f = 0, nan_d > 0.0f = 0, NaN > 0.0f = 0
nan_f < 0.0f = 0, nan_d < 0.0f = 0, NaN < 0.0f = 0
nan_f == 0.0f = 0, nan_d == 0.0f = 0, NaN == 0.0f = 0
inf_f > 0.1f = 1, inf_d > 0.1f = 1, Inf > 0.1f = 1
inf_f < 0.1f = 0, inf_d < 0.1f = 0, Inf < 0.1f = 0
inf_f == 0.1f = 0, inf_d == 0.1f = 0, Inf == 0.1f = 0
nan_f > inf_f = 0, nan_d > inf_d = 0, NaN > Inf = 0
nan_f < inf_f = 0, nan_d < inf_d = 0, NaN < Inf = 0
nan_f == inf_f = 0, nan_d == inf_d = 0, NaN == Inf = 0
msh />

```

