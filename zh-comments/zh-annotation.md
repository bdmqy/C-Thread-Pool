#### structure and data

**全局变量**

```c
static volatile int threads_keepalive;
static volatile int threads_on_hold;
```

​		二者均用来表示线程的状态，取值为0或1。threads_keepalive用来控制线程池中所有线程的终止。threads_keepalive=1表示线程可以继续运行下去。threads_on_hold默认为0，当其为1时表明线程被用户”休眠“（一直调用sleep()，直至threads_on_hold恢复为0）。

​		由于二者均不存在增（减）量操作，不需要互斥量进行“保护”。

**数据结构**

```c
typedef struct job{
	struct job*  prev;                   /* pointer to previous job   */
	void   (*function)(void* arg);       /* function pointer          */
	void*  arg;                          /* function's argument       */
} job;
```

​		任务用三元组表示，prev指针指向前一个任务，function为函数指针指向”任务函数“，arg为其参数。

```c
typedef struct bsem {
	pthread_mutex_t mutex;
	pthread_cond_t   cond;
	int v;
} bsem;
typedef struct jobqueue{
	pthread_mutex_t rwmutex;             /* used for queue r/w access */
	job  *front;                         /* pointer to front of queue */
	job  *rear;                          /* pointer to rear  of queue */
	bsem *has_jobs;                      /* flag as binary semaphore  */
	int   len;                           /* number of jobs in queue   */
} jobqueue;

```

​		任务队列已单向链表的形式实现，front指向队列的头，rear指向队尾（即指向最后一个任务）。len为队列长度。rwmutex用于限制对任务队列的访问。

​		has_jobs用来反映任务队列的是否为空。在任务队列中每添加一个任务或取走一个任务后任务队列不为空，就向线程发送信号（唤醒阻塞线程去执行任务）。has_jobs->v的取值为0或1，为1是表示任务队列非空。



```c
typedef struct thread{
	int       id;                        /* friendly id               */
	pthread_t pthread;                   /* pointer to actual thread  */
	struct thpool_* thpool_p;            /* access to thpool          */
} thread;


/* Threadpool */
typedef struct thpool_{
	thread**   threads;                  /* pointer to threads        */
	volatile int num_threads_alive;      /* threads currently alive   */
	volatile int num_threads_working;    /* threads currently working */
	pthread_mutex_t  thcount_lock;       /* used for thread count etc */
	pthread_cond_t  threads_all_idle;    /* signal to thpool_wait     */
	jobqueue  jobqueue;                  /* job queue                 */
} thpool_;
```

num_threads_alive“活着的”线程数量，num_threads_working正在执行任务的线程数量。当所有线程均结束后利用条件变量threads_all_idle发送信号。

**scheme**



#### functions

