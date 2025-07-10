# Concurrency and Synchronization

Operating systems must manage multiple processes or threads running concurrently. Some processes are **independent** (they do not interact) while others are **cooperating** (they work together on a task). Regardless, concurrent processes often **share resources** (CPU, memory, I/O) or communicate, which can lead to conflicts or the need for coordination. The OS must provide mechanisms for *synchronization* (ordering or coordinating tasks) and *mutual exclusion* (preventing conflicting accesses) so that shared resources are used safely.

A classic problem in concurrency is the **critical section**: a section of code that accesses a shared resource and must not be executed by more than one thread at a time. A **race condition** occurs when two or more threads/processes access shared data simultaneously and the final outcome depends on the unpredictable timing of their execution. To prevent races, only one thread may enter its critical section at a time – this property is called **mutual exclusion**.

## Synchronization Primitives

To implement mutual exclusion and synchronization, operating systems provide synchronization primitives – special objects or operations that threads can use. Common primitives include **mutex locks**, **semaphores**, and **condition variables**.

- **Mutex (Mutual Exclusion Lock):**  
  A mutex is a lock that enforces mutual exclusion by allowing at most one thread to hold it at a time.  
  ```c
  #include <pthread.h>

  pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
  int counter = 0;

  void* increment(void* arg) {
      pthread_mutex_lock(&lock);
      // Critical section: only one thread here at a time
      counter++;
      pthread_mutex_unlock(&lock);
      return NULL;
  }
  ```

- **Semaphore:**  
  A semaphore is a synchronization tool, represented by a nonnegative integer count, that can control access to a resource pool.  
  ```c
  #include <semaphore.h>
  sem_t sem;

  // Initialize semaphore to 1 (binary semaphore)
  sem_init(&sem, 0, 1);

  void critical_section() {
      sem_wait(&sem);   // P operation (wait)
      // Critical section begins
      // ...
      sem_post(&sem);   // V operation (signal)
  }
  ```

- **Condition Variables:**  
  A condition variable is used with a mutex to block a thread until a condition becomes true.  
  ```c
  pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
  pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
  int ready = 0;

  void* worker(void* arg) {
      pthread_mutex_lock(&lock);
      while (!ready) {
          pthread_cond_wait(&cond, &lock);
      }
      // ... do work ...
      pthread_mutex_unlock(&lock);
      return NULL;
  }

  void signaler() {
      pthread_mutex_lock(&lock);
      ready = 1;
      pthread_cond_signal(&cond);
      pthread_mutex_unlock(&lock);
  }
  ```

## Classic Synchronization Problems

- **Producer-Consumer (Bounded Buffer):**  
  ```c
  #define N 10
  int buffer[N], count = 0;
  pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
  sem_t empty, full;

  sem_init(&empty, 0, N);  // N empty slots initially
  sem_init(&full, 0, 0);   // 0 full slots initially

  void* producer(void* arg) {
      int item;
      while (produce_item(&item)) {
          sem_wait(&empty);
          pthread_mutex_lock(&mutex);
          buffer[count++] = item;
          pthread_mutex_unlock(&mutex);
          sem_post(&full);
      }
  }

  void* consumer(void* arg) {
      int item;
      while (1) {
          sem_wait(&full);
          pthread_mutex_lock(&mutex);
          item = buffer[--count];
          pthread_mutex_unlock(&mutex);
          sem_post(&empty);
          consume_item(item);
      }
  }
  ```

## CPU Scheduling

The CPU scheduler decides **which thread runs on the CPU** at any given time. Key scheduling **criteria** include:
- **CPU Utilization:** keep the CPU busy.
- **Throughput:** number of processes completed per unit time.
- **Turnaround Time:** total time from submission to completion.
- **Waiting Time:** total time spent in the ready queue.
- **Response Time:** time from submission until first response.
- **Fairness/Priority:** respect process priorities.

### Scheduling Algorithms

1. **First-Come, First-Served (FCFS):**  
   ```c
   queue ready_queue;  // FIFO queue
   while (true) {
       if (!ready_queue.empty()) {
           process = ready_queue.front();
           ready_queue.pop();
           run(process);  // run to completion or blocking
       } else {
           idle_cpu();
       }
   }
   ```

2. **Shortest-Job-First (SJF):**  
   Select the process with the smallest next CPU burst (ideal if burst lengths are known).

3. **Priority Scheduling:**  
   Run the highest-priority ready process; may be preemptive or nonpreemptive.

4. **Round-Robin (RR):**  
   ```c
   queue ready_queue;
   int time_quantum = 10; // ms
   while (true) {
       if (!ready_queue.empty()) {
           process = ready_queue.front();
           ready_queue.pop();
           run_for_time_slice(process, time_quantum);
           if (!process.is_done()) {
               ready_queue.push(process);
           }
       } else {
           idle_cpu();
       }
   }
   ```

5. **Multilevel Feedback Queues:**  
   Multiple queues with different priorities and time quanta; processes move between queues based on behavior.

### Multiprocessor Scheduling (Brief)

On multiprocessor systems, schedulers handle global or per-CPU queues and consider load balancing and cache affinity.

## Deadlocks

A **deadlock** occurs when processes are blocked, each waiting for a resource held by another. Four conditions are necessary:
1. **Mutual Exclusion**
2. **Hold and Wait**
3. **No Preemption**
4. **Circular Wait**

### Resource-Allocation Graph (RAG)

A graph with process and resource nodes; a cycle implies deadlock in single-instance resources.

### Deadlock Prevention

Eliminate one of the four conditions (e.g., impose resource ordering to avoid circular wait).

### Deadlock Avoidance (Banker’s Algorithm)

Grant requests only if the system remains in a safe state where every process can complete.

### Deadlock Detection and Recovery

Detect cycles and recover by process termination or resource preemption.

### Starvation and Livelock

Starvation: some process waits indefinitely; addressed by aging.  
Livelock: processes change state without making progress.
