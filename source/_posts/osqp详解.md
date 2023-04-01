---
title: osqp详解
date: 2023-03-02 16:58:07
tags:
---

osqp 是一种二次规划求解器，可用求解线性组合或二次规划问题，在同类问题中求解效率极高，其由牛津大学开源
<!-- more -->

[osqp github开源地址](https://github.com/osqp/osqp)

# OSQP的数据结构：

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

# 使用osqp求解实际问题
## 问题描述
![](/source/img/osqp-demo.png)
对应的优化函数为：
$f=2x_1^2 + x_1x_2+x_2^2+x_1+x_2$
约束为：
$x_1+x_2=1, 0<=x_1<=0.7 , 0<=x_2<=0.7$
## 工程构建
利用bazel来构建项目，首先新建一个 test_osqp 的目录，然后克隆osqp的代码到该目录下，并在 test_osqp 目录下新建 WORKSPACE 文件
```shell
mkdir test_osqp && cd test_osqp
git clone https://github.com/osqp/osqp.git
touch WORKSPACE
```
首先在新建一个脚本 build.sh 并加上执行权限，用 cmake 去编译osqp，内容如下:
```shell
set -e
base_dir=$(cd "$(dirname "$0")";pwd)
echo $base_dir
cd $base_dir/osqp/
rm -rf $base_dir/osqp/build/
mkdir $base_dir/osqp/build/

cd $base_dir/osqp/build/

cmake -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=/usr/local -DBUILD_SHARED_LIBS=ON ..
make -j16
make install 

rm -rf $base_dir/osqp/build/
```
运行完脚本后会在本地的 /usr/local 目录下生成 osqp 相关的头文件以及动、静态库，将他们拷贝到工程目录下的 include 文件夹和 lib 文件夹：
```shell
mkdir include &&  cd include
cp -r /usr/local/include/osqp/ ./
cp -r /usr/local/include/qdldl/ ./
cd ..
mkdir lib && cd lib
cp /usr/local/lib/libosqp.* .
```
这时候以及将编译好的osqp.so和osqp.lib放在了工程目录下的 lib，将对应的头文件放在了工程目录下的 include。然后在工程目录下新建BUILD文件，内容如下：
```
package(default_visibility = ["//visibility:public"])

load("@rules_cc//cc:defs.bzl", "cc_library")
global_definitions()

cc_library(
    name = "osqp",
    srcs = ["lib/libosqp.so"],
    hdrs = glob(["include/**/*.h"]),
    includes = ["include"],
    linkopts = ["-losqp"],
    visibility = ["//visibility:public"],
)
```
这时候就能在 bazel 里面去使用 osqp 库了，在osqp 目录下新建一个 WORKSPACE 文件，内容为：
```shell
new_local_repository(
    name = "osqp",
    path = "/home/weiz3/learncpp/test_osqp/osqp",
    build_file = "BUILD"
)
```
这里 path 指向本地 osqp 的路径，此时就建立了一个本地的 osqp 库，在 osqp 同级目录下新建一个main 目录，新建 main.cpp 来求解上面的数学问题，内容为：
```c++
#include "osqp/osqp.h"
#include <iostream>

int main(int argc, char **argv) {
    // Load problem data
    c_float P_x[3] = {4.0, 1.0, 2.0, };//目标矩阵的非零值
    c_int P_nnz = 3;                   //目标矩阵的非零值的个数
    c_int P_i[3] = {0, 0, 1, };        //目标矩阵的非零值所在的row，与P_x一一对应

    //P_p[i]=n,P_p[i+1]=m, 表示
    //for k from n to m:
    //  将P_x[k]填在第i列，P_i[k]行
    c_int P_p[3] = {0, 1, 3, };        //每一列的第一个非零元素所对应的P_x数组的indice，最后一个值肯定是P_nnz

    c_float q[2] = {1.0, 1.0, };
    c_float A_x[4] = {1.0, 1.0, 1.0, 1.0, };
    c_int A_nnz = 4;
    c_int A_i[4] = {0, 1, 0, 2, };
    c_int A_p[3] = {0, 2, 4, };
    c_float l[3] = {1.0, 0.0, 0.0, };
    c_float u[3] = {1.0, 0.7, 0.7, };
    c_int n = 2;
    c_int m = 3;

    // Exitflag
    c_int exitflag = 0;

    // Workspace structures
    OSQPWorkspace *work;
    OSQPSettings  *settings = (OSQPSettings *)c_malloc(sizeof(OSQPSettings));
    OSQPData      *data     = (OSQPData *)c_malloc(sizeof(OSQPData));

    // Populate data
    if (data) {
      data->n = n;
      data->m = m;
      data->P = csc_matrix(data->n, data->n, P_nnz, P_x, P_i, P_p);
      data->q = q;
      data->A = csc_matrix(data->m, data->n, A_nnz, A_x, A_i, A_p);
      data->l = l;
      data->u = u;
    }

    // Define solver settings as default
    if (settings) {
      osqp_set_default_settings(settings);
      settings->alpha = 1.0; // Change alpha parameter
    }

    // Setup workspace
    exitflag = osqp_setup(&work, data, settings);

    // Solve Problem
    osqp_solve(work);

    std::cout <<"solution x1 = " << *work->solution->x << " x2= " << *(work->solution->x+1) <<std::endl;

    // Cleanup
    if (data) {
      if (data->A) c_free(data->A);
      if (data->P) c_free(data->P);
      c_free(data);
    }
    if (settings) c_free(settings);

    return exitflag;
  };
```
在main目录下新建WORKSPACE文件，用来将刚刚的 osqp 加载过来，文件内容为：
```shell
local_repository( name = "osqp",        path = "../osqp" )
```
再新建BUILD文件，构建最后的可执行程序，文件内容为：
```shell
package(default_visibility = ["//visibility:public"])
load("@rules_cc//cc:defs.bzl", "cc_binary", "cc_library")
global_definitions()

cc_binary(
  name = "test",
  srcs = [
    "main.cpp"
  ],
  deps = [
    "@osqp//:osqp"
  ],
  copts = [
        # 指定使用 c++11 编译
        "-std=c++11"
    ]
)
```
最后进入 main 目录，使用 `bazel build :test`，编译完后在 main/bazel-bin 下便会存在 test 的可执行程序，在 main 目录执行 `./bazel-bin/test`，可以得到程序的运行结果为：
```shell
-----------------------------------------------------------------
           OSQP v0.0.0  -  Operator Splitting QP Solver
              (c) Bartolomeo Stellato,  Goran Banjac
        University of Oxford  -  Stanford University 2021
-----------------------------------------------------------------
problem:  variables n = 2, constraints m = 3
          nnz(P) + nnz(A) = 7
settings: linear system solver = qdldl,
          eps_abs = 1.0e-03, eps_rel = 1.0e-03,
          eps_prim_inf = 1.0e-04, eps_dual_inf = 1.0e-04,
          rho = 1.00e-01 (adaptive),
          sigma = 1.00e-06, alpha = 1.00, max_iter = 4000
          check_termination: on (interval 25),
          scaling: on, scaled_termination: off
          warm start: on, polish: off, time_limit: off

iter   objective    pri res    dua res    rho        time
   1  -4.9384e-03   1.00e+00   2.00e+02   1.00e-01   2.95e-05s
  50   1.8800e+00   1.91e-07   7.50e-07   1.38e+00   4.48e-05s

status:               solved
number of iterations: 50
optimal objective:    1.8800
run time:             4.77e-05s
optimal rho estimate: 1.36e+00

solution x1 = 0.3 x2= 0.7
```
得到的最优解为 x1 = 0.3, x2 = 0.7
