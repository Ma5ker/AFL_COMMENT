# AFL源码阅读

对AFL的workflow的梳理，一些细枝末节的代码并未细究。

## main函数

### Sync_fuzzer


### fuzz_one

*参数程序运行参数*

*返回1表示新生成的测试用例出现了问题，之后跳过了测试用例；返回0 表示成功测试*

#### workflow

- 取queue头部测试用例q作为当前测试用例queue_cur
- 如果当前有favored测试用例在等待测试的话
  - 如果当前的q被fuzz过或者非favored的测试用例，则随机以99%概率跳过(返回1)
- 当前无favored测试用例等待测试时，如果当前的q不是favored，并且queue长度大于10
  - 如果轮次大于1，且当前的没被fuzz，随机以75%概率跳过(返回1)
  - 否则以95%概率跳过，(返回1)
- 如果当前的q->cal_failed不为0，表示校准测试用例出现了问题
  - 如果是之前在calibration_case校验失败进入了abort_calibration，则尝试再次校验
    - 运行程序出错则停止整个fuzz；校验失败则跳过当前的测试用例，跳转至**abandon_entry**(这段后面到了再看)
- 修剪
- 性能分数评估 calculate_score
  - 基于运行速度打分（与平均执行时间对比，越短，分越高）
  - 基于q的bitmap_size（即测试用例触发的bitmap为1的数量）分数倍增；越大，分乘的倍数越高
  - 基于q的轮次对分数进行倍增（与知道路径的早晚有关）；轮次越多，分数越高
  - 基于q的深度打分；越深越好
  - 分数上限1600
- 进入变异部分
  - deterministic stage 确定性阶段
    - 位翻转
    - 算术变异（加减）
    - 比特位替换为一些interesting value
    - 基于字典的替换和插入：有用户提供的token、或者自动检测生成的token
  - havoc stage 大破坏阶段
    - 确定性阶段的各种方法随机组合使用
  - splice stage 拼接阶段
- abandon_entry：
  - 修改当前测试用例信息：fuzz过
  - 修改全局计数器：待测试用例减少1，如果当前为favored测试用例，则待测试favored减少1

#### deterministic stage

确定性阶段，对输入本身进行一些变异

##### BITFLAP

位翻转过程

- 1bit翻转
  - 每一轮翻转一个bit
  - 调用common_fuzz_stuff测试并保存
  - 然后恢复被翻转的bit
  - 后面的翻转基本与此阶段一样
- 2bit翻转 
  - 每次翻转相邻的两个bit
- 4bit翻转
- 8bit翻转
  - 同时将完全翻转都不会对程序执行trace产生影响的字节进行记录（effector map），在后续其他确定性阶段部分直接跳过
- 16bit翻转:每相邻两个字节翻转
  - 如果翻转的两个字节均对trace无影响（effector map）,则跳过
- 32bit翻转：相邻四个字节翻转
  - 如果翻转的四个字节均对trace无影响（effector map）,则跳过

##### ARITHMETIC INC/DEC

对字节(s)进行增减测试

- 8bit 增减测试
  - 从1开始到ARITH_MAX（35）
    - 进行加法运算，并确保变异后输入不会与位翻转得到一样的；运行测试
    - 进行减法运算，并确保变异后输入不会与位翻转得到一样的；运行测试
- 16 bit增减测试
  - 大端编码与小端编码分别进行加减，并确保不会通过位翻转得到
- 32 bit增减测试 和16nit的基本一样

##### INTERESTING VALUES

利用一些特殊的int值进行替换

- int8替换
  - 设置了一些interesting 8bit值，挨个字节替换；同样检测了effector map 和是是否与前面的操作结果一样来跳过
- int16替换
  - 与8bit的区别只是分了大小端，其他基本一样
- int32替换
  - 与8bit的区别只是分了大小端，其他基本一样

##### DICTIONARY STUFF

利用字典的extras进行替换或插入

- 用户提供的extras替换
  - 以每个字节作为起始，将后续内存覆盖为extras
  - 遇到覆盖的字节在effector map都是无效，或者长度不够则跳过
  - 遇到数量超过阈值时，生成随机数来决定是否跳过
- 用户提供的extras插入
  - 以每个字节作为起始，尝试将extras和尾部插入
- 自动生成的extras替换
  - 和用户提供的流程一样

确定性阶段结束，进入随意大破坏阶段

#### RANDOM HAVOC

随意大破坏阶段。havoc可以理解为对多个确定性阶段的不同操作的组合。

*需要注意的是，havoc阶段，在splicing后也会调用到。*

- 设置havoc轮数stage_max，参考了是否跳过确定性阶段，跳过的轮数更少(因为可能是之前fuzz过的恢复执行)
- 对每个阶段
  - 随机确定组合使用的操作数量use_stacking，开始此阶段的组合
    - bitflap (case 0)
      - 1bit翻转
    - interesting value(case 1-3)
      - byte/word/dword用interesting value替换，随机大小端
    - ARITHMETIC INC/DEC(case 4-9)
      - 随机选取某个位置的byte/word/dowrd，随机大小端，进行随机大小的加减
    - 随机字节设置为随机值(case 10)
    - 随机找个位置删除随机长度的字节(case 11-12)
    - 对输入进行重组，随机选择下面的一种（case 13）
      - clone(75%)
        - [0,end] -> [ 0, clone_to ] + [ clone_from, clone_from+clone_len ] + [ clone_to, end ]
      - insert(25%)
        -  [0,end] -> [ 0, clone_to ] + [随机0-255 | 随机输入的某个值 ] * clone_len    + [ clone_to, end ]
    - 随机选择一部分去覆盖随机选择的另一部分（case 14）
      - 75%：[0,end] -> [0 , copy_to ] + [copy_from , copy_from+copy_len] + [..., end]
      - 25%：[0,end] -> [0 , copy_to ] + [随机0-255 | 随机输入的某个值 ] * copy_len + [..., end]
    - 替换和插入extras （case 15-16）
      - 随机找到一个位置将其内容替换为extras
      - 随即找到一个位置，插入extras

#### SPLICING

大致为选一个测试用例与当前测试用例进行一个拼接，具体流程如下：

- 轮次执行SPLICE_CYCLES(15)
- 一些基本保证(当前queue大小至少为2；当前testcase大于1)
- 从queue中选择一个testcase准备进行拼接
- 随机选择两者第一个不同字节与最后一个不同字节之间的位置作为拼接位置split_at
- 进行拼接，拼接结果 ：[0,end] ->  当前测试用例[0,split_at ] + 随机选定测试用例[split_at,end]
- 转至havoc阶段进一步变异

