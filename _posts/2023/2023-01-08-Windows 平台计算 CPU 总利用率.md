---
title: Windows 平台计算 CPU 总利用率
date: 2023-01-08 21:12
---

利用 [GetSystemTimes](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-getsystemtimes) 可以获得 Windows 系统的 Idle Time、 Kernel Time 和 User Time。Idle Time 是系统空闲的时间，也就是系统没有利用的时间。Kernel Time 是系统在内核模式下的执行时间。User Time 是系统在用户模式下的执行时间。由于 Kernel Time 包含了 Idle Time，因此 CPU 的总利用时间为 Kernel Time + User Time - Idle Time。间隔一段时间连续两次调用 GetSystemTimes，再利用时间差可以计算这段时间内的平均 CPU 总利用率。这个 API 返回的时间类型是 FILETIME，不能直接用其 dwHighDateTime 和 dwLowDateTime 成员相减。需要先将其转化为 uint64_t 再进行计算。下面是示例代码：

```c++
#include <Windows.h>
#include <iostream>
#include <thread>
#include <vector>

void do_busy_things()
{
    while (true) {
        volatile int sum = 0;
        for (volatile int i = 0; i < 10000; ++i) {
            sum += i;
        }
    }
}

void measure_cpu_usage()
{
    struct system_time_t
    {
        uint64_t idle_time, kernel_time, user_time;
        system_time_t operator-(system_time_t other) {
            return {
                .idle_time = idle_time - other.idle_time,
                .kernel_time = kernel_time - other.kernel_time,
                .user_time = user_time - other.user_time,
            };
        }
    };

    struct system_time_file_t
    {
        FILETIME idle_time, kernel_time, user_time;
        system_time_t to_uint64_t() {
            system_time_t time{};
            time.idle_time = ((uint64_t)idle_time.dwHighDateTime << 32) | idle_time.dwLowDateTime;
            time.kernel_time = ((uint64_t)kernel_time.dwHighDateTime << 32) | kernel_time.dwLowDateTime;
            time.user_time = ((uint64_t)user_time.dwHighDateTime << 32) | user_time.dwLowDateTime;
            return time;
        }
    };

    while (true) {
        system_time_file_t begin_time{}, end_time{};
        GetSystemTimes(&begin_time.idle_time, &begin_time.kernel_time, &begin_time.user_time);
        std::this_thread::sleep_for(std::chrono::seconds(1));
        GetSystemTimes(&end_time.idle_time, &end_time.kernel_time, &end_time.user_time);
        auto system_time = end_time.to_uint64_t() - begin_time.to_uint64_t();
        double cpu_usage_percent = 1.0 * (system_time.kernel_time + system_time.user_time - system_time.idle_time) / (system_time.kernel_time + system_time.user_time) * 100;
        std::cout << "cpu usage percent: " << cpu_usage_percent << "%" << std::endl;
    }
}

int main()
{
    //std::vector<std::thread> v(std::thread::hardware_concurrency());
    std::vector<std::thread> v(std::thread::hardware_concurrency() / 2);
    std::thread t0(measure_cpu_usage);

    for (auto& t : v) {
        t = std::thread(do_busy_things);
    }

    for (auto& t : v) {
        t.join();
    }

    t0.join();
}
```

这段代码启动了物理线程数一半的线程执行 do_busy_things 函数，再启动一个线程测量 CPU 总利用率。可以观察到 CPU 总利用率在 50% ~  52% 之间。打开任务管理器，显示 CPU 总利用率有 62%。使用 `perfmon /res` 打开资源监视器，在 CPU 一栏有 "62% CPU 使用率 118% 最大频率"。51% * 118%  = 61%。任务管理器超频下的执行时间也包括在内，而 GetSystemTimes 得到的时间不包括超频下的执行时间。

将 do_busy_things 线程数改为 `std::thread::hardware_concurrency()`，也就是物理线程数。神奇的事情发生了，上面的代码显示 100%，任务管理器显示 100%，资源监视器显示 "116% CPU 使用率 116% 最大频率"。看来任务管理器进行了截断，如果 CPU 总利用率大于 100%，会直接显示100%。而资源监视器会实事求是地显示。

总的来说，使用 GetSystemTimes  计算 CPU 总利用率虽然没有包含超频执行时间，但是不会超过 100%，符合直觉。
