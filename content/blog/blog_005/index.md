---
title: "CPU quota: cpu_limit: 2000m is 2 core ?"
description: Understand what CPU limits really mean in Docker and Kubernetes through Linux CPU time, quota/period, and throttling behavior.
author: MinhBoo
date: 2026-04-20T22:00:00+07:00
draft: false
tags:
  - docker
  - linux
  - kubernetes
categories:
  - Linux
hideSummary: false
summary: This post explains how Linux schedules CPU time, why CPU utilization differs from wall-clock time, and how cgroup quota/period translates to vCPU limits. With stress-ng examples, it shows why workloads can be throttled and lose performance even when monitoring charts look acceptable.
cover:
  image: images/03.jpeg
  alt: ""
  caption: ""
  relative: false
  hidden: false
---
# 1. CPU switch context

First, I want everyone to understand how the CPU handles processes. Assume we have a single CPU with one core. In principle, at any given moment, only one process can use that CPU core for computation.

That means processes must be scheduled to take turns using the CPU, and the `Linux kernel scheduler` is responsible for deciding that schedule, including how long each process gets to run.

As for how the `Linux kernel scheduler` determines time slices and process priority, that is a deeper topic and something we will cover in a separate blog post.

![blog_005](images/01.jpeg)

As shown in the diagram, each process is only allowed to use the CPU for a limited window of time. Once that `time slice` is over, the CPU is handed off to another process. This actions is what we call a `context switch`.

These switches happen so quickly that users often perceive the system as running many tasks in parallel, even though a single CPU core can only execute one process at a time.

As we have seen, no process is actually assigned to a specific “1 core” or “2 cores.” However, when using **Kubernetes** or **containers**, we often set a **CPU limit** such as 1 core or 2 cores. So what does this really mean? What exactly is being limited?

# 2. CPU Time
These are the key concepts we need to understand:

`wall-clock time (elapsed time)`: the total time the process has been running

`cpu time`: the total time the process has actually **used the CPU**

![blog_005](images/02.jpeg)

Note: Both wall-clock time and CPU time are not the same as the time slice mentioned in the previous section. The purpose of that section was only to give a high-level understanding of how processes use the CPU.

Example:
- **Thread A:** uses 10ms of CPU time
- **Thread B:** uses 50ms of CPU time

=> Total CPU time of the process = 60ms

Note:
CPU time is the total execution time across all CPU cores (excluding waiting time).
If multiple threads run in parallel on different cores, their CPU time is still accumulated.

CPU time can be broken down into several components:
- `user_time:` time spent executing in **user space** (application code)
- `system_time: `time spent in **kernel space** (system calls, I/O handling, etc.)
- `nice_time`
- `irq_time`
- `softirq_time`
- `steal_time`

So, when you use monitoring tools and see a "% CPU" chart, what does it actually represent?

```
CPU % = CPU time / Wall clock time x vcore
```

Ref docs:
- [Source for htop calculation](https://github.com/htop-dev/htop/blob/37d30a3a7d6c96da018c960d6b6bfe11cc718aa8/linux/Platform.c#L324)
- [CPU Utilization - A useful metric?](https://www.green-coding.io/case-studies/cpu-utilization-usefulness/#:~:text=CPU%20utilization%20is%20defined%20as,on%20the%20idle%20thread.%E2%80%9D)
# 3. `CPU quota` in Linux
If you have used Docker or Kubernetes before, you may know that we can set a CPU limit for each container or pod. But what does this limit actually control?

First, we need to understand two important values:
- `period`: the time window that the kernel uses to control **CPU usage**
- `quota`: the maximum amount of **CPU time** that can be used within one period

**Example:**
- `period = 100000 µs = 100 ms`
- `quota = 50000 µs = 50 ms`

This means that in every `100ms`, the process can use up to `50ms` of **CPU time**.  In simpler terms, every `100ms`, the process is allowed to use `50ms` of **CPU time**.

**How to calculate vCPU (CPU cores):**
```
vCPU (cores) = quota / period
```

**Example:**
- 200ms / 100ms = 2 core

Wait, how can a process use `200ms` of CPU time within a `100ms` period?

This happens when the process runs on multiple CPU cores at the same time.

For example:
If a process has 2 threads and each thread runs on a separate CPU core:
- Thread A uses **50ms** of CPU time
- Thread B uses **50ms** of CPU time

=> Total CPU time = **100ms**

This means the process has already used 100% of its CPU quota, even though only **50ms** of **wall-clock time** has passed.

This is important:
**Using multiple cores does not increase the CPU quota. It only makes the quota get used up faster.**

**CPU throttle:**
Imagine you are given a fixed budget of **50 dollars** each week.
If you spend it all in the **first few days,** you won’t be able to spend anything until the next week.
**CPU throttling** works in a similar way: 1 the `CPU quota` is used up, the process must wait until the next `period` to continue.

If a process consumes its entire `CPU quota` before the end of the current `period`, it will be prevented from running for the rest of that period.
This is known as **CPU throttling.** The process must wait until the next period begins, when the **quota is reset**, before it can run use  CPU again.
![blog_005](images/04.jpeg)
**Example:**
- If a container uses up `50ms` of **CPU time** within the first `20ms`, then it will be blocked for the remaining `80ms` of the period.

So, when we set a CPU limit in **Kubernetes** or **containers**, we are essentially limiting the **CPU quota.**

**That is how we limit CPU resources!**

Let create 1 container:
```sh
docker run -it --rm --cpus="1.0" --entrypoint /bin/bash ghcr.io/colinianking/stress-ng
```

We can check `period` and `quota` of container:
```sh
---
cat /sys/fs/cgroup/cpu.max

OUTPUT:
100000 100000

#Note:
quota = 100000 µs = 100 ms
period = 100000 µs = 100 ms

---
cat /sys/fs/cgroup/cpu.stat

OUTPUT:
usage_usec 53558 # CPU time
user_usec 34843 # CPU time in user space
system_usec 18715 # CPU time in kernel space
nice_usec 0
core_sched.force_idle_usec 0
nr_periods 21 # The total number of CPU periods that have elapsed
nr_throttled 0 # The number of times the container was CPU throttled
throttled_usec 0 # The total time the container was throttled (in microseconds)
nr_bursts 0
burst_usec 0
```
![blog_005](images/03.png)

```sh
---
stress-ng --cpu 1 --timeout 10s
cat /sys/fs/cgroup/cpu.stat

OUTPUT:
usage_usec 10025053
user_usec 9996608
system_usec 28444
nice_usec 0
core_sched.force_idle_usec 0
nr_periods 109
nr_throttled 0
throttled_usec 0
nr_bursts 0
burst_usec 0
```
![blog_005](images/05.png)

```sh
---
stress-ng --cpu 2 --timeout 10s
cat /sys/fs/cgroup/cpu.stat

OUTPUT:
usage_usec 20050187
user_usec 20012132
system_usec 38055
nice_usec 0
core_sched.force_idle_usec 0
nr_periods 218
nr_throttled 100 # Throttle 100 time
throttled_usec 9911798 # ~9.9s
nr_bursts 0
burst_usec 0
```
![blog_005](images/06.png)
As you can see, when we run a stress test with 2 CPU workers, the process is throttled around **100 times**. But why does this number come out to be about **100**?

```sh
stress-ng --cpu 2 --timeout 10s
---
quota  = 100ms
period = 100ms
runtime ≈ 10s
nr_throttled = 100
---
nr_throttled ≈ 10s / 0.1s = 100
throttled_usec ≈ 9.9s
```
# 4. Monitor chart: 50% CPU but why performance drop

We are going to build an application, but we will limit thread of process. We will give it 2 vCores and run a load test. Let's see what happens!

We will use:
- JS (JavaScript) 
- Docker 
- Wrk: to send traffic 
- cAdvisor: for monitoring

```js
# server.js
const http = require('http');
const crypto = require('crypto');

function cpuWork() {
  return new Promise((resolve, reject) => {
    crypto.pbkdf2('password', 'salt', 200000, 64, 'sha512', (err, key) => {
      if (err) return reject(err);
      resolve(key);
    });
  });
}

const server = http.createServer(async (req, res) => {
  const start = Date.now();

  try {
    await cpuWork();
    const duration = Date.now() - start;

    res.writeHead(200, { 'Content-Type': 'text/plain' });
    res.end(`Done in ${duration} ms\n`);
  } catch (err) {
    res.statusCode = 500;
    res.end('Internal Server Error\n');
  }
});

server.listen(3000, () => {
  console.log('Server running on port 3000');
  console.log('UV_THREADPOOL_SIZE =', process.env.UV_THREADPOOL_SIZE || '(default: 4)');
});
```
From what I understand, when **Node.js** runs, it uses one main thread and a thread pool. 
We can limit the size of this thread pool using the **UV_THREADPOOL_SIZE** variable.
![blog_005](images/08.png)
Doc refs:
- [Thread pool work scheduling](https://docs.libuv.org/en/v1.x/threadpool.html)

```yaml
# docker-compose.yml
services:
  cadvisor:
    image: ghcr.io/google/cadvisor:0.54.1
    container_name: cadvisor
    privileged: true
    ports:
      - "8080:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /sys:/sys:ro
      - /home/docker:/home/docker:ro
      - /dev/disk:/dev/disk:ro
      - /sys/fs/cgroup:/sys/fs/cgroup:ro
    devices:
      - /dev/kmsg
    restart: unless-stopped
```

```Dockerfile
# Dockerfile
FROM node:20-alpine
WORKDIR /app
COPY server.js .
CMD ["node", "server.js"]
```

## **Case 1: Limit application thread**
Build and start docker container:
```sh 
docker build -t cpu-quota .

docker run -d --cpus="2.0" --name cpu-quota-1  \
  -e UV_THREADPOOL_SIZE=1 \
  -p 3000:3000 \
  cpu-quota

docker compose up # To start cAdvisor
```

Open **cAdvisor** at `localhost:8080` and find the `cpu-quota-1` container that we created.
![blog_005](images/14.png)

Checking the metrics before the load test:
![blog_005](images/09.png)
![blog_005](images/12.png)
Exactly as we configured, this container is limited to 2 vCores.

Load test with `wrk`
```sh 
wrk -t100 -c500 -d60s http://localhost:3000

# t100: Create 100 thread to send request
# c500: Create 500 TCP connrection in 1 time 
# d60s: Test duration  
```
![blog_005](images/10.png)

**Test result:** 
![blog_005](images/15.png)
As you can see, even though we gave the container 2 vCores, it only uses 1 vCore no matter how hard we push the load test.

Ok, that is **cAdvisor**'s calculation. Now, let's verify it ourselves by reading the `cpu.stat` file.
![blog_005](images/11.png)
`period` = 100ms ,`quota`  = 200ms        
`vcore` = `quota` / `period` = 200 / 100 = 2 (vcore)

`wall-clock time` = `nr_periods` * `period` 
= 783 * 100ms 
= 78.3 s 

`cpu_time` = usage_usec = 78.1s 

`%CPU` = `cpu_time` / (`wall-clock * vcore`) 
= 78.1 / 78.3 * 2 
= **~50%** 
## **Case 2: Increase limit application thread**
```sh 
docker run -d --cpus="2.0" \
  -e UV_THREADPOOL_SIZE=4 \
  -p 3000:3000 \
  cpu-quota
```

Before load test:
![blog_005](images/12.png)
Đúng như những gì ta đã cấp, container này được limit 2vcore
![blog_005](images/16.png)

Load test with `wrk`
```sh 
wrk -t100 -c500 -d60s http://localhost:3000

# t100: Create 100 thread to send request
# c500: Create 500 TCP connrection in 1 time 
# d60s: Test duration   
```

**Test result:** 
![blog_005](images/17.png)

Ok, that is **cAdvisor**'s calculation. Now, let's verify it ourselves by reading the `cpu.stat` file.
![blog_005](images/13.png)

`period` = 100ms, `quota` = 200ms  
`vcore` = `quota` / `period` = 200 / 100 = 2 (vcore)  
  
`wall-clock time` = `nr_periods` * `period`  
= 796 × 100ms  
= 79.6s  
  
`cpu_time` = `usage_usec` ≈ 158.12s  
  
`%CPU` = `cpu_time` / `(wall-clock × vcore)`
= 158.12 / (79.6 × 2)  
= **~ 99.3%**
## Conclusion
From the 2 cases above, we can draw a hard truth: Giving more CPUs to a container doesn't mean the application will use them all

Beware of monitoring blind spots!