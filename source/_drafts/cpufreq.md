# Phytium 处理器 CPU 调频

## D30000M

### UEFI 固件

cpufreq-info 输出

```bash
root@Ubuntu:/sys/devices/system/cpu/cpufreq# cpufreq-info
cpufrequtils 008: cpufreq-info (C) Dominik Brodowski 2004-2009
Report errors and bugs to cpufreq@vger.kernel.org, please.
analyzing CPU 0:
  driver: cppc_cpufreq
  CPUs which run at the same hardware frequency: 0
  CPUs which need to have their frequency coordinated by software: 0
  maximum transition latency: 4294.55 ms.
  hardware limits: 600 MHz - 1.90 GHz
  available cpufreq governors: powersave, conservative, ondemand, userspace, performance, schedutil
  current policy: frequency should be within 600 MHz and 1.90 GHz.
                  The governor "schedutil" may decide which speed to use
                  within this range.
  current CPU frequency is 600 MHz (asserted by call to hardware).
analyzing CPU 1:
  driver: cppc_cpufreq
  CPUs which run at the same hardware frequency: 1
  CPUs which need to have their frequency coordinated by software: 1
  maximum transition latency: 4294.55 ms.
  hardware limits: 600 MHz - 1.90 GHz
  available cpufreq governors: powersave, conservative, ondemand, userspace, performance, schedutil
  current policy: frequency should be within 600 MHz and 1.90 GHz.
                  The governor "schedutil" may decide which speed to use
                  within this range.
  current CPU frequency is 1.80 GHz (asserted by call to hardware).
analyzing CPU 2:
  driver: cppc_cpufreq
  CPUs which run at the same hardware frequency: 2
  CPUs which need to have their frequency coordinated by software: 2
  maximum transition latency: 4294.55 ms.
  hardware limits: 600 MHz - 1.90 GHz
  available cpufreq governors: powersave, conservative, ondemand, userspace, performance, schedutil
  current policy: frequency should be within 600 MHz and 1.90 GHz.
                  The governor "schedutil" may decide which speed to use
                  within this range.
  current CPU frequency is 1.80 GHz (asserted by call to hardware).
analyzing CPU 3:
  driver: cppc_cpufreq
  CPUs which run at the same hardware frequency: 3
  CPUs which need to have their frequency coordinated by software: 3
  maximum transition latency: 4294.55 ms.
  hardware limits: 600 MHz - 2.90 GHz
  available cpufreq governors: powersave, conservative, ondemand, userspace, performance, schedutil
  current policy: frequency should be within 600 MHz and 2.90 GHz.
                  The governor "schedutil" may decide which speed to use
                  within this range.
  current CPU frequency is 1.80 GHz (asserted by call to hardware).
analyzing CPU 4:
  driver: cppc_cpufreq
  CPUs which run at the same hardware frequency: 4
  CPUs which need to have their frequency coordinated by software: 4
  maximum transition latency: 4294.55 ms.
  hardware limits: 600 MHz - 1.90 GHz
  available cpufreq governors: powersave, conservative, ondemand, userspace, performance, schedutil
  current policy: frequency should be within 600 MHz and 1.90 GHz.
                  The governor "schedutil" may decide which speed to use
                  within this range.
  current CPU frequency is 1.20 GHz (asserted by call to hardware).
analyzing CPU 5:
  driver: cppc_cpufreq
  CPUs which run at the same hardware frequency: 5
  CPUs which need to have their frequency coordinated by software: 5
  maximum transition latency: 4294.55 ms.
  hardware limits: 600 MHz - 1.90 GHz
  available cpufreq governors: powersave, conservative, ondemand, userspace, performance, schedutil
  current policy: frequency should be within 600 MHz and 1.90 GHz.
                  The governor "schedutil" may decide which speed to use
                  within this range.
  current CPU frequency is 600 MHz (asserted by call to hardware).
analyzing CPU 6:
  driver: cppc_cpufreq
  CPUs which run at the same hardware frequency: 6
  CPUs which need to have their frequency coordinated by software: 6
  maximum transition latency: 4294.55 ms.
  hardware limits: 600 MHz - 1.90 GHz
  available cpufreq governors: powersave, conservative, ondemand, userspace, performance, schedutil
  current policy: frequency should be within 600 MHz and 1.90 GHz.
                  The governor "schedutil" may decide which speed to use
                  within this range.
  current CPU frequency is 600 MHz (asserted by call to hardware).
analyzing CPU 7:
  driver: cppc_cpufreq
  CPUs which run at the same hardware frequency: 7
  CPUs which need to have their frequency coordinated by software: 7
  maximum transition latency: 4294.55 ms.
  hardware limits: 600 MHz - 2.90 GHz
  available cpufreq governors: powersave, conservative, ondemand, userspace, performance, schedutil
  current policy: frequency should be within 600 MHz and 2.90 GHz.
                  The governor "schedutil" may decide which speed to use
                  within this range.
  current CPU frequency is 600 MHz (asserted by call to hardware).
```

CPU 温度信息如下：

```bash
root@Ubuntu:/sys/devices/system/cpu/cpufreq# sensors
acpitz-acpi-0
Adapter: ACPI interface
temp1:        +31.0 C
temp2:        +30.0 C
```
