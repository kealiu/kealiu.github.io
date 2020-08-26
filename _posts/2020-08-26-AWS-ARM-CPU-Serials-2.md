# ARM 验证之NodeJS

[Node.js](https://nodejs.org/) 是一个基于 Chrome V8 引擎的 JavaScript 运行时。使用了事件驱动、非阻塞式 I/O 的模型。用于方便地搭建响应速度快、易于扩展的网络应用。Node 使用事件驱动， 非阻塞I/O 模型而得以轻量和高效，非常适合在分布式设备上运行数据密集型的实时应用

ARM CPU相关介绍，请参考： [AWS ARM CPU 简介](https://kealiu.github.io/AWS-ARM-CPU-Serials-1/)

## 验证目标

常见的node模块使用的是`javascirpt`源码，但是为了高性能以及特定功能，nodejs也支持native addon。我们通过验证addon 功能可用性，来确定m6g系列对nodejs生态的支持。

### node-gyp

`node-gyp`是用Node.js编写的跨平台命令行工具，用于为Node.js编译CPU native & OS native 模块。 它源自于chromium项目，用于统一管理所有软硬件环境addon。

### node-addon简介

Addons are dynamically-linked shared objects written in C++. The require() function can load addons as ordinary Node.js modules. Addons provide an interface between JavaScript and C/C++ libraries.

addons 是用`C++`编写的动态链接库文件(`.so`)。 `require（）`函数可以将.so加载项作为普通的`Node.js`模块加载。 addons实现了`JavaScript`调用`C/C++` lib 的接口。

# 验证测试

通过成功运行 [https://github.com/freezer333/nodecpp-demo/tree/master/quickstart](https://github.com/freezer333/nodecpp-demo/tree/master/quickstart) 中的示例代码进行验证。

## 环境初始化

### 机器配置
cpu arch |  实例类型 | CPU核数 | 内存 |   硬盘  |
--------|---------|-------|----|-------|
arm      | m6g.large | 2       | 8G   | 50G SSD |

### 系统环境

Ubuntu 18.04 TLS, AWS offical version

### 安装软件

```
# 安装 nodejs npm 等基础包，以及g++ make 等编译环境
sudo apt install -y git g++ make nodejs npm

# 安装 nan (native abstractions for nodejs), node-gyp
sudo npm install -g nan node-gyp
```

### 下载源码
```
git clone https://github.com/freezer333/nodecpp-demo
```

### 运行
```
export NODE_PATH=$(npm root -g)
cd nodecpp-demo/quickstart/test
npm install
```

## 验证代码


```
$node index.js
Hooray!  The addon worked as expected
```

## 验证结果

测试代码repo内还包含其他demo，可以自行进入目录后执行`npm install`，然后执行对应的 `node demo-name.js` 。如 `objectwrap_nan` demo

```
..nodecpp-demo/objectwrap_nan$ npm install

> primes@1.0.0 install /home/ubuntu/nodejs/nodecpp-demo/objectwrap_nan
> node-gyp rebuild

make: Entering directory '/home/ubuntu/nodejs/nodecpp-demo/objectwrap_nan/build'
  CXX(target) Release/obj.target/polynomial/polynomial.o
  SOLINK_MODULE(target) Release/obj.target/polynomial.node
  COPY Release/polynomial.node
make: Leaving directory '/home/ubuntu/nodejs/nodecpp-demo/objectwrap_nan/build'
npm WARN primes@1.0.0 No description
npm WARN primes@1.0.0 No repository field.

..nodecpp-demo/objectwrap_nan$ node polynomial.js
30
[ -1, -2 ]
28
Polynomial { c: 0, b: 3, a: 1 }
ubuntu@ip-172-31-3-251:~/nodejs/nodecpp-demo/objectwrap_nan$
```

# 总结

整个测试过程十分顺利，没有碰到任何坑。可见nodejs生态对arm支持还是很完善的。可以更进一步结合项目进行实战。
