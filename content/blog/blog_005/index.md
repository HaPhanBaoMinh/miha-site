---
title: "CPU quota: cpu_limit 2000Mi is 2 core ?"
description:
author: MinhBoo Okk
date: 2026-04-20T22:00:00+07:00
draft: false
tags:
  - docker
  - linux
  - kubernetes
categories:
  - Linux
hideSummary: false
summary:
---
# 1. CPU switch context

First, I want everyone to understand how the CPU handles processes. Assume we have a single CPU with one core. In principle, at any given moment, only one process can use that CPU core for computation.

That means processes must be scheduled to take turns using the CPU, and the `Linux kernel scheduler` is responsible for deciding that schedule, including how long each process gets to run.

As for how the `Linux kernel scheduler` determines time slices and process priority, that is a deeper topic and something we will cover in a separate blog post.

![blog_005](images/01.jpeg)

As shown in the diagram, each process is only allowed to use the CPU for a limited window of time. Once that `time slice` is over, the CPU is handed off to another process. This actions is what we call a `context switch`.

These switches happen so quickly that users often perceive the system as running many tasks in parallel, even though a single CPU core can only execute one process at a time.

Như chúng ta đã thấy, không có một process nào thực sự được gán "1 core" hay "2 core" cả. Vậy khi các bạn dùng k8s hoặc container thường sẽ cho phép set cpu_limit là 1 core, 2 core. Vậy bản chất nó là gì, nó đang limit điều gì ?

# 2. CPU Time
Đây chính là keyword cần chúng ta tìm hiểu:

`wall-clock time (elapsed time)`: tổng thời gian process đã chạy

`cpu time`: tổng thời gian mà process đó sử dụng **CPU**
![blog_005](images/02.jpeg)

**Lưu ý:** Cả `wall-clock time` và `cpu time` đều không phải `time slice` được đề cập ở phần 1,  ở phần 1 chỉ là tôi muốn các bạn nắm được tổng quan việc cách mà các process sử dụng CPU.

**Ví dụ:**  
- thread A: sử dụng CPU 10ms  
- thread B: sử dụng CPU 50ms  
  
=> Tổng CPU time của process = 60ms  
  
**Lưu ý:**  
CPU time là tổng thời gian thực thi trên tất cả CPU cores (không tính thời gian khác).  
Nếu các thread chạy song song trên nhiều cores, CPU time vẫn được cộng dồn.  
  
CPU time được chia thành nhiều phần:
- `user time`: thời gian thực thi trong user space (application)  
- `system time`: thời gian CPU dùng trong kernel space để xử lý các system call, I/O,...
- `nice_time`
- `irq_time` 
- `softirq_time` 
- `steal_time`

Ok so if you used monitor tools, it have 1 chart about % CPU vậy nó chính xác là gì?

```
CPU % = CPU time / Wall clock time
```

Ref docs:
- [Source for htop calculation](https://github.com/htop-dev/htop/blob/37d30a3a7d6c96da018c960d6b6bfe11cc718aa8/linux/Platform.c#L324)
- [CPU Utilization - A useful metric?](https://www.green-coding.io/case-studies/cpu-utilization-usefulness/#:~:text=CPU%20utilization%20is%20defined%20as,on%20the%20idle%20thread.%E2%80%9D)
# 3. `CPU quota` in Linux
Nếu các bạn trước đây đã từng sử dụng qua `Docker` hoặc `K8s`, mỗi container chạy lên chúng ta đều có thể quy định được số CPU limit của từ container/pod, vậy thực chất là nó đang limit điều gì ở đây. 

Trước tiên chúng ta cần quan tâm tới 2 giá trị sau:
- `period`: Là khoảng thời gian lặp lại mà kernel dùng để kiểm soát CPU usage
- `quota`: CPU time tối đa được dùng trong chu kỳ `period` time đó.

**Ví dụ:**
- `period = 100000 µs = 100 ms`
- `quota = 50000 µs = 50 ms`

Mỗi `100ms`, process được dùng CPU tối đa `50ms`. Hiểu đơn dãn hơn là cứ sau `100ms` thì process sẽ được cấp `50ms` thời gian sử dụng CPU (CPU time).

**Cách tính vCore:**
```
vCPU (cores) = quota / period
```

**Ví dụ:**
- 200ms / 100ms = 2 core

Khoan nhưng trong `100ms period` làm sao dùng CPU hết `200ms`? Đây là lúc process dùng nhiều hơn 1 core tại 1 thời điểm. Giả sử process đó có 2 thread mỗi thread sử dụng 1 core CPU, mỗi thread chạy `100ms` thì process đó vẫn dùng hết `200ms` ~ **100% CPU** được cấp. It cool !

**CPU throttle:**

Hãy tưởng tượng rằng 1 tuần vợ bạn sẽ cho bạn `50 đồng` để tiêu vặt, và nếu mới đến giữa tuần thôi mà bạn đã hết tiền rồi thì những ngày sau đó bạn sẽ không được tiêu gì nữa! **CPU throttle** cũng hoạt động như vậy! 

Nếu chưa hết thời gian `period` để được cấp `quota` mới mà process đã sử dụng hết `quota` thì trong khoảng thời gian sau đó process sẽ không được sử dụng CPU nữa. Khi đó CPU sẽ throttle, bắt buộc phải chờ tới khi tới `period` time để được cấp `quota` mới.
![blog_005](images/04.jpeg)
**Ví dụ:**  
- Nếu container dùng hết 50ms CPU chỉ trong 20ms đầu,  thì 80ms còn lại trong period sẽ bị block và không được chạy.

**Đó chính là cách mà chúng ta limit CPU resource!**

Chúng ta cùng khởi tạo 1 container như sau:
```sh
docker run -it --rm --cpus="1.0" --entrypoint /bin/bash ghcr.io/colinianking/stress-ng
```

Chúng ta có thể lấy thông tin `period` và `quota` của container:
```
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
![[miha-site/content/blog/blog_005/images/03.png]]

```
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

```
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
Như các bạn đã thấy thì khi strees test với 2 cpu thì process đó bị  có 100 lần throttle, vậy tại sao lại là 100 lần. 

```
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
# 5. Những thông tin khác
# 6. Document tham khảo 