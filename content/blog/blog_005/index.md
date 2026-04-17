---
title: "CPU quota: cpu_limit 2000Mi is 2 core ?"
description:
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
summary:
---
# 1. CPU switch context

First, I want everyone to understand how the CPU handles processes. Assume we have a single CPU with one core. In principle, at any given moment, only one process can use that CPU core for computation.

That means processes must be scheduled to take turns using the CPU, and the `Linux kernel scheduler` is responsible for deciding that schedule, including how long each process gets to run.

As for how the `Linux kernel scheduler` determines time slices and process priority, that is a deeper topic and something we will cover in a separate blog post.

![blog_005](images/01.jpeg)

As shown in the diagram, each process is only allowed to use the CPU for a limited window of time. Once that time slice is over, the CPU is handed off to another process. This actions is what we call a context switch.

These switches happen so quickly that users often perceive the system as running many tasks in parallel, even though a single CPU core can only execute one process at a time.

# 2. `CPU quota` in Linux
Nếu các bạn trước đây đã từng sử dụng qua `Docker` hoặc `K8s`, mỗi container chạy lên chúng ta đều có thể quy định được số CPU limit của từ container/pod, vậy thực chất là nó đang limit điều gì ở đây. 

Trước tiên chúng ta cần quan tâm tới 2 giá trị sau:
- `period`: 1 chu kì sử dụng CPU.
- `quota`: CPU time tối đa được dùng trong chu kỳ đó.

Ví dụ:
- `period = 100000us`
- `quota = 50000us`

Mỗi `100ms`, cgroup được dùng CPU tối đa `50ms`




# 3. Quick Demo
# 4. Những thông tin khác
# 5. Document tham khảo 