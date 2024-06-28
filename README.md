# ysyx_ci_result

> 注意：从这次流片班车开始，CICD的代码分支命名方式修改为当前班车接收代码的起始时间，格式为`年份+月份`，比如这次班车的分支就是`test`。

这是存放一生一芯CICD测试后返回报告的仓库。`report`目录存放有SoC和后端团队对前端代码进行功能仿真和综合后返回的报告。运行一次完整的vcs测试时间大约在40-50分钟之间，一次完整dc测试时间在0.5-2个小时不等。由于一次测试时间较长，所以我们在`report`自己学号目录下的`state`文件记录有当前核在提交队列中的位置，具体格式为：
```txt
state: under test
```
表示当前核正在测试中，或者为：
```txt
state: wait [nums] duts
```
表示前面还有`[nums]`个核在等待测试。自己核的测试在`[nums]+1`个。

## CICD功能
目前测试环境已经切换到28nm，并添加了新的功能：
1. 支持将vcs测试中fail时的run.log添加到返回的报告中。比如跑sdram中的rtthread出错，则会在vcs_report中的`!!!!rtthread test in sdram fail!!!!`一行下添加运行log：
```txt
!!!!rtthread test in sdram fail!!!!


The run log is:
======================================================
Chronologic VCS simulator copyright 1991-2020
Contains Synopsys proprietary information.
Compiler version R-2020.12_Full64; Runtime version R-2020.12_Full64;  Mar  1 12:37 2023
==============================================
===INFO=== Model Configuration Info
===INFO=== Device Name:  N25Q128A13E
===INFO=== DQ3 = HOLD pin
===INFO=== XIP type = Numonyx
===INFO=== VCC 3V
==============================================
[0.0 ns ns] ==INFO== Load sdram content.
[0.0 ns ns] ==INFO== Hold condition disabled: communication with the device has been activated.
[0.0 ns ns] ==INFO== Protocol selected is extended
[0.0 ns ns] ==INFO== 2. Load memory content from file: "mem_Q128_bottom.vmf".
[0.0 ns ns] ==INFO== Power up: polling allowed.
[0.0 ns ns] ==INFO== VCC has been driven below threshold : internal logic will be reset.
[0.0 ns ns] ==INFO== Protocol selected is extended
[0.0 ns ps] ==INFO== Single Transfer Rate selected
[1.0 ns ns] ==INFO== Load flash discovery paramater table content from file: "sfdp.vmf".
[150.0 ns ns] ==INFO== Power up: device fully accessible.
[150.0 ns ns] ==INFO== Protocol selected is extended
[150.0 ns ps] ==INFO== Single Transfer Rate selected
jump to sdram...
heap: [   0022280 - 0 88422210]
 2006 - 2021 Copyright by rt-thread team
$finish called from file "../tb/asic_system.v", line 431.
$finish at simulation time 200000000.0 ns
           V C S   S i m u l a t i o n   R e p o r t 
Time: 200000000000 ps
CPU Time:    345.030 seconds;       Data structure size:  38.5Mb
Wed Mar  1 12:43:25 2023
======================================================
```
> 注意：`[150.0 ns ps] ==INFO== Single Transfer Rate selected`的下一行开始到`$finish called from file "../tb/asic_system.v", line 431.`的上一行截止为vcs仿真运行的结果。上面可以明显看到仿真时处理器核打印的结果异常。

2. 支持进行单独的dc测试并设置时钟约束的频率，并返回相应的报告。
3. 对于dc+vcs的联合测试，为了尽可能减少核等待测试的时间，CICD设置了一个测试策略：
   * 先运行vcs测试，如果vcs编译出错，则直接返回结果。
   * 如果vcs编译通过，则会依次运行全部的9个测试程序。
   * 如果9个测试程序中有一个出错，则会直接返回结果，否则开始运行dc测试。

目前CICD使用 **提交仓库中的 config.toml 来配置测试任务**，`config.toml` 可配置内容见：[def_config.toml](https://github.com/oscc-soc/ci/blob/main/src/def_config.toml)，添加并修改 `config.toml` 的具体步骤如下。

* 拷贝 [def_config.toml](https://github.com/oscc-soc/ci/blob/main/src/def_config.toml) 到自己代码提交仓库，并修改文件名为 `config.toml`。

* 按照 `config.toml` 中的注释要求修改 `[dut]`, `[vcs]` 和 `[dc]` 下的配置选项。

* 注意：当设置 `[vcs]` 下的 `wave = on`时， CICD会返回vcs的仿真波形。考虑到仿真波形的大小，目前暂且只支持生成 `hello-flash, hello-mem, hello-sdram` 这3个程序的波形，波形返回时间大约30~40分钟。波形文件格式为FST，可以使用GTKWave打开。文件命名方式为：`处理器核顶层名_hello_[flash|mem|sdram]_待测试commit的提交时间.fst.tar.bz2`。波形文件存储在网盘服务器上，并只会存储24小时，过时自动删除。网盘服务器地址为：http://wave.maksyuki.com/share/65ec12649eb39/

实际config.toml设置可参考：[ysyx_000000/config.toml](https://gitee.com/maksyuki/ysyx_000000/blob/master/config.toml)



## CICD目录结构
### report 目录结构
```sh
├─ ysyx_XXXXXX(学号)          #每位同学的报告存放在自己学号的顶层目录下
   ├─ 2024-3-xx...xx:xx:xx    #根据不同的代码拉取时间，支撑团队返回的报告存放在不同的目录下
   │  ├─ dc_report            #DC综合报告
   │  └─ vcs_report           #VCS仿真报告
   └─ state                   #当前提交测试状态
```

## report
SoC和后端团队的report返回后，同学需要按照以下步骤确认报告内容及对代码做出相应的修改。

### 一、确认功能正确性
* 阅读VCS报告 `report/ysyx_学号/2024-3-xx...xx:xx:xx/vcs_report`，确认编译结果是否有错误，确认VCS流程是否存在 **`fail`**。如果VCS程序未通过，则首先需要修改代码以确保功能正确，确认控制信号寄存器是否已做初始化。

### 二、消除DC综合报告Warning/Error
* 阅读DC综合报告 `report/ysyx_学号/2024-3-xx...xx:xx:xx/dc_report`的 **`RUN LOG`** 部分，关注报告中的Warning和Error。清除所有的Warning和Error，根据Warning的提示相应地去修改代码，对于无法清除的Warning，需要填写该仓库下的[syn_warning.md](./syn_warning.md)，发给各自的组长，由组长反馈给支撑团队。
> 注意：必须要修改代码清除掉的Warning/Error类型为LINT-3、LINT-38、LINT-59、LINT-60、LINT-X4和LINT-58

### 三、确认综合后面积是否在约束范围内
同学需要确保设计综合后的 **`Total cell area`** 在约束范围内，整个SoC面积应不超过 **2.0平方毫米**，其中共享SRAM的大小约为 **0.4平方毫米**，**所以同学们自己核的面积应不超过1.6平方毫米**。

> 注意：以上面积约束大小需要根据最终综合后结果确定。

* 阅读DC综合报告 `report/ysyx_学号/2024-3-xx...xx:xx:xx/dc_report`的 **`AREA REPORT`** 部分，确认 `Total cell area` 是否满足约束范围内。
  * 如做了五级流水线的设计 **Total cell area** 超过了约束范围，请对设计进行优化，将面积优化到约束范围内。
  * 如果做了乱序多发射的设计 **Total cell area** 超过了约束范围，请分析报告，找出面积较大的模块，说明面积过大的原因，编写说明文档[area_warning.md](./area_warning.md)，没有具体格式要求，按要求说明原因即可，编写完成后发给各自的组长，再由组长反馈给支撑团队进一步评估，如果支撑团队评估后觉得不合适，则需要请设计人员进一步简化设计。

### 四、确认频率
支撑团队提供 **`100MHz`** 约束基准的DC综合流程。这里采用100MHz频率综合是为了给后端设计修时序留余量，设了0.35的过约比，实际的频率会比100M高，时序约束条件更为严格。
>注意：通过100MHz的时序约束是满足流片要求的最低要求，目前CICD已经提供返回更高频率dc报告的功能，学有余力的同学可以自己尝试依据报告优化时序。

  例如：
  ![1633343064(1)](https://user-images.githubusercontent.com/82496491/135835292-fad710f7-aa2f-46a8-aacb-c981f21f43ac.png)

  在100MHz频率下，图中标记的data required time应该接近10ns，可以看到实际的data required time只有6.2189ns，约束更为严格。

* 阅读DC综合报告 `report/ysyx_学号/2024-3-xx...xx:xx:xx/dc_report` 的 **`STATISTICS REPORT`** 部分，查看DC综合流程在100MHz频率下是否 **PASS**。时序通过的标志为： `wns >= 0` 且 `tns >= 0`：

```txt
#========================================================================
# Timing
#========================================================================
group      org_freq  over_freq  wns    tns    num
CLK_clock  100.0     153.8      0.000  0.000  0 

```

>注意：由于存在DC license的问题，支撑团队无法提供DC工具的使用，工程师人力及资源不足，无法提供一对一优化。因此，支撑团队只保证芯片的正确性，不辅助优化设计，有意愿追求高性能的同学可以自行优化，例如：参考开源EDA工具yosys。如果核的设计足够好，回片后是可以进行调频的。

100MHz频率未通过或想要做优化的同学，可以关注报告中以下部分的关键时序路径。
* 阅读DC综合报告 `report/ysyx_学号/2024-3-xx...xx:xx:xx/dc_report` 的 **`TIME REPORT`** 部分，报告中列出5条最长路径供参考，关于时序报告的格式可自行参考相关资料，例如：Vivado的时序报告。

# 代码拉取时间
支撑团队提供的CICD流程，**每隔3分钟** 都会拉取一次代码。每次拉取代码后，报告返回时间不是确定的。当次拉取的所有代码在跑完DC&VCS流程后，会将当次拉取的所有代码的报告会按照学号划分返回到github仓库ysyx_submit/report目录下。

# 代码提交截止日期
* 2024年4月1日之后不允许修改代码。
