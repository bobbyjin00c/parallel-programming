# 并行程序设计笔记总结

## Chapter5.OpenMP进行共享内存编程

### 预备知识
- 编译与运行
```bash
gcc -fopenmp program.c -o program
```
- 错误检查：使用环境变量OMP_DISPLAY_ENV=1显式OpenMP配置
```bash
export OMP_DISPLAY_ENV=1
./program
```

### 梯形积分法
- 原理：积分区间划分为多个梯形，多线程并行计算每个梯形面积后求和
- 代码示例
```c
#include<studio.h>
#include<omp.h>

double f(double x){ return x * x;}

int main(){
    double a=0.0, b=1.0;    //积分区间
    int n=100000;           //梯形数量
    double h= (b-a)/n;
    double sum =0.0;
    
    #pragma omp parallel for reduction(+:sum) //reduction子句确保sum的线程私有变量在并行结束后正确累加
    for(int i=0;i<n;i++){
        double x=a+i*h;
        sum+=(f(x)+f(x+h))*h/2;
    }
    return 0;
} 
```

### 变量作用域
- 共享变量：默认在并行区域内共享
- 私有变量：通过private子句指定，每个线程拥有独立副本
```c
int x=10;
#pragma omp parallel private(x)
{
    printf("%d",x); //未初始化，输出不稳定
}
```
### OpenMP编译指令
指导编译器将代码并行化，以#pragma omp 开头

1. #pragma omp parallel for reduction(+:sum)
- 功能：并行化一个循环，并将循环的迭代分配给多个线程执行
- reduction(+:sum): 对变量sum执行规约操作（求和），确保多线程操作sum时结果正确

2. #pragma omp parallel private(x)
- 功能：创建一个并行区域，并指定变量x为线程私有（private），即每个线程拥有独立的x副本 

3. 其他常见omp指令
|指令|功能|
|--|--|
|#pragma omp|开始|
|#pragma omp parallel|创建并行区域（不分配具体任务）|
|#pragma omp for|将循环迭代分配给线程（在parallel区域内使用）|
|#pragma omp sections|将代码块分配给不同线程执行|
|#pragma omp critical|所有未命名critical指令互斥（作用在全局匿名临界区|）
|#pragma omp critical(name)|仅与同名的critical(name)互斥（作用在命名临界区）|
|#pragma omp atomic|保证单条语句原子性（如x++）|
|#pragma omp barriwe|线程同步点（所有线程到达后才继续执行）|

- 简单操作：atomic 
- 复杂代码块：critical(name) 
- 全局唯一资源：critical

### 归约子句
- 用途：合并多个线程的局部效果（如求和、最大值）
```c
int sum=0;
#pragma omp parallel for reduction(+:sum)
for(int i=0;i<100;i++){
    sum+=i;
}
```
- reduction子句变量初始值
- （+）     初始值为0 ；私有副本初始为0，最终结果原始值+各线程和
- （*）     初始值为1 ；私有副本初始为1，最终结果为原始值*各线程乘积
- （&&）    初始值为1（true）
- （||）    初始值为0（false）
- （max）   初始值为对应数据类型最小值（如INT_MIN）
- （min）   初始值为对应数据类型最大值（如INT_MAX）

### parallel for指令
- 警告：循环必须无数组依赖性
- 错误示例：(存在依赖)
```c
int x[100]={0};
#pragma omp parallel for
for(int i=1;i<100;i++){
    x[i]=x[i-1]+1; //错误：x[i]依赖于x[i-1]
}
```
- 正确示例：（计算π值）
```c
double pi=0.0
#pragma omp parallel for reduction(+:pi)
for(int i=0;i<10000;i++){
    pi+=(i%2 == 0? 1.0 : -1.0) / (2*k + 1);
}
pi*=4;
```

### 排序
**冒泡排序**（不适合并行化）
```c
// 串行版本（无法直接并行化）
for (int i = 0; i < n; i++) {
    for (int j = 0; j < n-i-1; j++) {
        if (arr[j] > arr[j+1]) swap(&arr[j], &arr[j+1]);
    }
}
```
**并行奇偶排序**（部分可并行）
```c
#pragma omp parallel
{
    for (int phase = 0; phase < n; phase++) {
        if (phase % 2 == 0) {
            #pragma omp for
            for (int i = 0; i < n-1; i += 2) {
                if (arr[i] > arr[i+1]) swap(&arr[i], &arr[i+1]);
            }
        } else {
            #pragma omp for
            for (int i = 1; i < n-1; i += 2) {
                if (arr[i] > arr[i+1]) swap(&arr[i], &arr[i+1]);
            }
        }
    }
}
```
### 循环调度
**schedule子句**：控制迭代分配到线程的方式
|调度类型|行为|使用场景|
|---|---|---|
|statistic |提前均分块|迭代计算量均匀|
|dynamic|动态分配小块（默认1次迭代）|迭代计算量不均匀|
|guided|块大小逐渐减小|负载不平衡但可预测|
|runtime|通过环境变量OMP_SCHEDULE设置|灵活调整|

- 示例
```c
#pragma omp parallel for schedule(dynamic,10)
for(int i=0;i<100;i++){
    heavy_computation(i); //计算量不均匀的任务
}
```
### 生产者-消费者问题
- 实现要点：enqueue和dequeue需要锁保护
- 终止检测：通过标志位或特殊消息
- 代码框架
```c
#include <omp.h>
#include <queue>

std::queue<int> q;
omp_lock_t lock;

void producer() {
    for (int i = 0; i < 10; i++) {
        omp_set_lock(&lock);
        q.push(i);  // enqueue
        omp_unset_lock(&lock);
    }
    q.push(-1);  // 终止信号
}

void consumer() {
    while (true) {
        omp_set_lock(&lock);
        if (!q.empty()) {
            int item = q.front();  // dequeue
            q.pop();
            omp_unset_lock(&lock);
            if (item == -1) break;
            printf("Consumed: %d\n", item);
        } else {
            omp_unset_lock(&lock);
        }
    }
}

int main() {
    omp_init_lock(&lock);
    #pragma omp parallel sections
    {
        #pragma omp section
        producer();
        #pragma omp section
        consumer();
    }
    omp_destroy_lock(&lock);
    return 0;
}
```
### 同步机制对比

|机制|用途|示例|
|--|--|--|
|critical|保护临界区（自动加锁）|#pragma omp critical{x++;}|
|atomic|简单原子操作（+,-）|#pragma omp atomic x+=1|
|omp_lock_t|手动控制锁|omp_set_lock() omp_unset_lock() omp_destory_lock() omp_lock_t|

### 缓存与伪共享
- 伪共享：多个线程频繁修改同一缓存行的不同变量导致缓存失败
- 解决方法：填充或私有化变量
  示例：
```c
struct Data{
    int x;
    char padding[64];   //填充缓存行（通常64字节）
};
Data arr[10];
#pragma omp parallel for
for(int i=0;i<10;i++){
    arr[i].x++; //不同线程修改不同缓存行
}
```
### 线程安全性
- 不安全函数：rand(), 改用rand_r()或加锁
- 线程局部存储：通过threadprivate字句实现
  示例：
```c
int counter;
#pragma omp threadprivate(counter)
```

## Question
1. **直接在循环体外面加上# pragma omp parallel for来实现循环体并行化是否总可行？**
   OpenMP编译器不会检查使用parallel for指令并行化的循环中各迭代之间的依赖性
   如果一个循环中某些迭代结果依赖于其他迭代，那么该循环就无法被OpenMP正确并行化，可能引发竞争条件或错误结果

2. **#pragma omp parallel for 和 #pragma omp for有何区别谁的效率更高**
   区别主要在于并行区域的创建方式，效率取决于具体使用场景
   #pragma omp parallel for 隐式创建并行区域并在该区域内并行化循环；适用于直接并行化单个循环，无需额外并行操作
   #pragma omp for必须在显式的parallel区域使用，仅分配循环迭代，不创建新线程；适用于嵌套并行或多个循环共享同一并行区域
