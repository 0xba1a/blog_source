---
title: kexec-tools with the hidden purgatory
date: 2020-01-20 17:35:13
tags:
 - linux
 - ARMv8
 - kernel
---

This is one of the unforgettable experience in my engineering life. Last year when we brought-up a ARM64 based platform, we faced lots of hurdles. But this one was very interesting and had lots of surprises. Though this triaging effort tolled multiple frustrating days, I had very good learning. I had to understand `kexec_load` system call, kernel reboot path and kernel early boot path to root cause the issue. We were using 4.14.19 kernel for the bring-up. But I'll give code samples from 5.4.13 as it is latest while I write this article. There is not much code difference.

The boot flow of our new platform was `uboot --> service Linux --> main Linux`. We had `service Linux` to offload major hardware initialization work from bootloader. Also to keep that code platform agnostic [same `service Linux` ran on x86 platforms too]. To make the jump from `service Linux` to `main Linux` we used `kexec`. The problem here was it took almost 2 minutes for the `main Linux` to start booting. This was a significant difference while comparing with x86 platforms. And this kind of delay would definitely annoy our customers.

After hundreds of rebuilds and thousands of print statements, I found the cause and fixed the issue. Now that 2 minutes delay was reduced to 4 seconds. Let me just tell you the code flow rather dump you with my debugging methods. I kept deferring from writing this article for almost an year. Because I was afraid of the code references and connecting them to show the flow.

# Code reference
| package     | version                                                        |
| ---         | ---                                                            |
| kexec-tools | today's latest master 2c9f26ed20a791a7df0182ba82e93abb52f5a615 |
| linux       | 5.4.13                                                         |

# User story
Login to the `service Linux`. Load `main Linux`'s kernel, initrd. The current `device-tree` will be taken for `main Linux` too. Boot to `main Linux`.

```sh
# kexec -l /main/vmlinuz --initrd=/main/initrd.img
# kexec -e
```

As I mentioned earlier there was 2 mins delay between last `Bye` from `service Linux` and first message from `main Linux`. So I started looking for time consuming operations between those two prints.

# Kernel reboot path
`kexec -e` calls `reboot()` system call with `LINUX_REBOOT_CMD_KEXEC`
``` C
kexec/kexec.c my_exec +900:
---
reboot(LINUX_REBOOT_CMD_KEXEC);
```

Reboot system call in kernel calls `kernel_kexec()` for the argument `LINUX_REBOOT_CMD_KEXEC`. You can refer to my earlier article [Anatomy of Linux system call in ARM64](http://eastrivervillage.com/Anatomy-of-Linux-system-call-in-ARM64/) to understand how arguments are passed from user space to kernel space.
``` C
kernel/reboot.c SYSCALL_DEFINE4(reboot, ...) +380:
---
#ifdef CONFIG_KEXEC_CORE
	case LINUX_REBOOT_CMD_KEXEC:
		ret = kernel_kexec();
		break;
#endif
```

`kernel_kexec()` at `kernel/kexec_core.c +1119` does following things.

| line   | function call                          | action                                                                                                                                                  |
| ------ | -------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1123   | `mutex_trylock(&kexec_mutex)`          | This lock is held to avoid multiple entrance into kexec                                                                                                 |
| 1131   | `if (kexec_image->preserve_context) {` | Something related to keep the device status not disturbed after during kexec. We were not using it. Moving to else part.                                |
| 1165   | `migrate_to_reboot_cpu()`              | Continue execution from logical CPU 0. Only one control path is valid during reboot and startup.                                                        |
| 1175   | `machine_shutdown()`                   | Don't get confused by the name of this function. Its just a wrapper around `disable_nonboot_cpus()` in arm64. It does nothing but disables other CPUs   |
| 1178   | `machine_kexec(kexec_image)`           | Execution continues passing `kexec_image` as argument. This call should not return.                                                                     |

`machine_kexec()` at `arch/arm64/kernel/machine_kexec.c +144`,

| line | function call                                                                             | action                                                                                                                                                                                                                                                                                                                                                              |
| ---  | ---                                                                                       | ---                                                                                                                                                                                                                                                                                                                                                                 |
| 148  | `bool in_kexec_crash = (kimage == kexec_crash_image)`                                     | Not true in our case. `kimage` is `kexec_image`                                                                                                                                                                                                                                                                                                                     |
| 149  | `bool stuck_cpus = cpus_are_stuck_in_kernel();`                                           | Get number of stuck CPUs. Ideally it should be 0                                                                                                                                                                                                                                                                                                                    |
| 154  | <code>BUG_ON(!in_kexec_crash && (stuck_cpus &#124;&#124; (num_online_cpus() > 1)))</code> | In non-crash situations, no CPU should be stuck and no other CPU other than reboot CPU should be online.                                                                                                                                                                                                                                                            |
| 158  | `reboot_code_buffer_phys = page_to_phys(kimage->control_code_page)`                       | Get the physical page address from `kimage`. Jump happens to this address.                                                                                                                                                                                                                                                                                          |
| 159  | `reboot_code_buffer = phys_to_virt(reboot_code_buffer_phys)`                              | Store the virtual address of same to continue working on that area.                                                                                                                                                                                                                                                                                                 |
| 179  | `memcpy(reboot_code_buffer, arm64_relocate_new_kernel, arm64_relocate_new_kernel_size)`   | Copies `arm64_relocate_new_kernel_size` routines address to the jump location. It is implemented in assembly. This is the routine that copies new kernel to its correct place. To make sure the memory where this routine resides doesn't get overwritten during the copy, it is copied inside `kexec_image` and executed from there. Never expected right? me too. |

