# 并行程序设计笔记总结

## Chapter4.Pthreads进行共享内存编程
### 预备知识
- 线程：轻量级，共享进程资源，使用POSIX线程库
- 进程：重量级，独立内存空间，需要IPC（管道、共享内存等）通信
- 共享内存：所有线程共享同一片地址空间，通过全局变量通信
- 分布式内存：进程拥有独立内存（MPI）需要显式通信
- Pthreads的API只有在支持POSIX的系统上才有效
- Pthreads是C语言库，支持C/C++
  
### Pthreads编译与运行
```bash
gcc -pthread program.c -o program
./program
```
### Pthreads实现
**准备工作**
- 头文件&线程函数签名
```c
#include<pthread.h>
void* thread_function(void* arg); //必须返回void* 参数为void*
```

**创建线程**
```c
pthread_t thread_id;
pthread_create(&thread_id, NULL, thread_function, (void*) arg);
// thread_id: 存储线程标识符
// NULL: 使用默认线程属性
// arg: 传递给线程函数的参数，强制转换为void*
```
**运行线程**
- 线程执行：线程函数thread_function开始并发执行
- 主线程和其他线程并行运行

**停止线程**
- 等待线程结束
```c
pthread_join(thread_id,NULL); //阻塞，直到线程结束
```
- 线程退出
```c
pthread_exit(NULL); //在线程函数内调用
```

**例**
```c
#include<studio.h>
#include<pthread.h>

void* print_hello(void* arg){ //参数void*,返回void*
    printf("Hello\n");
    pthread_exit(NULL); //函数内调用，线程退出
}

int main(){
    pthread_t tid; //创建线程
    pthread_create(&tid,NULL,print_hello,NULL); //线程标识符，默认属性，线程函数，参数
    pthread_join(tid,NULL); //等待线程结束
    return 0;
}
```

### 矩阵-向量乘法
- 并行化思路：将矩阵分块，每个线程计算一部分行与向量的乘积
- 关键代码
```c
void* matvec(void* arg){
    int thread_id=*(int*)arg;
    for(int i=thread_id; i<rows; i+=num_threads){
        result[i]=0;
        for(int j=0; j<cols; j++){
            resulrs[i]+=matrix[i][j]* vector[j];
        }
    }
    pthread_exit(NULL);
}

int main(){
    pthread_t tid;
    pthread_create(&tid,NULL,matvec,NULL);
    pthread_join(tid,NULL);
    return 0;
}
```

### 临界区
- 问题：多个线程同时访问共享资源（如全局变量）导致竞态条件（Race Condition）
- 示例
```c
int counter=0;
void* increment(void* arg){
    for(int i=0;i<10000;i++)counter++; //counter++非原子操作
    pthread_exit(NULL);
}
// 结果可能小于预期（counter++非原子操作）
```
- 判断临界区
  1. 共享资源：多个线程同时访问变量或数据结构
  2. 非原子操作：对共享资源的操作可能被线程调度打断

- 判断原子操作：单条不可分割指令
- 非原子操作：任何包含多个步骤的操作（i++,a=b+c等）；对复杂数据结构（链表插入，动态数组扩容）的操作
  
### 互斥量Mutex
- 作用：保护临界区，确保同一时间只有一个线程访问共享资源
- 调用
```c
//初始化互斥量（在线程操作函数外初始化，全局互斥锁）
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
//加锁与解锁
pthread_mutex_lock(&mutex);
pthread_mutex_unlock(&mutex);
//销毁互斥量（在所有线程操作结束后再销毁）
pthread_mutex_destory(&mutex);
```
- 对于**临界区**中错误操作修复：
```c
//原错误操作：counter++非原子操作：
int counter=0;
void* increment(void* arg){
    for(int i=0;i<10000;i++)counter++; //counter++非原子操作
    pthread_exit(NULL);
}
```
```c
//添加互斥锁修正
#include<pthread.h>

pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER //创建全局互斥锁，所有线程共享
// pthread_mutex_init();
int counter =0;

void* increment(void& arg){
    for(int i=0;i<10000;i++){
        pthread_mutex_lock(&mutex);
        counter++; //非原子操作前后加锁解锁
        pthread_mutex_unlock(&mutex);
    }
    pthread_exit(NULL);
}

int main(){
    pthread_t tid1,tid2;
    pthread_create(&tid1, NULL, increment, NULL);
    pthread_create(&tid2, NULL, increment, NULL);
    pthread_join(tid1, NULL);
    pthread_join(tid2, NULL);

    pthread_mutex_destory(&mutex); //主函数中，所有线程操作结束后再销毁锁
    return 0;
}
```

### 忙等待
- 定义：线程通过循环不断检查条件（浪费CPU资源）
- 示例
```c
while(flag == 0) //忙等待直到flag为1
```
- 缺点：CPU占用率高，效率低

### 生产者-消费者模型
- **信号量**：计数器，控制对共享资源的访问
  - sem_init：初始化
  - sem_wait（P操作）：申请资源
  - sem_post（V操作）：释放资源
  
- **生产者-消费者模型**
- 核心原理：解决多线程协同处理共享数据问题，通过缓冲区解耦生产者和消费者的执行速度差异
- 生产者生成数据放入缓冲区
- 消费者从缓冲区取出数据并进行处理
- 缓冲区作为中间媒介，平衡生产者与消费者的速度差异
```c
sem_t empty, full;
sem_init(&empty, 0, BUFFER_SIZE); //初始化空缓冲区数量
sem_init(&full,0, 0)； //初始化满缓冲区数量

//生产者
sem_wait(&empty);   //等待空位
enqueue(item);      //生产者将数据插入队列尾部
sem_post(&full);    //通知消费者

//消费者
sem_wait(&full);    //等待数据
item = dequeue();   //消费者从队列头部取出数据
sem_post(&empty);   //释放空位
```

### 路障与条件变量
**路障**
- 作用：确保所有线程到达某一点后再继续执行，实现同步
- Pthread实现
```c
pthread_barrier_t barrier
pthread_barrier_init(&barrier, NULL, num_thread);
pthread_barrier_wait(&barrier); //线程阻塞直到全部到达
```
**忙等待+互斥量**：浪费CPU资源

**信号量实现路障**：复杂易出错，优先选择条件变量 

**条件变量**
- 作用：线程等待特定条件成立（避免忙等待）
- 使用步骤
```c
pthread_mytex_t mutex = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
//等待条件
pthread_mutex_lock(&mutex);
while(condition == false){
    pthread_cond_wait(&cond, &mutex); //释放mutex并阻塞
}
//通知条件成立
pthread_cond_signial(&cond); //唤醒一个线程
pthread_cond_broadcast(&cond); //唤醒所有线程
```

## 读写锁
- 特点：多个线程可同时读，写操作独占
- API
```c
pthread_rwlock_t lock;
pthread_rwlock_rdlock(&lock);   //读锁
pthread_rwlock_wrlock(&lock);   //写锁
pthread_rwlock_unlock(&lock);   //释放锁
```
## 缓存、线程安全性
- 缓存一致性：多核CPU中，线程可能读取过期的缓存数据
- 线程安全函数：rand()非线程安全，需用rand_r()或加锁

## Question
1. **pthreads中条件变量的用途是什么？**
   条件变量是一种线程同步机制，线程等待特定条件成立而避免忙等待；让线程在某个条件不满足时主动阻塞，在条件满足时被唤醒继续执行

2. **为什么pthreads中条件变量总与一个mutex相关联**
   条件变量等待操作pthread_and_wait()执行以下三步：释放关联Mutex;阻塞线程等待条件变量信号；被唤醒后重新获取Mutex;
   如果没有Mutex线程无法安全地释放和重新获取锁，导致共享数据被破坏

3. **如何使用pthreads实现一个barrier**
   方法一：busy_waiting+互斥锁实现
   方法二：信号量实现
   方法三：互斥锁+条件变量实现（推荐）
        使用计数器记录已到达线程数
        互斥锁保护计数器
        当计数器未到达总线程数时，线程通过条件变量等待
        最后一个线程到达时唤醒所有线程
    
4. **多个线程同时并行操作一个链表会出现什么问题**
   引发一系列线程安全问题，主要包括竞争关系、数据不一致性、程序崩溃
   - 插入+删除竞争
   - 删除+查询竞争
  
5. **线程之间如何实现数据传递？**
   通过访问共享变量或数据结构（全局变量、堆内存、队列）传递数据

6. **Busy-waiting与Mutex在实现多线程对临界区访问时有何区别？哪个效率高**
   - Busy-waiting:低延迟高CPU占用适合极端操作
   - Mutex:高延迟减少CPU占用，适合通用场景

7. 互斥锁与信号量区别
|特性|Mutex|Semaphore|
|---|-----|--------|
|用途|保护共享资源的独占访问（临界区）|控制资源数量或线程/进程的同步|
|持有者|必须由加锁的线程解锁|可由任意线程释放|
|计数|0（未锁）1（已锁）|任意非负整数（允许多个访问）|
|线程阻塞|竞争失败的线程阻塞|计数<=0线程阻塞|
|典型场景|保护全局变量、数据结构|限流、生产者-消费者|
