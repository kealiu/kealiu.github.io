# AWS ARM CPU 简介

以下内容来自[AWS Gravition CPU介绍](https://aws.amazon.com/cn/ec2/graviton/?nc1=h_ls)，直接引用不再重复介绍。

> AWS Graviton 由 Amazon Web Services 使用 64 位 Arm Neoverse 内核定制而成，为在 Amazon EC2 中运行的云工作负载提供更高的性价比。Amazon EC2 提供更为广泛且深入的计算实例组合，其中包括许多由新一代 Intel 和 AMD 处理器提供支持的实例。AWS Graviton 处理器带来更多选择，帮助客户优化性能和降低工作负载成本。

> 第一代 AWS Graviton 处理器支持 Amazon EC2 A1 实例 – AWS 上第一个基于 Arm 的实例。对于扩展的应用程序（如 Web 服务器、容器化微服务、数据/日志处理和其他可以在更小的内核上运行并适合可用内存占用空间的工作负载），这些实例与其他通用实例相比可节省大量成本。

> 与第一代 AWS Graviton 处理器相比，AWS Graviton2 处理器不管在性能还是功能上都实现了巨大的飞跃。 它们都支持 Amazon EC2 M6g、C6g 和 R6g 实例及其具有本地基于 NVMe 的 SSD 存储的变体，而且与当前这一代基于 x86 的实例1相比，这些实例为各种工作负载（包括应用程序服务器、微服务、高性能计算、电子设计自动化、游戏、开源数据库和内存中的缓存）提供高达 40% 的性价比提升。AWS Graviton2 处理器也为视频编码工作负载提供增强的性能，为压缩工作负载提供硬件加速，并为基于 CPU 的机器学习推理提供支持。它们可以提供高 7 倍的性能、多 4 倍的计算核心、快 5 倍内存和大 2 倍缓存。

# CPU 性能测试

CPU性能测试涉及的因素非常多，是一个复杂的场景，测试出来的结果可能与你的实际应用相差甚远。因而，我们会进行一次通用测试，一次特定场景应用测试。

## 测试配置


### 硬件环境

cpu arch | 实例类型 | CPU核数 | 内存 | 硬盘
---|--- |--- |--- |---
x86 | m5.large | 2 | 8G | 50G SSD
arm | m6g.large | 2 | 8G | 50G SSD

### CPU详情

#### arm

```
$ lscpu
Architecture:        aarch64
Byte Order:          Little Endian
CPU(s):              2
On-line CPU(s) list: 0,1
Thread(s) per core:  1
Core(s) per socket:  2
Socket(s):           1
NUMA node(s):        1
Vendor ID:           ARM
Model:               1
Stepping:            r3p1
BogoMIPS:            243.75
L1d cache:           64K
L1i cache:           64K
L2 cache:            1024K
L3 cache:            32768K
NUMA node0 CPU(s):   0,1
Flags:               fp asimd evtstrm aes pmull sha1 sha2 crc32 atomics fphp asimdhp cpuid asimdrdm lrcpc dcpop asimddp ssbs
```

#### x86

```
$ lscpu
Architecture:        x86_64
CPU op-mode(s):      32-bit, 64-bit
Byte Order:          Little Endian
CPU(s):              2
On-line CPU(s) list: 0,1
Thread(s) per core:  2
Core(s) per socket:  1
Socket(s):           1
NUMA node(s):        1
Vendor ID:           GenuineIntel
CPU family:          6
Model:               85
Model name:          Intel(R) Xeon(R) Platinum 8175M CPU @ 2.50GHz
Stepping:            4
CPU MHz:             3103.491
BogoMIPS:            5000.00
Hypervisor vendor:   KVM
Virtualization type: full
L1d cache:           32K
L1i cache:           32K
L2 cache:            1024K
L3 cache:            33792K
NUMA node0 CPU(s):   0,1
Flags:               fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ss ht syscall nx pdpe1gb rdtscp lm constant_tsc rep_good nopl xtopology nonstop_tsc cpuid aperfmperf tsc_known_freq pni pclmulqdq ssse3 fma cx16 pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand hypervisor lahf_lm abm 3dnowprefetch invpcid_single pti fsgsbase tsc_adjust bmi1 hle avx2 smep bmi2 erms invpcid rtm mpx avx512f avx512dq rdseed adx smap clflushopt clwb avx512cd avx512bw avx512vl xsaveopt xsavec xgetbv1 xsaves ida arat pku ospke
```

### 软件环境
Ubuntu 18.04 TLS, AWS offical version

## 测试工具

[sysbench](https://github.com/akopytov/sysbench)是一个模块化、跨平台、多线程的常用测试工具。为了简单起见，我们采用其默认的CPU测试对x86, arm CPU进行benchmark打分。

## 测试方法

直接使用默认CPU测试集进行打分。

```
sudo apt update -y
sudo apt install -y sysbench

# for test 
sudo sysbench --test=cpu run
```

## 测试结果

sysbench测试中，arm获得了2959 events/s得分，接近x86 1107 events/s 的3倍，表现优异。

当然，该结果仅仅表示sysbench测试结果。实际场景需要结合你的应用进行benchmark。

### arm sysbench结果

```
$ sudo sysbench --test=cpu run
WARNING: the --test option is deprecated. You can pass a script name or path on the command line without any options.
sysbench 1.0.11 (using system LuaJIT 2.1.0-beta3)

Running the test with following options:
Number of threads: 1
Initializing random number generator from current time


Prime numbers limit: 10000

Initializing worker threads...

Threads started!

CPU speed:
    events per second:  2959.98

General statistics:
    total time:                          10.0004s
    total number of events:              29605

Latency (ms):
         min:                                  0.34
         avg:                                  0.34
         max:                                  1.82
         95th percentile:                      0.34
         sum:                               9994.91

Threads fairness:
    events (avg/stddev):           29605.0000/0.00
    execution time (avg/stddev):   9.9949/0.00
```

### x86 sysbench结果

```
$ sudo sysbench --test=cpu run
WARNING: the --test option is deprecated. You can pass a script name or path on the command line without any options.
sysbench 1.0.11 (using system LuaJIT 2.1.0-beta3)

Running the test with following options:
Number of threads: 1
Initializing random number generator from current time


Prime numbers limit: 10000

Initializing worker threads...

Threads started!

CPU speed:
    events per second:  1107.32

General statistics:
    total time:                          10.0009s
    total number of events:              11076

Latency (ms):
         min:                                  0.90
         avg:                                  0.90
         max:                                  1.16
         95th percentile:                      0.90
         sum:                               9998.49

Threads fairness:
    events (avg/stddev):           11076.0000/0.00
    execution time (avg/stddev):   9.9985/0.00
```


# 以7z为例专项测试

arm与x86各有所长，所以我们需要根据应用特点进行对应的对比测试。 下面我们以7z为例进行专项对比测试

```
sudo apt install -y p7zip-full

# 单核测试
sudo 7z b -mmt1

# 多核测试
sudo 7z b
```
## 单核测试

单核测试结果arm与x86非常接近，没有明显的差异。

### arm 测试结果
```
$ sudo 7z b -mmt1

7-Zip [64] 16.02 : Copyright (c) 1999-2016 Igor Pavlov : 2016-05-21
p7zip Version 16.02 (locale=C.UTF-8,Utf16=on,HugeFiles=on,64 bits,2 CPUs LE)

LE
CPU Freq:  2493  2496  2497  2498  2496  2498  2497  2498  2498

RAM size:    7769 MB,  # CPU hardware threads:   2
RAM usage:    435 MB,  # Benchmark threads:      1

                       Compressing  |                  Decompressing
Dict     Speed Usage    R/U Rating  |      Speed Usage    R/U Rating
         KiB/s     %   MIPS   MIPS  |      KiB/s     %   MIPS   MIPS

22:       3412   100   3320   3320  |      41204   100   3518   3518
23:       3234   100   3295   3295  |      40549   100   3510   3510
24:       3102   100   3336   3336  |      39729   100   3488   3488
25:       2954   100   3374   3374  |      38929   100   3465   3465
----------------------------------  | ------------------------------
Avr:             100   3331   3331  |              100   3495   3495
Tot:             100   3413   3413
```

### x86测试结果

```
$ sudo 7z b -mmt1

7-Zip [64] 16.02 : Copyright (c) 1999-2016 Igor Pavlov : 2016-05-21
p7zip Version 16.02 (locale=C.UTF-8,Utf16=on,HugeFiles=on,64 bits,2 CPUs Intel(R) Xeon(R) Platinum 8175M CPU @ 2.50GHz (50654),ASM,AES-NI)

Intel(R) Xeon(R) Platinum 8175M CPU @ 2.50GHz (50654)
CPU Freq:  3086  3090  3090  3091  3091  3092  3092  3092  3092

RAM size:    7666 MB,  # CPU hardware threads:   2
RAM usage:    435 MB,  # Benchmark threads:      1

                       Compressing  |                  Decompressing
Dict     Speed Usage    R/U Rating  |      Speed Usage    R/U Rating
         KiB/s     %   MIPS   MIPS  |      KiB/s     %   MIPS   MIPS

22:       4058   100   3948   3948  |      36604   100   3125   3125
23:       3423   100   3488   3488  |      36043   100   3121   3120
24:       3315   100   3565   3565  |      35657   100   3130   3130
25:       3201   100   3655   3655  |      35309   100   3143   3143
----------------------------------  | ------------------------------
Avr:             100   3664   3664  |              100   3130   3130
Tot:             100   3397   3397
```

## 多核测试

多核测试中，arm全方位超过x86，以MIPS指标计算，性能提升40%以上。

### arm测试结果

```
$ sudo 7z b

7-Zip [64] 16.02 : Copyright (c) 1999-2016 Igor Pavlov : 2016-05-21
p7zip Version 16.02 (locale=C.UTF-8,Utf16=on,HugeFiles=on,64 bits,2 CPUs LE)

LE
CPU Freq:  2495  2497  2497  2498  2498  2498  2498  2498  2498

RAM size:    7769 MB,  # CPU hardware threads:   2
RAM usage:    441 MB,  # Benchmark threads:      2

                       Compressing  |                  Decompressing
Dict     Speed Usage    R/U Rating  |      Speed Usage    R/U Rating
         KiB/s     %   MIPS   MIPS  |      KiB/s     %   MIPS   MIPS

22:       7201   175   4002   7006  |      82223   200   3511   7020
23:       6985   177   4014   7118  |      80877   200   3505   7001
24:       7106   186   4117   7641  |      79481   200   3493   6978
25:       6863   186   4221   7836  |      77114   199   3456   6864
----------------------------------  | ------------------------------
Avr:             181   4088   7400  |              200   3491   6966
Tot:             190   3790   7183
```

### x86测试结果

```
$ sudo 7z b

7-Zip [64] 16.02 : Copyright (c) 1999-2016 Igor Pavlov : 2016-05-21
p7zip Version 16.02 (locale=C.UTF-8,Utf16=on,HugeFiles=on,64 bits,2 CPUs Intel(R) Xeon(R) Platinum 8175M CPU @ 2.50GHz (50654),ASM,AES-NI)

Intel(R) Xeon(R) Platinum 8175M CPU @ 2.50GHz (50654)
CPU Freq:  3082  3089  3090  3091  3089  3090  3092  3092  3092

RAM size:    7666 MB,  # CPU hardware threads:   2
RAM usage:    441 MB,  # Benchmark threads:      2

                       Compressing  |                  Decompressing
Dict     Speed Usage    R/U Rating  |      Speed Usage    R/U Rating
         KiB/s     %   MIPS   MIPS  |      KiB/s     %   MIPS   MIPS

22:       5535   170   3168   5385  |      50915   199   2184   4347
23:       5385   174   3158   5488  |      49831   200   2157   4313
24:       5335   178   3218   5736  |      51397   200   2259   4512
25:       5250   182   3285   5994  |      48639   199   2175   4329
----------------------------------  | ------------------------------
Avr:             176   3207   5651  |              199   2194   4375
Tot:             188   2700   5013
```

# 价格比较

以测试所用机器在美国ohio的价格为基准进行比较。全部定价请参考[AWS EC2官网价格](https://aws.amazon.com/cn/ec2/pricing/on-demand/)


实例 | 价格
---|---
m6g.large | $0.077/hour
m5.large | $0.096/hour

arm系列的价格约为x86价格的80% `(0.077/0.096=0.802)`

# 总结

arm系列整体性能较x86大幅提升，以7z场景为例，性能提升约40%，成本却可以降低20%。假设同样的算力需求。arm系列总体成本将只有x86的50%左右 `(0.6*0.8=0.48)`。

随着公有云厂商相继加入，未来arm发展肯定会越来越快，同时价格还会进一步降低。想节省基础设施成本的同学们可以加速入坑了。
