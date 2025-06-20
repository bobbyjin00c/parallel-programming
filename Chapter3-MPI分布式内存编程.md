# 并行程序设计笔记总结

## Chapter3.用MPI进行分布式内存编程

**MPI**：信息传递接口：使用消息传递的实现
· MPI是一个函数库

### MPI分布式变成函数

**开始与结束**
- MPI_Init
```cpp
void MPI_Init(argc_p, argv_p) {...}

```
初始化MPI环境，为所有进程分配资源并设置进程号
功能：MPI程序必须首先调用的初始化函数

- MPI_Finalize
```cpp
void MPI_Finalize() {...}
```
释放MPI占用的所有资源
功能：MPI程序结束前必须调用的清理函数

**通信子操作**
- MPI_Comm_size
```cpp
MPI_Comm_size(comm, size_p) {...}
```
获取指定通信子comm中的进程总数，结果存入size_p
功能：查询通信子包含的进程数量

- MPI_Comm_rank
```cpp
MPI_Comm_rank(comm, rank_p) {...}
```
获取当前进程在通信子comm中的编号，结果存入rank_p
功能：获取当前进程的标识号

**点对点通信**
- MPI_Send
```cpp
MPI_Send(buf_p, size, type, dest, tag, comm) {...}
```
发送buf_p指向的size个type类型数据，目标为comm通信子中dest进程，消息标记为tag
功能：向指定进程发送消息

- MPI_Recv
```cpp
MPI_Recv(buf_p, size, type, source, tag, comm, status_p) {...}
```
从comm通信子中source进程接收tag标记的消息，存入buf_p指向的size大小type类型缓冲区
功能：接收指定进程发来的消息

**通信机制**
- 点对点通信时阻塞式的，发送（MPI_Send）与接收（MPI_Recv）必须配对
- 消息顺序保持发送顺序，与接收顺序无关

**状态查询**
- MPI_Get_count
```cpp
MPI_Get_count(status_p, type, count_p){...}
```
通过status_p和type查询实际接收的数据量，存入count_p中
功能：获取接收到的数据量

- MPI_Probe
```cpp
MPI_Probe(source, tag, comm, status_p){...}
```
预先探测消息信息，而不接收数据
功能：消息预检查，用于动态内存分配

**集合通信**
- MPI_Bcast
```cpp
MPI_Bcast(buf_p, count, type, root, comm) {...}
```
root进程将buf_p指向的count个type类型数据广播给comm中所有进程
功能：一对多广播通信

- MPI_Scatter
```cpp
MPI_Scatter(send_p, send_count, send_type, recv_p, recv_count, recv_type, root, comm) {...}
```
root进程将send_p数据均匀分发给comm中所有进程，每个进程接受recv_count个recv_type类型数据到recv_p
功能：均匀数据分发

- MPI_Gather
```cpp
MPI_Gather(send_p, send_count, send_type, recv_p, recv_count, recv_type, root, comm){...}
```
所有进程将send_p数据发送到root进程的recv_p缓冲区
功能：多对一收集数据

- MPI_AllGather
```cpp
MPI_Allgather(send_p, send_count, send_type, recv_p, recv_count, recv_type, comm){...}
```
所有进程将send_p指向的send_count个send_type数据发送到所有进程的recv_P缓冲区
功能：全收集存储，所有进程获得全局数据

- MPI_Barrier
```cpp
MPI_Barrier(comm){...}
```
同步所有进程等待全部到达屏障点
功能：进程同步

**归约操作**
- MPI_Reduce
```cpp
MPI_Reduce(input_p, output_p, count, type, op, root, comm){...}
```
对所有进程的input_p数据执行op操作，结果存入root进程的output_p
功能：全局规约计算

- MPI_Allreduce
```cpp
MPI_Allreduce(intput_p, output_p, count, type, op, comm){...}
```
所有进程提供input_P指向的count个type数据，执行op操作（MPI_MAX,MPI_SUM等），结果存入所有进程的output_p

**计时函数**
- MPI_Wtime
```cpp
MPI_Wtime(){...}
```
返回单个进程的执行时间
功能：高精度计时，用于性能分析

**Gather**与**Allgather**对比
MPI_Gather: 多对一通信，存储到root进程，数据汇总到主进程
MPI_Allgather: 多对多通信，存储到所有进程，全局数据共享

**Reduce**与**Allreduce**对比
MPI_Reduce：多对一+规约，存储到root进程，主进程计算全局结果
MPI_Allreduce: 多对多+规约，存储到所有进程，全局同步计算结果

## Question
1. MPI相关知识：
   用于编写计算机程序的标准库，主要用于多进程间通信与数据交换
   不是语言，是函数库
   适合分布式内存的多处理系统


2. 集合通信：MPI中一种通信模式，涉及一组进程（通常是通信域中所有进程）之间的协同操作
   【特点】
   全局性：集合通信涉及通信域中的所有进程而不是单独的两个进程
   同步性：所有参与的进程必须同时调用集合通信函数，否则程序报错或挂起
   高效性：集合通信经过优化，能够利用底层硬件与网络的特性提高通信效率


3. 集合通信与点到点通信的区别
|特性	|点对点	|集合通信|
--------|--------|----------|
|参与进程	|两个进程	|通信域所有进程|
|同步性	|不需要全局同步	|隐式同步|
|通信模式|	一对一	|一对多，多对一，多对多|
|典型操作|	MPI_Send MPI_Recv	|MPI_Bcast MPI_Reduce|
|性能	|适用于局部通信	|适用于全局通信|
|代码复杂性|	较高	|较低|
|使用场景	|不规则通信，精细控制	|规则通信，全局协同操作|


4. MPI_Reduce:将通信域所有进程的数据通过某种op操作合并为一个结果存储在根目录中


5. MPI通过调用顺序匹配集合通信操作，确保每个进程的发送与接收可以正确匹配


6. Walk Clock Time: 程序实际运行时间，包括所有计算、等待与I/O时间
   CPU time：程序实际使用CPU进行计算的时间，不包括等待时间


7. MPI derived dadtatypes:MPI派生数据类型：用于定义复杂的数据结构以便在MPI进程之间高效输出非连续异构结构
   【用途】简化通信操作，提高代码可读性和性能


8. MPI_Bcast, MPI_Scatter, MPI_gather, MPI_Reduce集合函数区别
   Bcast广播：将指定进程中数据广播给通信子中所有进程
   Scatter分发：将某进程的所有数据按块分割均匀分发给通信子中所有进程（包括自己）
   Gather收集：将所有进程（包括自己）的数据合并收集到某进程缓冲区（scatter逆操作）//数据拼接
   Reduce归约：对所有进程某数据进行op操作后的结果存入某进程 //逻辑运算


9. MPI_Safety：当MPI实现或使用不当时会导致程序在多进程环境下出现未定义行为，数据竞争，死锁或其他错误的现象


10. 调用MPI_SendRecv和分别调用Send&Recv有何区别：
    调用MPI_SendRecv同时完成发送与接收，操作都是原子的，避免死锁
    分别调用可能引发死锁

 