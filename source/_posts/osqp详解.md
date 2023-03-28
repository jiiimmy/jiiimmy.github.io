---
title: osqp详解
date: 2023-03-02 16:58:07
tags:
---




OSQP的数据结构：

## OSQPData
定义为：
```c++
typedef struct {
  c_int    n; // 变量的个数 n
  c_int    m; // 约束的个数 m
  csc     *P; // csc格式下的二次cost矩阵P的上三角部分，大小为 n*n
  csc     *A; // csc格式下的线性约束矩阵A
  c_float *q; // cost函数线性部分的密集阵列（大小n）
  c_float *l; // 约束下界密集阵列(大小 m)
  c_float *u; // 约束上界密集阵列(大小 m)
} OSQPData;
```

## csc
```c++
typedef struct {
  c_int    nzmax; // 最大条目数
  c_int    m;     // 行数
  c_int    n;     // 列数
  c_int   *p;     // 列指针（大小n+1）；当使用三元组格式（直接KKT矩阵形成）时，列索引（大小nzmax）从0开始
  c_int   *i;     // 行索引，大小nzmax从0开始
  c_float *x;     // 数值，size为nzmax
  c_int    nz;    // 三元组矩阵中的条目数，csc为-1
} csc;
```
$\rho \sigma$
## OSQPSettings
```c++
typedef struct {
  c_float rho;                    // ADMM 算法 ρ(rho)步长
  c_float sigma;                  // ADMM 算法 σ(sigma)步长
  c_int   scaling;                // 启发式数据缩放迭代次数，若为0则禁用

# if EMBEDDED != 1
  c_int   adaptive_rho;           // 布尔值，rho步长是否自适应？
  c_int   adaptive_rho_interval;  // rho自适应之间的迭代次数，如果为0，则为自动设置
  c_float adaptive_rho_tolerance; // 自适应rho的公差X。新的ρ必须比当前的rho大X倍或小1/X倍才能触发新的因子分解。
#  ifdef PROFILING
  c_float adaptive_rho_fraction;  // 调整rho的间隔（设置时间的一部分）
#  endif // Profiling
# endif // EMBEDDED != 1

  c_int                   max_iter;      // 最大迭代次数
  c_float                 eps_abs;       // 绝对收敛公差
  c_float                 eps_rel;       // 相对收敛公差
  c_float                 eps_prim_inf;  // 原始不可行性公差
  c_float                 eps_dual_inf;  // 对偶不可行性公差
  c_float                 alpha;         // 松弛参数
  enum linsys_solver_type linsys_solver; // 要使用的线性系统解算器

# ifndef EMBEDDED
  c_float delta;                         // 平滑正则化参数
  c_int   polish;                        // 布尔值，是否平滑 ADMM 结果
  c_int   polish_refine_iter;            // 平滑中的迭代求精步骤数

  c_int verbose;                         // 布尔值，是否写出进度
# endif // ifndef EMBEDDED

  c_int scaled_termination;              // 布尔值，是否使用缩放终止条件
  c_int check_termination;               // 整数，检查终止间隔；如果为0，则禁用终止检查
  c_int warm_start;                      // 布尔型，是否热启动

# ifdef PROFILING
  c_float time_limit;                    // 解决问题所允许的最大秒数；如果为0，则禁用
# endif // ifdef PROFILING
} OSQPSettings;
```

## OSQPScaling
```c++
typedef struct {
  c_float  c;    // 成本函数标度 cost function scaling 
  c_float *D;    // 原始变量标度 primal variable scaling
  c_float *E;    // 对偶变量标度  dual variable scaling
  c_float  cinv; // 损失函数重改标量 cost function rescaling
  c_float *Dinv; // 原始变量重新缩放 primal variable rescaling
  c_float *Einv; ///< dual variable rescaling
} OSQPScaling;
```

## OSQPSolution

```c++
typedef struct {
  c_float *x; // 原始解
  c_float *y; ///< Lagrange multiplier associated to \f$l <= Ax <= u\f$ 拉格朗日乘数
} OSQPSolution;
```

## OSQPInfo
```c++
typedef struct {
  c_int iter;          // 所用迭代次数
  char  status[32];    // 状态字符串，如 'solved'
  c_int status_val;    // 状态值，定义在 constants.h

# ifndef EMBEDDED
  c_int status_polish; // 平滑状态：成功（1），未完成（0），（-1）不成功
# endif // ifndef EMBEDDED

  c_float obj_val;     // 原始目标
  c_float pri_res;     // 原始剩余范数 norm of primal residual
  c_float dua_res;     // 对偶剩余范数 norm of dual residual

# ifdef PROFILING
  c_float setup_time;  // 设置阶段所用的时间（秒）
  c_float solve_time;  // 求解阶段所用的时间（秒）
  c_float update_time; // 更新阶段所用的时间（秒）
  c_float polish_time; // 平滑阶段所用的时间（秒）
  c_float run_time;    // 总花费时间
# endif // ifdef PROFILING

# if EMBEDDED != 1
  c_int   rho_updates;  // rho 更新次数
  c_float rho_estimate; // 迄今为止从残差得到的最佳ρ估计
# endif // if EMBEDDED != 1
} OSQPInfo;
```

## OSQPPolish
```c++
typedef struct {
  csc *Ared;          // A的活动行 Ared = vstack[Alow, Aupp]
  c_int    n_low;     // 较低活动行数 number of lower-active rows
  c_int    n_upp;     // 较高活动行数 number of upper-active rows
  c_int   *A_to_Alow; ///< Maps indices in A to indices in Alow
  c_int   *A_to_Aupp; ///< Maps indices in A to indices in Aupp
  c_int   *Alow_to_A; ///< Maps indices in Alow to indices in A
  c_int   *Aupp_to_A; ///< Maps indices in Aupp to indices in A
  c_float *x;         ///< optimal x-solution obtained by polish
  c_float *z;         ///< optimal z-solution obtained by polish
  c_float *y;         ///< optimal y-solution obtained by polish
  c_float  obj_val;   ///< objective value at polished solution
  c_float  pri_res;   ///< primal residual at polished solution
  c_float  dua_res;   ///< dual residual at polished solution
} OSQPPolish;
```