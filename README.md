```
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/mman.h>
#include <semaphore.h>
#include <sys/wait.h>

#define BUFFER_SIZE    10
#define NUM_PRODUCERS   3
#define NUM_CONSUMERS   2
#define PRODUCE_COUNT   20  // 每个生产者生产的项目数
#define CONSUME_COUNT   ((NUM_PRODUCERS * PRODUCE_COUNT) / NUM_CONSUMERS)

typedef struct {
    int buffer[BUFFER_SIZE];
    int in;
    int out;
    sem_t mutex;
    sem_t empty;
    sem_t full;
} shared_data;

int main() {
    // 1. 在匿名映射区创建共享数据
    shared_data *shm = mmap(NULL, sizeof(shared_data),
                            PROT_READ | PROT_WRITE,
                            MAP_SHARED | MAP_ANON,
                            -1, 0);
    if (shm == MAP_FAILED) {
        perror("mmap");
        exit(1);
    }

    // 2. 初始化索引和信号量
    shm->in = shm->out = 0;
    sem_init(&shm->mutex, 1, 1);              // 互斥锁，初始值1
    sem_init(&shm->empty, 1, BUFFER_SIZE);    // 空槽计数，初始为BUFFER_SIZE
    sem_init(&shm->full,  1, 0);              // 满槽计数，初始为0

    // 3. 创建生产者进程
    for (int p = 0; p < NUM_PRODUCERS; ++p) {
        pid_t pid = fork();
        if (pid < 0) {
            perror("fork producer");
            exit(1);
        }
        if (pid == 0) {
            // 子进程（生产者）
            for (int i = 0; i < PRODUCE_COUNT; ++i) {
                int item = p * 100 + i;  // 生产一个示例项目
                sem_wait(&shm->empty);
                sem_wait(&shm->mutex);

                shm->buffer[shm->in] = item;
                printf("Producer %d produced item %d at index %d\n",
                       p, item, shm->in);
                shm->in = (shm->in + 1) % BUFFER_SIZE;

                sem_post(&shm->mutex);
                sem_post(&shm->full);

                usleep(100000); // 模拟生产时间
            }
            exit(0);
        }
    }

    // 4. 创建消费者进程
    for (int c = 0; c < NUM_CONSUMERS; ++c) {
        pid_t pid = fork();
        if (pid < 0) {
            perror("fork consumer");
            exit(1);
        }
        if (pid == 0) {
            // 子进程（消费者）
            for (int i = 0; i < CONSUME_COUNT; ++i) {
                sem_wait(&shm->full);
                sem_wait(&shm->mutex);

                int item = shm->buffer[shm->out];
                printf("Consumer %d consumed item %d from index %d\n",
                       c, item, shm->out);
                shm->out = (shm->out + 1) % BUFFER_SIZE;

                sem_post(&shm->mutex);
                sem_post(&shm->empty);

                usleep(150000); // 模拟消费时间
            }
            exit(0);
        }
    }

    // 5. 父进程关闭映射后等待所有子进程
    for (int i = 0; i < NUM_PRODUCERS + NUM_CONSUMERS; ++i) {
        wait(NULL);
    }

    // 6. 清理信号量和共享内存
    sem_destroy(&shm->mutex);
    sem_destroy(&shm->empty);
    sem_destroy(&shm->full);
    munmap(shm, sizeof(shared_data));

    printf("All producers and consumers have finished.\n");
    return 0;
}
```
