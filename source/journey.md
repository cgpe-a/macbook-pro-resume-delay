# MacBook Pro Resume Delay
## Problem
My MacBook Pro does not promptly resume from sleep. Typically it takes about 20s after the lid is opened before the display comes on and the unlock password prompt appears. The problem is *almost* consistent, but sometime goes away for a period. I've not observed a pattern to this. Certainly rebooting does not fix the problem.
## Environment
My machine is...
```
$ sudo dmidecode -s system-product-name
MacBookPro11,4
```
... an Apple mid-2015 MacBook Pro (15" Retina), according to [this model list](https://theapplewiki.com/wiki/List_of_MacBook_Pros) on [the Apple Wiki](https://theapplewiki.com/wiki/Main_Page), with these [tech specs](https://support.apple.com/en-gb/111955) from Apple.

It is over 10 years old, so the original [Mac OS X](https://en.wikipedia.org/wiki/MacOS) operating system is no longer supported by Apple on this machine, and so it has been replaced with...
```
$ grep PRETTY /etc/os-release
PRETTY_NAME="Pop!_OS 24.04 LTS"

$ grep LIKE /etc/os-release
ID_LIKE="ubuntu debian"
```
... [Pop!_OS](https://system76.com/pop) from [system76](https://system76.com/), which is based on [Ubuntu](https://ubuntu.com/), which is based on [Debian](https://www.debian.org/).

The desktop environment is...
```
$ sudo apt info cosmic-session 2>/dev/null | grep -e ^Version -e ^Description
Version: 0.1.0~1757100934~24.04~2441be2
Description: The session for the COSMIC desktop
```
... [COSMIC](https://system76.com/cosmic/), also from system76.

The kernel is...
```
$ uname -r
6.16.3-76061603-generic

$ sudo apt info linux-image-generic 2>/dev/null | grep -e Version -e Maintainer
Version: 6.16.3-76061603.202508231538~1757347095~24.04~5472d08
Maintainer: Ubuntu Kernel Team <kernel-team@lists.ubuntu.com>
```
... the Ubuntu version of [Linux](https://www.kernel.org/) 6.16.3.
## Initial investigation
What is actually happening when I close and open the lid? Let's look at the system logs (the [systemd journal](https://www.freedesktop.org/software/systemd/man/latest/systemd-journald.service.html)) for the current boot, using:
```
$ sudo journalctl -b
```
The first line of the output shows the version of Linux that was started:
```
Sep 11 15:55:45 macbookpro kernel: Linux version 6.16.3-76061603-generic (jenkins@warp.pop-os.org) (x86_64-linux-gnu-gcc-14 (Ubuntu 14.2.0-4ubuntu2~24.04) 14.2.0, GNU ld (GNU Binutils for Ubuntu) 2.42) #202508231538~1757347095~24.04~5472d08 SMP PREEMPT_DYNAMIC Mon S
```
Looking at the time when the laptop lid was closed, you can see the suspend process being initiated:
```
Sep 11 16:02:48 macbookpro systemd-logind[875]: Lid closed.
Sep 11 16:02:48 macbookpro systemd-logind[875]: Suspending...
```
There is then a flurry of activity as services react to the system being suspended, followed by systemd asking the kernel to suspend the system (I've removed some non-systemd/kernel lines for clarity):
```
Sep 11 16:02:49 macbookpro systemd[1]: Reached target sleep.target - Sleep.
Sep 11 16:02:49 macbookpro systemd[1]: Starting systemd-suspend.service - System Suspend...
Sep 11 16:02:49 macbookpro systemd[1]: Starting systemd-rfkill.service - Load/Save RF Kill Switch Status...
Sep 11 16:02:49 macbookpro systemd-sleep[4753]: Performing sleep operation 'suspend'...
Sep 11 16:02:49 macbookpro kernel: PM: suspend entry (deep)
Sep 11 16:02:49 macbookpro systemd[1]: Started systemd-rfkill.service - Load/Save RF Kill Switch Status.
Sep 11 16:02:49 macbookpro kernel: Filesystems sync: 0.023 seconds
Sep 11 16:04:13 macbookpro kernel: Freezing user space processes
Sep 11 16:04:13 macbookpro kernel: Freezing user space processes completed (elapsed 0.001 seconds)
Sep 11 16:04:13 macbookpro kernel: OOM killer disabled.
Sep 11 16:04:13 macbookpro kernel: Freezing remaining freezable tasks
Sep 11 16:04:13 macbookpro kernel: Freezing remaining freezable tasks completed (elapsed 0.001 seconds)
Sep 11 16:04:13 macbookpro kernel: printk: Suspending console(s) (use no_console_suspend to debug)
Sep 11 16:04:13 macbookpro kernel: sd 1:0:0:0: [sda] Synchronizing SCSI cache
Sep 11 16:04:13 macbookpro kernel: ata1.00: Entering standby power mode
Sep 11 16:04:13 macbookpro kernel: ACPI: EC: interrupt blocked
Sep 11 16:04:13 macbookpro kernel: ACPI: PM: Preparing to enter system sleep state S3
Sep 11 16:04:13 macbookpro kernel: ACPI: EC: event blocked
Sep 11 16:04:13 macbookpro kernel: ACPI: EC: EC stopped
Sep 11 16:04:13 macbookpro kernel: ACPI: PM: Saving platform NVS memory
Sep 11 16:04:13 macbookpro kernel: Disabling non-boot CPUs ...
Sep 11 16:04:13 macbookpro kernel: smpboot: CPU 7 is now offline
Sep 11 16:04:13 macbookpro kernel: smpboot: CPU 6 is now offline
Sep 11 16:04:13 macbookpro kernel: smpboot: CPU 5 is now offline
Sep 11 16:04:13 macbookpro kernel: smpboot: CPU 4 is now offline
Sep 11 16:04:13 macbookpro kernel: smpboot: CPU 3 is now offline
Sep 11 16:04:13 macbookpro kernel: smpboot: CPU 2 is now offline
Sep 11 16:04:13 macbookpro kernel: smpboot: CPU 1 is now offline
```
Note that the timestamps above during suspend appear to skip to the time that the system was *resumed*, presumably because they were only retreived and logged then.

The next lines show the beginning of the resume process:
```
Sep 11 16:04:13 macbookpro kernel: ACPI: PM: Low-level resume complete
Sep 11 16:04:13 macbookpro kernel: ACPI: EC: EC started
Sep 11 16:04:13 macbookpro kernel: ACPI: PM: Restoring platform NVS memory
Sep 11 16:04:13 macbookpro kernel: Enabling non-boot CPUs ...
Sep 11 16:04:13 macbookpro kernel: smpboot: Booting Node 0 Processor 1 APIC 0x2
Sep 11 16:04:13 macbookpro kernel: hrtimer: interrupt took 2832505 ns
Sep 11 16:04:13 macbookpro kernel: CPU1 is up
Sep 11 16:04:13 macbookpro kernel: smpboot: Booting Node 0 Processor 2 APIC 0x4
Sep 11 16:04:13 macbookpro kernel: CPU2 is up
Sep 11 16:04:13 macbookpro kernel: smpboot: Booting Node 0 Processor 3 APIC 0x6
Sep 11 16:04:13 macbookpro kernel: CPU3 is up
Sep 11 16:04:13 macbookpro kernel: smpboot: Booting Node 0 Processor 4 APIC 0x1
Sep 11 16:04:13 macbookpro kernel: CPU4 is up
Sep 11 16:04:13 macbookpro kernel: smpboot: Booting Node 0 Processor 5 APIC 0x3
Sep 11 16:04:13 macbookpro kernel: CPU5 is up
Sep 11 16:04:13 macbookpro kernel: smpboot: Booting Node 0 Processor 6 APIC 0x5
Sep 11 16:04:13 macbookpro kernel: CPU6 is up
Sep 11 16:04:13 macbookpro kernel: smpboot: Booting Node 0 Processor 7 APIC 0x7
Sep 11 16:04:13 macbookpro kernel: CPU7 is up
Sep 11 16:04:13 macbookpro kernel: ACPI: PM: Waking up from system sleep state S3
Sep 11 16:04:13 macbookpro kernel: ACPI: EC: interrupt unblocked
Sep 11 16:04:13 macbookpro kernel: pcieport 0000:00:1c.0: Enabling MPC IRBNCE
Sep 11 16:04:13 macbookpro kernel: pcieport 0000:00:1c.0: Intel PCH root port ACS workaround enabled
Sep 11 16:04:13 macbookpro kernel: pcieport 0000:00:1c.3: Enabling MPC IRBNCE
Sep 11 16:04:13 macbookpro kernel: pcieport 0000:00:1c.3: Intel PCH root port ACS workaround enabled
Sep 11 16:04:13 macbookpro kernel: pcieport 0000:00:1c.2: Enabling MPC IRBNCE
Sep 11 16:04:13 macbookpro kernel: pcieport 0000:00:1c.2: Intel PCH root port ACS workaround enabled
Sep 11 16:04:13 macbookpro kernel: ACPI: EC: event unblocked
Sep 11 16:04:13 macbookpro kernel: ata1: SATA link up 6.0 Gbps (SStatus 133 SControl 300)
Sep 11 16:04:13 macbookpro kernel: ata1.00: LPM support broken, forcing max_power
Sep 11 16:04:13 macbookpro kernel: ata1.00: unexpected _GTF length (8)
Sep 11 16:04:13 macbookpro kernel: sd 1:0:0:0: [sda] Starting disk
Sep 11 16:04:13 macbookpro kernel: ata1.00: LPM support broken, forcing max_power
Sep 11 16:04:13 macbookpro kernel: ata1.00: unexpected _GTF length (8)
Sep 11 16:04:13 macbookpro kernel: ata1.00: configured for UDMA/133
Sep 11 16:04:13 macbookpro kernel: usb 1-8: reset full-speed USB device number 2 using xhci_hcd
Sep 11 16:04:13 macbookpro kernel: OOM killer enabled.
Sep 11 16:04:13 macbookpro kernel: Restarting tasks: Starting
Sep 11 16:04:13 macbookpro kernel: usb 2-4: USB disconnect, device number 2
Sep 11 16:04:13 macbookpro kernel: Restarting tasks: Done
Sep 11 16:04:13 macbookpro kernel: random: crng reseeded on system resumption
Sep 11 16:04:13 macbookpro kernel: video LNXVIDEO:00: Restoring backlight state
Sep 11 16:04:13 macbookpro kernel: PM: suspend exit
```
There then follows a flurry of service activity for the resume, including the lid open event matching the suspend-initiating lid close event:
```
Sep 11 16:04:13 macbookpro systemd-logind[875]: Lid opened.
```
Soon systemd regards the suspend/resume as complete:
```
Sep 11 16:04:14 macbookpro systemd[1]: systemd-suspend.service: Deactivated successfully.
Sep 11 16:04:14 macbookpro systemd[1]: Finished systemd-suspend.service - System Suspend.
Sep 11 16:04:14 macbookpro systemd[1]: Stopped target sleep.target - Sleep.
Sep 11 16:04:14 macbookpro systemd[1]: Reached target suspend.target - Suspend.
Sep 11 16:04:14 macbookpro systemd-logind[875]: Operation 'suspend' finished.
Sep 11 16:04:14 macbookpro systemd[1]: Stopped target suspend.target - Suspend.
```
To understand where the time is going on resume, we can look at the kernel ring buffer directly. This shows the value of the kernel monotonic clock for each message. The first kernel message was 1s after the lid was closed in the journal (see above). The kernel part of the suspend appears to take about 6.2s:
```
[  426.169808] PM: suspend entry (deep)
[  426.193051] Filesystems sync: 0.023 seconds
[  426.196977] Freezing user space processes
[  426.198967] Freezing user space processes completed (elapsed 0.001 seconds)
[  426.198973] OOM killer disabled.
[  426.198974] Freezing remaining freezable tasks
[  426.200272] Freezing remaining freezable tasks completed (elapsed 0.001 seconds)
[  426.200325] printk: Suspending console(s) (use no_console_suspend to debug)
[  426.217341] sd 1:0:0:0: [sda] Synchronizing SCSI cache
[  426.238039] ata1.00: Entering standby power mode
[  431.429256] ACPI: EC: interrupt blocked
[  432.363282] ACPI: PM: Preparing to enter system sleep state S3
[  432.364207] ACPI: EC: event blocked
[  432.364209] ACPI: EC: EC stopped
[  432.364210] ACPI: PM: Saving platform NVS memory
[  432.364231] Disabling non-boot CPUs ...
[  432.365864] smpboot: CPU 7 is now offline
[  432.372376] smpboot: CPU 6 is now offline
[  432.378680] smpboot: CPU 5 is now offline
[  432.389927] smpboot: CPU 4 is now offline
[  432.392641] smpboot: CPU 3 is now offline
[  432.395269] smpboot: CPU 2 is now offline
[  432.397906] smpboot: CPU 1 is now offline
```
The kernel part of the resume seems to take 23.3s:
```
[  432.400350] ACPI: PM: Low-level resume complete
[  432.400387] ACPI: EC: EC started
[  432.400389] ACPI: PM: Restoring platform NVS memory
[  432.400838] Enabling non-boot CPUs ...
[  432.400986] smpboot: Booting Node 0 Processor 1 APIC 0x2
[  432.424395] hrtimer: interrupt took 2832505 ns
[  433.004880] CPU1 is up
[  433.009853] smpboot: Booting Node 0 Processor 2 APIC 0x4
[  433.695568] CPU2 is up
[  433.701283] smpboot: Booting Node 0 Processor 3 APIC 0x6
[  434.443495] CPU3 is up
[  434.444636] smpboot: Booting Node 0 Processor 4 APIC 0x1
[  434.464561] CPU4 is up
[  434.464616] smpboot: Booting Node 0 Processor 5 APIC 0x3
[  438.541201] CPU5 is up
[  438.568225] smpboot: Booting Node 0 Processor 6 APIC 0x5
[  443.102704] CPU6 is up
[  443.120631] smpboot: Booting Node 0 Processor 7 APIC 0x7
[  448.968119] CPU7 is up
[  448.979098] ACPI: PM: Waking up from system sleep state S3
[  448.982064] ACPI: EC: interrupt unblocked
[  448.982214] pcieport 0000:00:1c.0: Enabling MPC IRBNCE
[  448.982219] pcieport 0000:00:1c.0: Intel PCH root port ACS workaround enabled
[  448.982338] pcieport 0000:00:1c.3: Enabling MPC IRBNCE
[  448.982342] pcieport 0000:00:1c.3: Intel PCH root port ACS workaround enabled
[  448.983140] pcieport 0000:00:1c.2: Enabling MPC IRBNCE
[  448.983149] pcieport 0000:00:1c.2: Intel PCH root port ACS workaround enabled
[  450.062494] ACPI: EC: event unblocked
[  450.409568] ata1: SATA link up 6.0 Gbps (SStatus 133 SControl 300)
[  450.409781] ata1.00: LPM support broken, forcing max_power
[  450.409995] ata1.00: unexpected _GTF length (8)
[  450.410235] sd 1:0:0:0: [sda] Starting disk
[  450.410494] ata1.00: LPM support broken, forcing max_power
[  450.410661] ata1.00: unexpected _GTF length (8)
[  450.410758] ata1.00: configured for UDMA/133
[  455.571194] usb 1-8: reset full-speed USB device number 2 using xhci_hcd
[  455.701207] OOM killer enabled.
[  455.701211] Restarting tasks: Starting
[  455.701427] usb 2-4: USB disconnect, device number 2
[  455.709553] Restarting tasks: Done
[  455.709575] random: crng reseeded on system resumption
[  455.709608] video LNXVIDEO:00: Restoring backlight state
[  455.709857] PM: suspend exit
```
The last kernel message is 1s before systemd shows the resume as complete in the journal (see above). Interestingly, the total resume time of 24s is similar, but actually slightly longer than my perceived resume time of just under 20s.

These timestamps show the resume time split between 16.5s bringing up the CPUs, and 6.7s doing everything else. Why does it take so long to bring up the CPUs? This is clearly where we should concentrate our efforts.
## Reproduction with mainline kernel
To investigate what's going on, it will be easier to work with a mainline kernel instead of the Ubuntu kernel. The Ubuntu kernel is based on 6.16.3 (see above). Let's use the latest 6.16, which is also the latest stable kernel series on kernel.org. In my clone of the kernel source:
```
$ git checkout stable/linux-6.16.y
...
HEAD is now at 131e2001572b Linux 6.16.7
```
We use the Ubuntu kernel config as the basis for our mainline kernel config, adjusting some config values:
```
$ cp /boot/config-6.16.3-76061603-generic .config
$ make oldconfig
$ make menuconfig
```
The adjusted config values are as follows...

Make sure the kernel version is clearly a custom one with full versioning information:
```
$ grep LOCALVERSION .config
CONFIG_LOCALVERSION="-custom"
CONFIG_LOCALVERSION_AUTO=y
```
Clear the paths to debian/Ubuntu keys that I don't have:
```
$ grep -e SYSTEM_TRUSTED_KEYS -e SYSTEM_REVOCATION_KEYS .config
CONFIG_SYSTEM_TRUSTED_KEYS=""
CONFIG_SYSTEM_REVOCATION_KEYS=""
```
That is, these key paths have to be removed:
```
$ grep -e SYSTEM_TRUSTED_KEYS -e SYSTEM_REVOCATION_KEYS /boot/config-6.16.3-76061603-generic
CONFIG_SYSTEM_TRUSTED_KEYS="debian/canonical-certs.pem"
CONFIG_SYSTEM_REVOCATION_KEYS="debian/canonical-revoked-certs.pem"
```
It's tempting to try and minimise the kernel config to match my specific hardware in order to reduce compilation time, particularly of modules that will not be loaded, but I've not attempted this.

Build the kernel & modules, install the modules, install the kernel (& initramfs), reboot:
```
$ make -j$(nproc)
$ sudo make modules_install
$ sudo make install
$ reboot
...
$ uname -r
6.16.7-custom
```
After suspending and resuming the mainline kernel, dmesg reveals a similar 13.5s delay in bringing up the CPUs:
```
[  102.942035] Enabling non-boot CPUs ...
[  102.942180] smpboot: Booting Node 0 Processor 1 APIC 0x2
[  103.023929] hrtimer: interrupt took 3613289 ns
[  103.596184] CPU1 is up
[  103.601009] smpboot: Booting Node 0 Processor 2 APIC 0x4
[  104.220023] CPU2 is up
[  104.225500] smpboot: Booting Node 0 Processor 3 APIC 0x6
[  105.063083] CPU3 is up
[  105.064190] smpboot: Booting Node 0 Processor 4 APIC 0x1
[  105.074552] CPU4 is up
[  105.074607] smpboot: Booting Node 0 Processor 5 APIC 0x3
[  108.772530] CPU5 is up
[  108.774106] smpboot: Booting Node 0 Processor 6 APIC 0x5
[  112.350794] CPU6 is up
[  112.352357] smpboot: Booting Node 0 Processor 7 APIC 0x7
[  116.447284] CPU7 is up
```
## Finding the problematic code path
Where are we in the kernel code? Somewhere between...
```
$ git grep 'Booting Node %.'
arch/x86/kernel/smpboot.c:              pr_info("Booting Node %d Processor %d APIC 0x%x\n",
```
[`announce_cpu()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/smpboot.c#L781) and...
```
$ git grep 'CPU%. is up'
kernel/cpu.c:                   pr_info("CPU%d is up\n", cpu);
```
[`thaw_secondary_cpus()`](https://elixir.bootlin.com/linux/v6.16.6/source/kernel/cpu.c#L1965).

### _cpu_up()

In `thaw_secondary_cpus()`, there is a call to [`_cpu_up()`](https://elixir.bootlin.com/linux/v6.16.6/source/kernel/cpu.c#L1629). We can measure the time spent in the various functions that this calls using [ftrace](https://www.kernel.org/doc/html/v6.16/trace/ftrace.html), made easier to use with [`trace-cmd`](https://opensource.com/article/21/7/linux-kernel-trace-cmd):
```
$ sudo apt install trace-cmd
$ sudo trace-cmd start -p function_graph -g _cpu_up --max-graph-depth 3
  plugin 'function_graph'
```
... suspend & resume ...
```
$ sudo trace-cmd stop
$ sudo trace-cmd extract
```
The calls to `_cpu_up()` take 4.7s, 3.8s, 3.6s, 0.1s, 3.7s, 3.9s, 4.2s for each of the 7 secondary CPUs being brought up:
```
$ trace-cmd report -O fgraph:tailprint | grep }.*_cpu_up
   systemd-sleep-10168 [000]  7653.527608: funcgraph_exit:       $ 4714843 us |  } /* _cpu_up */
   systemd-sleep-10168 [000]  7657.367648: funcgraph_exit:       $ 3840015 us |  } /* _cpu_up */
   systemd-sleep-10168 [000]  7660.985030: funcgraph_exit:       $ 3617339 us |  } /* _cpu_up */
   systemd-sleep-10168 [000]  7661.114031: funcgraph_exit:       # 128947.782 us |  } /* _cpu_up */
   systemd-sleep-10168 [000]  7664.802806: funcgraph_exit:       $ 3688745 us |  } /* _cpu_up */
   systemd-sleep-10168 [000]  7668.742224: funcgraph_exit:       $ 3939383 us |  } /* _cpu_up */
   systemd-sleep-10168 [000]  7672.892377: funcgraph_exit:       $ 4150112 us |  } /* _cpu_up */
```
The time is dominated by the final (for each CPU) of many calls to [`cpuhp_invoke_callback()`](https://elixir.bootlin.com/linux/v6.16.6/source/kernel/cpu.c#L169). e.g.:
```
$ trace-cmd report -O fgraph:tailprint | grep }.*_cpu_up
   systemd-sleep-10168 [000]  7668.747873: funcgraph_entry:                   |    __cpuhp_invoke_callback_range() {
   systemd-sleep-10168 [000]  7668.747875: funcgraph_entry:        0.691 us   |      cpuhp_next_state();
   systemd-sleep-10168 [000]  7668.747877: funcgraph_entry:        6.613 us   |      cpuhp_invoke_callback();
   systemd-sleep-10168 [000]  7668.747884: funcgraph_entry:        0.300 us   |      cpuhp_next_state();
   systemd-sleep-10168 [000]  7668.747885: funcgraph_entry:        6.153 us   |      cpuhp_invoke_callback();
   systemd-sleep-10168 [000]  7668.747892: funcgraph_entry:        0.621 us   |      cpuhp_next_state();
   systemd-sleep-10168 [000]  7668.747894: funcgraph_entry:      + 11.746 us  |      cpuhp_invoke_callback();
   systemd-sleep-10168 [000]  7668.747906: funcgraph_entry:        0.268 us   |      cpuhp_next_state();
   systemd-sleep-10168 [000]  7668.747907: funcgraph_entry:        0.823 us   |      cpuhp_invoke_callback();
   systemd-sleep-10168 [000]  7668.747909: funcgraph_entry:        0.488 us   |      cpuhp_next_state();
   systemd-sleep-10168 [000]  7668.747910: funcgraph_entry:        1.755 us   |      cpuhp_invoke_callback();
   systemd-sleep-10168 [000]  7668.747912: funcgraph_entry:        0.693 us   |      cpuhp_next_state();
   systemd-sleep-10168 [000]  7668.747913: funcgraph_entry:        1.106 us   |      cpuhp_invoke_callback();
   systemd-sleep-10168 [000]  7668.747915: funcgraph_entry:        0.499 us   |      cpuhp_next_state();
   systemd-sleep-10168 [000]  7668.747916: funcgraph_entry:      + 12.590 us  |      cpuhp_invoke_callback();
   systemd-sleep-10168 [000]  7668.747928: funcgraph_entry:        0.373 us   |      cpuhp_next_state();
   systemd-sleep-10168 [000]  7668.747929: funcgraph_entry:        1.517 us   |      cpuhp_invoke_callback();
   systemd-sleep-10168 [000]  7668.747931: funcgraph_entry:        0.351 us   |      cpuhp_next_state();
   systemd-sleep-10168 [000]  7668.747932: funcgraph_entry:        0.212 us   |      cpuhp_invoke_callback();
   systemd-sleep-10168 [000]  7668.747932: funcgraph_entry:        0.181 us   |      cpuhp_next_state();
   systemd-sleep-10168 [000]  7668.747933: funcgraph_entry:        2.132 us   |      cpuhp_invoke_callback();
   systemd-sleep-10168 [000]  7668.747935: funcgraph_entry:        0.481 us   |      cpuhp_next_state();
   systemd-sleep-10168 [000]  7668.747937: funcgraph_entry:      ! 364.570 us |      cpuhp_invoke_callback();
   systemd-sleep-10168 [000]  7668.748301: funcgraph_entry:        0.271 us   |      cpuhp_next_state();
   systemd-sleep-10168 [000]  7668.748302: funcgraph_entry:        0.641 us   |      cpuhp_invoke_callback();
   systemd-sleep-10168 [000]  7668.748303: funcgraph_entry:        0.371 us   |      cpuhp_next_state();
   systemd-sleep-10168 [000]  7668.748304: funcgraph_entry:      + 90.437 us  |      cpuhp_invoke_callback();
   systemd-sleep-10168 [000]  7668.748394: funcgraph_entry:        0.452 us   |      cpuhp_next_state();
   systemd-sleep-10168 [000]  7668.748396: funcgraph_entry:        0.585 us   |      cpuhp_invoke_callback();
   systemd-sleep-10168 [000]  7668.748397: funcgraph_entry:        0.593 us   |      cpuhp_next_state();
   systemd-sleep-10168 [000]  7668.748398: funcgraph_entry:        0.493 us   |      cpuhp_invoke_callback();
   systemd-sleep-10168 [000]  7668.748399: funcgraph_entry:        0.217 us   |      cpuhp_next_state();
   systemd-sleep-10168 [000]  7668.748399: funcgraph_entry:        2.109 us   |      cpuhp_invoke_callback();
   systemd-sleep-10168 [000]  7668.748401: funcgraph_entry:        0.661 us   |      cpuhp_next_state();
   systemd-sleep-10168 [000]  7668.748403: funcgraph_entry:      + 84.536 us  |      cpuhp_invoke_callback();
   systemd-sleep-10168 [000]  7668.748488: funcgraph_entry:        0.396 us   |      cpuhp_next_state();
   systemd-sleep-10168 [000]  7668.748488: funcgraph_entry:      $ 4143809 us |      cpuhp_invoke_callback();
   systemd-sleep-10168 [000]  7672.892300: funcgraph_entry:        0.328 us   |      cpuhp_next_state();
   systemd-sleep-10168 [000]  7672.892300: funcgraph_exit:       $ 4144427 us |    } /* __cpuhp_invoke_callback_range */
```
### cpuhp_invoke_callback()
Dig deeper:
```
$ mv trace.dat trace-_cpu_up.dat
$ sudo trace-cmd reset
$ sudo trace-cmd start -p function_graph -g cpuhp_invoke_callback --max-graph-depth 2
...
$ sudo trace-cmd stop
$ sudo trace-cmd extract
$ trace-cmd report -O fgraph:tailprint | grep \\$
   systemd-sleep-12109 [000]  9516.141799: funcgraph_exit:       $ 3478008 us |    } /* cpuhp_bringup_ap */
   systemd-sleep-12109 [000]  9516.141801: funcgraph_exit:       $ 3478012 us |  } /* cpuhp_invoke_callback */
   systemd-sleep-12109 [000]  9519.397964: funcgraph_exit:       $ 3245351 us |    } /* cpuhp_bringup_ap */
   systemd-sleep-12109 [000]  9519.397966: funcgraph_exit:       $ 3245354 us |  } /* cpuhp_invoke_callback */
   systemd-sleep-12109 [000]  9522.651937: funcgraph_exit:       $ 3246098 us |    } /* cpuhp_bringup_ap */
   systemd-sleep-12109 [000]  9522.651939: funcgraph_exit:       $ 3246103 us |  } /* cpuhp_invoke_callback */
         cpuhp/5-45    [005]  9526.506150: funcgraph_exit:       $ 1000111 us |  } /* cpuhp_invoke_callback */
   systemd-sleep-12109 [000]  9526.507978: funcgraph_exit:       $ 3579939 us |    } /* cpuhp_bringup_ap */
   systemd-sleep-12109 [000]  9526.507979: funcgraph_exit:       $ 3579941 us |  } /* cpuhp_invoke_callback */
         cpuhp/6-51    [006]  9529.034874: funcgraph_entry:      $ 1042192 us |    sched_cpu_activate();
         cpuhp/6-51    [006]  9530.077322: funcgraph_exit:       $ 1042656 us |  } /* cpuhp_invoke_callback */
   systemd-sleep-12109 [000]  9530.079152: funcgraph_exit:       $ 3564807 us |    } /* cpuhp_bringup_ap */
   systemd-sleep-12109 [000]  9530.079153: funcgraph_exit:       $ 3564811 us |  } /* cpuhp_invoke_callback */
         cpuhp/7-57    [007]  9532.807532: funcgraph_entry:      $ 1145106 us |    sched_cpu_activate();
         cpuhp/7-57    [007]  9533.952886: funcgraph_exit:       $ 1145577 us |  } /* cpuhp_invoke_callback */
   systemd-sleep-12109 [000]  9533.954817: funcgraph_exit:       $ 3841806 us |    } /* cpuhp_bringup_ap */
   systemd-sleep-12109 [000]  9533.954818: funcgraph_exit:       $ 3841809 us |  } /* cpuhp_invoke_callback */
$ trace-cmd report -O fgraph:tailprint | grep }.*cpuhp_bringup_ap
   systemd-sleep-12109 [000]  9516.141799: funcgraph_exit:       $ 3478008 us |    } /* cpuhp_bringup_ap */
   systemd-sleep-12109 [000]  9519.397964: funcgraph_exit:       $ 3245351 us |    } /* cpuhp_bringup_ap */
   systemd-sleep-12109 [000]  9522.651937: funcgraph_exit:       $ 3246098 us |    } /* cpuhp_bringup_ap */
   systemd-sleep-12109 [000]  9522.880168: funcgraph_exit:       # 69128.722 us |    } /* cpuhp_bringup_ap */
   systemd-sleep-12109 [000]  9526.507978: funcgraph_exit:       $ 3579939 us |    } /* cpuhp_bringup_ap */
   systemd-sleep-12109 [000]  9530.079152: funcgraph_exit:       $ 3564807 us |    } /* cpuhp_bringup_ap */
   systemd-sleep-12109 [000]  9533.954817: funcgraph_exit:       $ 3841806 us |    } /* cpuhp_bringup_ap */
```
So, the problematic callback is [`cpuhp_bringup_ap()`](https://elixir.bootlin.com/linux/v6.16.6/source/kernel/cpu.c#L826).
### cpuhp_bringup_ap()
Deeper:
```
$ mv trace.dat trace-cpuhp_invoke_callback.dat
$ sudo trace-cmd reset
$ sudo trace-cmd start -p function_graph -g cpuhp_bringup_ap --max-graph-depth 3
...
$ sudo trace-cmd stop
$ sudo trace-cmd extract
$ trace-cmd report -O fgraph:tailprint | grep \\$
   systemd-sleep-13653 [000] 10960.482144: funcgraph_entry:      $ 3770912 us |      wait_for_completion();
   systemd-sleep-13653 [000] 10964.253057: funcgraph_exit:       $ 3770916 us |    } /* __cpuhp_kick_ap */
   systemd-sleep-13653 [000] 10964.253058: funcgraph_exit:       $ 3858876 us |  } /* cpuhp_bringup_ap */
   systemd-sleep-13653 [000] 10964.384966: funcgraph_entry:      $ 3394687 us |      wait_for_completion();
   systemd-sleep-13653 [000] 10967.779654: funcgraph_exit:       $ 3394692 us |    } /* __cpuhp_kick_ap */
   systemd-sleep-13653 [000] 10967.779655: funcgraph_exit:       $ 3464522 us |  } /* cpuhp_bringup_ap */
   systemd-sleep-13653 [000] 10967.851335: funcgraph_entry:      $ 3317698 us |      wait_for_completion();
   systemd-sleep-13653 [000] 10971.169035: funcgraph_exit:       $ 3317704 us |    } /* __cpuhp_kick_ap */
   systemd-sleep-13653 [000] 10971.169036: funcgraph_exit:       $ 3381857 us |  } /* cpuhp_bringup_ap */
   systemd-sleep-13653 [000] 10971.327013: funcgraph_entry:      $ 3413068 us |      wait_for_completion();
   systemd-sleep-13653 [000] 10974.740082: funcgraph_exit:       $ 3413072 us |    } /* __cpuhp_kick_ap */
   systemd-sleep-13653 [000] 10974.740082: funcgraph_exit:       $ 3469283 us |  } /* cpuhp_bringup_ap */
   systemd-sleep-13653 [000] 10974.804581: funcgraph_entry:      $ 3985241 us |      wait_for_completion();
   systemd-sleep-13653 [000] 10978.789823: funcgraph_exit:       $ 3985245 us |    } /* __cpuhp_kick_ap */
   systemd-sleep-13653 [000] 10978.789823: funcgraph_exit:       $ 4043214 us |  } /* cpuhp_bringup_ap */
   systemd-sleep-13653 [000] 10978.880196: funcgraph_entry:      $ 4184594 us |      wait_for_completion();
   systemd-sleep-13653 [000] 10983.064791: funcgraph_exit:       $ 4184601 us |    } /* __cpuhp_kick_ap */
   systemd-sleep-13653 [000] 10983.064791: funcgraph_exit:       $ 4240767 us |  } /* cpuhp_bringup_ap */
```
### complete_ap_thread()
So, it looks like [`__cpuh_kick_ap()`](https://elixir.bootlin.com/linux/v6.16.6/source/kernel/cpu.c#L761) is waiting for a thread. Specifically, [`wait_for_ap_thread()`](https://elixir.bootlin.com/linux/v6.16.6/source/kernel/cpu.c#L261) is waiting for a call to [`complete_ap_thread()`](https://elixir.bootlin.com/linux/v6.16.6/source/kernel/cpu.c#L267). There are 2 calls made where the `bringup` parameter can be set to `true`: [`cpuhp_thread_fun()`](https://elixir.bootlin.com/linux/v6.16.6/source/kernel/cpu.c#L1060), and [`cpuhp_online_idle()`](https://elixir.bootlin.com/linux/v6.16.6/source/kernel/cpu.c#L1608). Let's work out which it is:
```
$ mv trace.dat trace-cpuhp_bringup_ap.dat
$ sudo trace-cmd reset
$ sudo trace-cmd start -p function_graph -g cpuhp_thread_fun -g cpuhp_online_idle --max-graph-depth 3
...
$ sudo trace-cmd stop
$ sudo trace-cmd extract
$ trace-cmd report -O fgraph:tailprint | grep -B 2 complete\(
...
          <idle>-0     [006] 11673.780809: funcgraph_entry:      # 2302.229 us |      kthread_unpark();
          <idle>-0     [006] 11673.783355: funcgraph_exit:       # 2749.440 us |    } /* stop_machine_unpark */
          <idle>-0     [006] 11673.783585: funcgraph_entry:                   |    complete() {
--
         cpuhp/6-51    [006] 11677.274644: funcgraph_entry:                   |  cpuhp_thread_fun() {
         cpuhp/6-51    [006] 11677.274880: funcgraph_entry:      ! 197.166 us |    cpuhp_next_state();
         cpuhp/6-51    [006] 11677.275330: funcgraph_entry:                   |    complete() {
--
          <idle>-0     [007] 11677.421006: funcgraph_entry:      # 2294.661 us |      kthread_unpark();
          <idle>-0     [007] 11677.423543: funcgraph_exit:       # 21848.303 us |    } /* stop_machine_unpark */
          <idle>-0     [007] 11677.423768: funcgraph_entry:                   |    complete() {
--
         cpuhp/7-57    [007] 11681.316060: funcgraph_entry:                   |  cpuhp_thread_fun() {
         cpuhp/7-57    [007] 11681.316299: funcgraph_entry:      ! 191.564 us |    cpuhp_next_state();
         cpuhp/7-57    [007] 11681.316757: funcgraph_entry:                   |    complete() {
```
### smpboot_thread_fn()
It looks like for each CPU there is a completion from `cpuhp_online_idle()` by the `<idle>-0` thread, followed by a completion from `cpuhp_thread_fun()` by the CPU specific `cpuhp/N-t` thread. The idle thread completion seems to happen immediately after the CPU thread completion, which in turn seems to be delayed. This is consistent with it being the *final* per-CPU call to `cpuhp_bringup_ap() that is delayed waiting for this completion.

```
$ mv trace.dat trace-cpuhp_completion.dat
$ sudo trace-cmd reset
$ sudo trace-cmd start -p function_graph -g smpboot_thread_fn --max-graph-depth 3
...
$ sudo trace-cmd stop
$ sudo trace-cmd extract
```
This doesn't generate any results, presumably because `smpboot_thread_fn()` is already running, and the thread is "parked" & "unparked" when the CPU is brought offline & online.

Instead, trace the (indirectly) called functions.
```
$ sudo trace-cmd reset
$ sudo trace-cmd start -p function_graph -g cpuhp_should_run -g cpuhp_thread_fun --max-graph-depth 3
...
$ sudo trace-cmd stop
$ sudo trace-cmd extract
$ trace-cmd report -O fgraph:tailprint | grep -E -e '[6789][0-9]{5}\.' -e '[0-9]{7} us'
         cpuhp/1-21    [001]  4214.600812: funcgraph_entry:      # 789991.380 us |      workqueue_online_cpu();
         cpuhp/1-21    [001]  4215.391164: funcgraph_exit:       # 790664.444 us |    } /* cpuhp_invoke_callback */
         cpuhp/1-21    [001]  4215.391486: funcgraph_exit:       # 791979.853 us |  } /* cpuhp_thread_fun */
         cpuhp/1-21    [001]  4215.452977: funcgraph_entry:      # 863548.866 us |      cacheinfo_cpu_online();
         cpuhp/1-21    [001]  4216.316892: funcgraph_exit:       # 864229.706 us |    } /* cpuhp_invoke_callback */
         cpuhp/1-21    [001]  4216.317204: funcgraph_exit:       # 865542.710 us |  } /* cpuhp_thread_fun */
         cpuhp/2-27    [002]  4218.604395: funcgraph_entry:      # 620308.818 us |      workqueue_online_cpu();
         cpuhp/2-27    [002]  4219.225032: funcgraph_exit:       # 620911.301 us |    } /* cpuhp_invoke_callback */
         cpuhp/2-27    [002]  4219.225314: funcgraph_exit:       # 622089.121 us |  } /* cpuhp_thread_fun */
         cpuhp/2-27    [002]  4219.273510: funcgraph_entry:      # 750180.962 us |      cacheinfo_cpu_online();
         cpuhp/2-27    [002]  4220.024024: funcgraph_exit:       # 750788.950 us |    } /* cpuhp_invoke_callback */
         cpuhp/2-27    [002]  4220.024310: funcgraph_exit:       # 751968.013 us |  } /* cpuhp_thread_fun */
         cpuhp/3-33    [003]  4222.591940: funcgraph_entry:      # 692818.677 us |      cacheinfo_cpu_online();
         cpuhp/3-33    [003]  4223.285062: funcgraph_exit:       # 693378.148 us |    } /* cpuhp_invoke_callback */
         cpuhp/3-33    [003]  4223.285322: funcgraph_exit:       # 694474.756 us |  } /* cpuhp_thread_fun */
         cpuhp/3-33    [003]  4224.250925: funcgraph_entry:      # 658708.087 us |      sched_cpu_activate();
         cpuhp/3-33    [003]  4224.909939: funcgraph_exit:       # 659264.176 us |    } /* cpuhp_invoke_callback */
         cpuhp/3-33    [003]  4224.910201: funcgraph_exit:       # 660403.625 us |  } /* cpuhp_thread_fun */
         cpuhp/5-45    [005]  4225.778046: funcgraph_entry:      # 754764.841 us |      cacheinfo_cpu_online();
         cpuhp/5-45    [005]  4226.533065: funcgraph_exit:       # 755234.842 us |    } /* cpuhp_invoke_callback */
         cpuhp/5-45    [005]  4226.533297: funcgraph_exit:       # 756166.964 us |  } /* cpuhp_thread_fun */
         cpuhp/5-45    [005]  4227.443489: funcgraph_entry:      # 889761.786 us |      sched_cpu_activate();
         cpuhp/5-45    [005]  4228.333507: funcgraph_exit:       # 890233.119 us |    } /* cpuhp_invoke_callback */
         cpuhp/5-45    [005]  4228.333728: funcgraph_exit:       # 891203.825 us |  } /* cpuhp_thread_fun */
         cpuhp/6-51    [006]  4229.290745: funcgraph_entry:      # 749370.083 us |      cacheinfo_cpu_online();
         cpuhp/6-51    [006]  4230.040363: funcgraph_exit:       # 749836.462 us |    } /* cpuhp_invoke_callback */
         cpuhp/6-51    [006]  4230.040588: funcgraph_exit:       # 750749.368 us |  } /* cpuhp_thread_fun */
         cpuhp/6-51    [006]  4231.043442: funcgraph_entry:      # 983237.552 us |      sched_cpu_activate();
         cpuhp/6-51    [006]  4232.027001: funcgraph_exit:       # 983709.559 us |    } /* cpuhp_invoke_callback */
         cpuhp/6-51    [006]  4232.027226: funcgraph_exit:       # 984733.781 us |  } /* cpuhp_thread_fun */
         cpuhp/7-57    [007]  4232.925185: funcgraph_entry:      # 826889.882 us |      cacheinfo_cpu_online();
         cpuhp/7-57    [007]  4233.752331: funcgraph_exit:       # 827355.675 us |    } /* cpuhp_invoke_callback */
         cpuhp/7-57    [007]  4233.752554: funcgraph_exit:       # 828274.929 us |  } /* cpuhp_thread_fun */
         cpuhp/7-57    [007]  4234.828510: funcgraph_entry:      $ 1106878 us |      sched_cpu_activate();
         cpuhp/7-57    [007]  4235.935645: funcgraph_exit:       $ 1107346 us |    } /* cpuhp_invoke_callback */
         cpuhp/7-57    [007]  4235.935862: funcgraph_exit:       $ 1108295 us |  } /* cpuhp_thread_fun */
$ trace-cmd report -O fgraph:tailprint | grep -e workqueue_online_cpu -e cacheinfo_cpu_online -e sched_cpu_activate
         cpuhp/1-21    [001]  4214.600812: funcgraph_entry:      # 789991.380 us |      workqueue_online_cpu();
         cpuhp/1-21    [001]  4215.452977: funcgraph_entry:      # 863548.866 us |      cacheinfo_cpu_online();
         cpuhp/1-21    [001]  4217.548639: funcgraph_entry:      # 565371.841 us |      sched_cpu_activate();
         cpuhp/2-27    [002]  4218.604395: funcgraph_entry:      # 620308.818 us |      workqueue_online_cpu();
         cpuhp/2-27    [002]  4219.273510: funcgraph_entry:      # 750180.962 us |      cacheinfo_cpu_online();
         cpuhp/2-27    [002]  4221.069401: funcgraph_entry:      # 590256.793 us |      sched_cpu_activate();
         cpuhp/3-33    [003]  4222.018862: funcgraph_entry:      # 531270.020 us |      workqueue_online_cpu();
         cpuhp/3-33    [003]  4222.591940: funcgraph_entry:      # 692818.677 us |      cacheinfo_cpu_online();
         cpuhp/3-33    [003]  4224.250925: funcgraph_entry:      # 658708.087 us |      sched_cpu_activate();
         cpuhp/4-39    [004]  4224.929821: funcgraph_entry:      ! 699.761 us |      workqueue_online_cpu();
         cpuhp/4-39    [004]  4224.930551: funcgraph_entry:      ! 830.613 us |      cacheinfo_cpu_online();
         cpuhp/4-39    [004]  4224.932919: funcgraph_entry:      # 52594.281 us |      sched_cpu_activate();
         cpuhp/5-45    [005]  4225.309246: funcgraph_entry:      # 434291.854 us |      workqueue_online_cpu();
         cpuhp/5-45    [005]  4225.778046: funcgraph_entry:      # 754764.841 us |      cacheinfo_cpu_online();
         cpuhp/5-45    [005]  4227.443489: funcgraph_entry:      # 889761.786 us |      sched_cpu_activate();
         cpuhp/6-51    [006]  4228.735596: funcgraph_entry:      # 433868.245 us |      workqueue_online_cpu();
         cpuhp/6-51    [006]  4229.290745: funcgraph_entry:      # 749370.083 us |      cacheinfo_cpu_online();
         cpuhp/6-51    [006]  4231.043442: funcgraph_entry:      # 983237.552 us |      sched_cpu_activate();
         cpuhp/7-57    [007]  4232.361762: funcgraph_entry:      # 425174.823 us |      workqueue_online_cpu();
         cpuhp/7-57    [007]  4232.925185: funcgraph_entry:      # 826889.882 us |      cacheinfo_cpu_online();
         cpuhp/7-57    [007]  4234.828510: funcgraph_entry:      $ 1106878 us |      sched_cpu_activate();
$ echo $(( 789991 + 863548 + 565371 + 620308 + 750180 + 590256 + 531270 + 692818 + 658708 + 699 + 830 + 52594 + 434291 + 754764 + 889761 + 433868 + 749370 + 983237 + 425174 + 826889 + 1106878 ))
12720805
```
So, 12.7s is being taken by calls to  [`workqueue_online_cpu()`](https://elixir.bootlin.com/linux/v6.16.6/source/kernel/workqueue.c#L6656), [`cacheinfo_cpu_online()`](https://elixir.bootlin.com/linux/v6.16.6/source/drivers/base/cacheinfo.c#L958) & [`sched_cpu_activate()`](https://elixir.bootlin.com/linux/v6.16.6/source/kernel/sched/core.c#L8258) from `cpuhp_thread_fun()` for each CPU.
### workqueue_online_cpu(), cacheinfo_cpu_online() & sched_cpu_activate()
```
$ mv trace.dat trace-smpboot_thread_fn.dat
$ sudo trace-cmd reset
$ sudo trace-cmd start -p function_graph -g workqueue_online_cpu -g cacheinfo_cpu_online -g sched_cpu_activate --max-graph-depth 3
...
$ sudo trace-cmd stop
$ sudo trace-cmd extract
```

```
$ trace-cmd report -O fgraph:tailprint | grep -E -e '[56789][0-9]{5}\.' -e '[0-9]{7} us'
         cpuhp/1-21    [001] 31242.991863: funcgraph_exit:       $ 2742818 us |  } /* workqueue_online_cpu */
         cpuhp/1-21    [001] 31243.931214: funcgraph_exit:       # 895031.207 us |  } /* cacheinfo_cpu_online */
         cpuhp/1-21    [001] 31245.157457: funcgraph_entry:      # 510620.201 us |      partition_sched_domains();
         cpuhp/1-21    [001] 31245.669056: funcgraph_exit:       # 512611.868 us |    } /* cpuset_reset_sched_domains */
         cpuhp/1-21    [001] 31245.671272: funcgraph_exit:       # 519126.015 us |  } /* sched_cpu_activate */
         cpuhp/2-27    [002] 31248.489314: funcgraph_exit:       $ 2415162 us |  } /* workqueue_online_cpu */
         cpuhp/2-27    [002] 31249.275661: funcgraph_exit:       # 774330.519 us |  } /* cacheinfo_cpu_online */
         cpuhp/2-27    [002] 31250.287436: funcgraph_entry:      # 535856.918 us |      partition_sched_domains();
         cpuhp/2-27    [002] 31250.824192: funcgraph_exit:       # 537662.170 us |    } /* cpuset_reset_sched_domains */
         cpuhp/2-27    [002] 31250.826187: funcgraph_exit:       # 543506.688 us |  } /* sched_cpu_activate */
         cpuhp/3-33    [003] 31253.486444: funcgraph_exit:       $ 2320508 us |  } /* workqueue_online_cpu */
         cpuhp/3-33    [003] 31254.321677: funcgraph_exit:       # 823940.354 us |  } /* cacheinfo_cpu_online */
         cpuhp/3-33    [003] 31255.290712: funcgraph_entry:      # 742075.264 us |      partition_sched_domains();
         cpuhp/3-33    [003] 31256.033609: funcgraph_exit:       # 743748.147 us |    } /* cpuset_reset_sched_domains */
         cpuhp/3-33    [003] 31256.035553: funcgraph_exit:       # 749261.048 us |  } /* sched_cpu_activate */
         cpuhp/5-45    [005] 31258.361027: funcgraph_exit:       $ 1875589 us |  } /* workqueue_online_cpu */
         cpuhp/5-45    [005] 31259.180129: funcgraph_exit:       # 687764.134 us |  } /* cacheinfo_cpu_online */
         cpuhp/5-45    [005] 31260.377259: funcgraph_entry:      # 919813.056 us |      partition_sched_domains();
         cpuhp/5-45    [005] 31261.297705: funcgraph_exit:       # 921230.185 us |    } /* cpuset_reset_sched_domains */
         cpuhp/5-45    [005] 31261.299285: funcgraph_exit:       # 927204.284 us |  } /* sched_cpu_activate */
         cpuhp/6-51    [006] 31263.456680: funcgraph_exit:       $ 1842073 us |  } /* workqueue_online_cpu */
         cpuhp/6-51    [006] 31264.296160: funcgraph_exit:       # 686521.974 us |  } /* cacheinfo_cpu_online */
         cpuhp/6-51    [006] 31265.553453: funcgraph_entry:      # 939118.567 us |      partition_sched_domains();
         cpuhp/6-51    [006] 31266.493267: funcgraph_exit:       # 940540.035 us |    } /* cpuset_reset_sched_domains */
         cpuhp/6-51    [006] 31266.494838: funcgraph_exit:       # 964119.974 us |  } /* sched_cpu_activate */
         cpuhp/7-57    [007] 31268.737850: funcgraph_exit:       $ 1913270 us |  } /* workqueue_online_cpu */
         cpuhp/7-57    [007] 31269.828078: funcgraph_exit:       # 936085.800 us |  } /* cacheinfo_cpu_online */
         cpuhp/7-57    [007] 31271.034541: funcgraph_entry:      $ 1100707 us |      partition_sched_domains();
         cpuhp/7-57    [007] 31272.135946: funcgraph_exit:       $ 1102135 us |    } /* cpuset_reset_sched_domains */
         cpuhp/7-57    [007] 31272.169117: funcgraph_exit:       $ 1139818 us |  } /* sched_cpu_activate */
```
For each CPU, workqueue_online_cpu() takes a lot of time - up to 2.7s, but with no dominating called function. Similarly, cacheinfo_cpu_online() takes up to 0.9s. Calls to sched_cpu_activate() take up to 1.1s, and are dominated by a call to partition_sched_domains().

## Execution speed
Looking in more detail doesn't really shed any light on where the time is going. It seems to be spread out. However, in this run, CPU 4 seems to be much faster than the others:
```
$ trace-cmd report -O fgraph:tailprint | grep }.*workqueue_online_cpu
         cpuhp/1-21    [001] 31242.991863: funcgraph_exit:       $ 2742818 us |  } /* workqueue_online_cpu */
         cpuhp/2-27    [002] 31248.489314: funcgraph_exit:       $ 2415162 us |  } /* workqueue_online_cpu */
         cpuhp/3-33    [003] 31253.486444: funcgraph_exit:       $ 2320508 us |  } /* workqueue_online_cpu */
         cpuhp/4-39    [004] 31256.083504: funcgraph_exit:       # 1594.638 us |  } /* workqueue_online_cpu */
         cpuhp/5-45    [005] 31258.361027: funcgraph_exit:       $ 1875589 us |  } /* workqueue_online_cpu */
         cpuhp/6-51    [006] 31263.456680: funcgraph_exit:       $ 1842073 us |  } /* workqueue_online_cpu */
         cpuhp/7-57    [007] 31268.737850: funcgraph_exit:       $ 1913270 us |  } /* workqueue_online_cpu */
```
It's faster by a factor of over 1000, compared with the next fastest - CPU 6:
```
$ echo $(( 1842073 / 1596 ))
1154
```
Are the other CPUs performing more work, or doing the same work much slower? It the slowness spread out over all calls, or concentrated in something like waiting for locks?

### Call counts
We can construct - for each CPU - a count of function calls in the trace (not all function calls, as we only traced to depth 3, starting at the 3 current functions of interest):
```
$ for i in $(seq 1 7); do trace-cmd report -O fgraph:tailprint | grep cpuhp/$i | sed -n -r -e 's/^.*funcgraph_entry.*\| *(.*)\(\).*$/\1/p' | sort | uniq -c | sort -n > calls-$i; done
```
It looks like this (for CPU 6):
```
$ cat calls-6
      1 balance_push_set
      1 cacheinfo_cpu_online
      1 cpu_map_shared_cache
      1 cpuset_reset_sched_domains
      1 detect_cache_attributes
      1 get_cpu_device
      1 hrtimer_interrupt
      1 __kmalloc_noprof
      1 kstrdup_const
      1 lockdep_assert_cpus_held
      1 __node_distance
      1 partition_sched_domains
      1 _raw_spin_lock
      1 sched_core_idle_cpu
      1 sched_cpu_activate
      1 sched_domains_numa_masks_set
      1 sched_update_numa
      1 ___slab_alloc
      1 static_key_fast_inc_not_disabled
      1 static_key_slow_inc_cpuslocked
      1 update_per_cpu_data_slice_size
      1 workqueue_online_cpu
      2 _raw_spin_lock_irq
      2 raw_spin_rq_lock_nested
      2 _raw_spin_unlock
      2 _raw_spin_unlock_irq
      4 kthread_set_per_cpu
      5 cache_get_priv_group
      5 __kmalloc_node_track_caller_noprof
      6 cpu_device_create
      6 device_add
      6 device_initialize
      6 kfree_const
      6 __kmalloc_cache_noprof
      7 setup_pcp_cacheinfo
     10 last_level_cache_is_valid
     21 irq_enter_rcu
     21 irq_exit_rcu
     21 __sysvec_apic_timer_interrupt
     41 set_cpus_allowed_ptr
     41 __set_cpus_allowed_ptr
     56 wq_update_node_max_active
     76 __cond_resched
     76 mutex_lock
     76 mutex_unlock
    240 copy_workqueue_attrs
    240 wqattrs_actualize_cpumask
    240 wqattrs_equal
    240 wq_calc_pod_cpumask
    448 unbound_wq_update_pwq
```
Comparing CPUs 5 & 6, which take similar times, shows only minor differences in the call counts:
```
$ diff -y --suppress-common-lines calls-[56]
							      >	      4 kthread_set_per_cpu
      5 kthread_set_per_cpu				      <
      6 setup_pcp_cacheinfo				      |	      7 setup_pcp_cacheinfo
      9 last_level_cache_is_valid			      |	     10 last_level_cache_is_valid
     25 irq_enter_rcu					      |	     21 irq_enter_rcu
     25 irq_exit_rcu					      |	     21 irq_exit_rcu
     25 __sysvec_apic_timer_interrupt			      |	     21 __sysvec_apic_timer_interrupt
     42 set_cpus_allowed_ptr				      |	     41 set_cpus_allowed_ptr
     42 __set_cpus_allowed_ptr				      |	     41 __set_cpus_allowed_ptr
```
Comparing CPUs 4 & 5 mostly shows many more timer IRQs for slower CPU 5, which is to be expected due to the increase in elapsed time:
```
$ diff -y --suppress-common-lines calls-[45]
      1 irq_enter_rcu					      <
      1 irq_exit_rcu					      <
      1 jump_label_update				      <
      1 __sysvec_apic_timer_interrupt			      <
      4 kthread_set_per_cpu				      <
      5 setup_pcp_cacheinfo				      |	      5 kthread_set_per_cpu
      8 last_level_cache_is_valid			      |	      6 setup_pcp_cacheinfo
     41 set_cpus_allowed_ptr				      |	      9 last_level_cache_is_valid
     41 __set_cpus_allowed_ptr				      |	     25 irq_enter_rcu
							      >	     25 irq_exit_rcu
							      >	     25 __sysvec_apic_timer_interrupt
							      >	     42 set_cpus_allowed_ptr
							      >	     42 __set_cpus_allowed_ptr
     77 mutex_lock					      |	     76 mutex_lock
     77 mutex_unlock					      |	     76 mutex_unlock
```
So, the CPUs each seem to be calling the same functions, roughly the same number of times.

### Call times
Comparing histograms of CPUs 4 & 5 for the function with the most calls in our trace - unbound_wq_update_pwq(), using different bucket sizes due to the different execution times, confirms that the execution time difference is not concentrated in particular calls:
```
$ hline() { if [ $count -gt 0 ]; then printf '< %-5d : %d\n' $max $count; total=$(( $total + $count )); fi; }; hist() { step=$1; max=0; count=0; total=0; while read nf; do n=${nf%.*}; if [ $n -lt $max ]; then count=$(( $count + 1 )); else hline; max=$(( ($n + $step) / $step * $step )); count=1; fi; done; hline; echo "  total : $total"; }

$ trace-cmd report -O fgraph:tailprint | sed -n -r -e 's/^.*cpuhp\/4-.*:.*:[^0-9]*([0-9.]*) us .*unbound_wq_update_pwq.*$/\1/p' | sort -n | hist 1
< 1     : 207
< 2     : 11
< 3     : 105
< 4     : 92
< 5     : 14
< 6     : 11
< 7     : 7
< 51    : 1
  total : 448

$ trace-cmd report -O fgraph:tailprint | sed -n -r -e 's/^.*cpuhp\/5-.*:.*:[^0-9]*([0-9.]*) us .*unbound_wq_update_pwq.*$/\1/p' | sort -n | hist 100
< 200   : 160
< 300   : 44
< 2100  : 194
< 2200  : 15
< 19400 : 4
< 20000 : 1
< 20100 : 10
< 20200 : 1
< 20400 : 1
< 21200 : 7
< 21300 : 8
< 22300 : 1
< 22400 : 2
  total : 448
```
### CPU frequency
Is there a problem with the frequency that the CPUs are running at?

I doubt that it's a problem with cpufreq, because the min/max frequencies are not different enough:
```
$ grep . /sys/devices/system/cpu/cpu0/cpufreq/*
/sys/devices/system/cpu/cpu0/cpufreq/affected_cpus:0
/sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_avg_freq:798197
/sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_max_freq:3400000
/sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_min_freq:800000
/sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_transition_latency:20000
/sys/devices/system/cpu/cpu0/cpufreq/related_cpus:0
/sys/devices/system/cpu/cpu0/cpufreq/scaling_available_governors:conservative ondemand userspace powersave performance schedutil
/sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq:798197
/sys/devices/system/cpu/cpu0/cpufreq/scaling_driver:intel_cpufreq
/sys/devices/system/cpu/cpu0/cpufreq/scaling_governor:schedutil
/sys/devices/system/cpu/cpu0/cpufreq/scaling_max_freq:3400000
/sys/devices/system/cpu/cpu0/cpufreq/scaling_min_freq:800000
/sys/devices/system/cpu/cpu0/cpufreq/scaling_setspeed:<unsupported>
```
### Performance governor
We can check by always using the max frequency by changing the governor from schedutil to performance:
```
$ for n in $(seq 0 7); do echo performance | sudo tee /sys/devices/system/cpu/cpu$n/cpufreq/scaling_governor; done
```
It's still slow to resume, with all CPUs continuing to use the performance governor after resume and having a cpuinfo_avg_freq close to the max:
```
$ grep . /sys/devices/system/cpu/cpu*/cpufreq/cpuinfo_avg_freq
/sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_avg_freq:2704070
/sys/devices/system/cpu/cpu1/cpufreq/cpuinfo_avg_freq:2704074
/sys/devices/system/cpu/cpu2/cpufreq/cpuinfo_avg_freq:2253660
/sys/devices/system/cpu/cpu3/cpufreq/cpuinfo_avg_freq:2698960
/sys/devices/system/cpu/cpu4/cpufreq/cpuinfo_avg_freq:2392161
/sys/devices/system/cpu/cpu5/cpufreq/cpuinfo_avg_freq:2698163
/sys/devices/system/cpu/cpu6/cpufreq/cpuinfo_avg_freq:2365496
/sys/devices/system/cpu/cpu7/cpufreq/cpuinfo_avg_freq:2379451

$ grep . /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
/sys/devices/system/cpu/cpu0/cpufreq/scaling_governor:performance
/sys/devices/system/cpu/cpu1/cpufreq/scaling_governor:performance
/sys/devices/system/cpu/cpu2/cpufreq/scaling_governor:performance
/sys/devices/system/cpu/cpu3/cpufreq/scaling_governor:performance
/sys/devices/system/cpu/cpu4/cpufreq/scaling_governor:performance
/sys/devices/system/cpu/cpu5/cpufreq/scaling_governor:performance
/sys/devices/system/cpu/cpu6/cpufreq/scaling_governor:performance
/sys/devices/system/cpu/cpu7/cpufreq/scaling_governor:performance
```

### intel_pstate
The intel_pstate cpufreq driver is running in passive mode:
```
$ grep . /sys/devices/system/cpu/intel_pstate/*
/sys/devices/system/cpu/intel_pstate/max_perf_pct:100
/sys/devices/system/cpu/intel_pstate/min_perf_pct:23
/sys/devices/system/cpu/intel_pstate/no_turbo:0
/sys/devices/system/cpu/intel_pstate/num_pstates:27
/sys/devices/system/cpu/intel_pstate/status:passive
/sys/devices/system/cpu/intel_pstate/turbo_pct:45
``` 
This is why the scaling_driver shown above is intel_cpufreq, not intel_pstate. It means that the P-state is selected by the cpufreq governor, not by the hardware itself, as described [here](https://www.kernel.org/doc/html/latest/admin-guide/pm/intel_pstate.html).

Furthermore, looking at [`intel_cpufreq_suspend()`](https://elixir.bootlin.com/linux/v6.16.6/source/drivers/cpufreq/intel_pstate.c#L3436), which calls [`intel_pstate_suspend()`](https://elixir.bootlin.com/linux/v6.16.6/source/drivers/cpufreq/intel_pstate.c#L1441), and [`intel_pstate_resume()`](https://elixir.bootlin.com/linux/v6.16.6/source/drivers/cpufreq/intel_pstate.c#L1455), which is called directly for the [`intel_cpufreq`](https://elixir.bootlin.com/linux/v6.16.6/source/drivers/cpufreq/intel_pstate.c#L3457) scaling driver, they don't seem to actually do anything in passive mode.

CPU hotplug is also used on suspend/resume. The calls to [`intel_cpufreq_cpu_offline()`](https://elixir.bootlin.com/linux/v6.16.6/source/drivers/cpufreq/intel_pstate.c#L3034) and [`intel_pstate_cpu_online()`](https://elixir.bootlin.com/linux/v6.16.6/source/drivers/cpufreq/intel_pstate.c#L3059) also don't seem to do much in passive mode.

### cpufreq debug
We can turn on the debug printks in intel_pstate to see whether the speed of execution changes when it runs.
```
$ echo 8 | sudo tee /proc/sys/kernel/printk
$ echo 'module intel_pstate +pflmt' | sudo tee /sys/kernel/debug/dynamic_debug/control
```
The dmesg output shows suspend/offline/online/resume:
```
[76087.625631] [92333] intel_pstate:intel_pstate_suspend:1445: intel_pstate: CPU 7 suspending
[76087.625635] [92333] intel_pstate:intel_pstate_suspend:1445: intel_pstate: CPU 6 suspending
[76087.625637] [92333] intel_pstate:intel_pstate_suspend:1445: intel_pstate: CPU 5 suspending
[76087.625639] [92333] intel_pstate:intel_pstate_suspend:1445: intel_pstate: CPU 4 suspending
[76087.625640] [92333] intel_pstate:intel_pstate_suspend:1445: intel_pstate: CPU 3 suspending
[76087.625642] [92333] intel_pstate:intel_pstate_suspend:1445: intel_pstate: CPU 2 suspending
[76087.625644] [92333] intel_pstate:intel_pstate_suspend:1445: intel_pstate: CPU 1 suspending
[76087.625646] [92333] intel_pstate:intel_pstate_suspend:1445: intel_pstate: CPU 0 suspending
...
[76089.073109] Disabling non-boot CPUs ...
[76089.073474] [57] intel_pstate:intel_cpufreq_cpu_offline:3038: intel_pstate: CPU 7 going offline
[76089.074728] smpboot: CPU 7 is now offline
[76089.080115] [51] intel_pstate:intel_cpufreq_cpu_offline:3038: intel_pstate: CPU 6 going offline
[76089.081321] smpboot: CPU 6 is now offline
[76089.086378] [45] intel_pstate:intel_cpufreq_cpu_offline:3038: intel_pstate: CPU 5 going offline
[76089.087590] smpboot: CPU 5 is now offline
[76089.092691] [39] intel_pstate:intel_cpufreq_cpu_offline:3038: intel_pstate: CPU 4 going offline
[76089.093900] smpboot: CPU 4 is now offline
[76089.098964] [33] intel_pstate:intel_cpufreq_cpu_offline:3038: intel_pstate: CPU 3 going offline
[76089.100157] smpboot: CPU 3 is now offline
[76089.102385] [27] intel_pstate:intel_cpufreq_cpu_offline:3038: intel_pstate: CPU 2 going offline
[76089.103614] smpboot: CPU 2 is now offline
[76089.104833] [21] intel_pstate:intel_cpufreq_cpu_offline:3038: intel_pstate: CPU 1 going offline
[76089.106029] smpboot: CPU 1 is now offline
...
[76089.108966] Enabling non-boot CPUs ...
[76089.109209] smpboot: Booting Node 0 Processor 1 APIC 0x2
[76091.240680] [21] intel_pstate:intel_pstate_cpu_online:3063: intel_pstate: CPU 1 going online
[76091.247403] [21] intel_pstate:intel_pstate_update_perf_limits:2913: intel_pstate: cpu:1 min_policy_perf:8 max_policy_perf:34
[76091.254655] [21] intel_pstate:intel_pstate_update_perf_limits:2929: intel_pstate: cpu:1 global_min:8 global_max:34
[76091.261888] [21] intel_pstate:intel_pstate_update_perf_limits:2942: intel_pstate: cpu:1 max_perf_ratio:34 min_perf_ratio:8
[76091.508096] CPU1 is up
[76091.513183] smpboot: Booting Node 0 Processor 2 APIC 0x4
[76091.847972] [27] intel_pstate:intel_pstate_cpu_online:3063: intel_pstate: CPU 2 going online
[76091.856541] [27] intel_pstate:intel_pstate_update_perf_limits:2913: intel_pstate: cpu:2 min_policy_perf:8 max_policy_perf:34
[76091.863072] [27] intel_pstate:intel_pstate_update_perf_limits:2929: intel_pstate: cpu:2 global_min:8 global_max:34
[76091.871712] [27] intel_pstate:intel_pstate_update_perf_limits:2942: intel_pstate: cpu:2 max_perf_ratio:34 min_perf_ratio:8
[76092.123477] CPU2 is up
[76092.128861] smpboot: Booting Node 0 Processor 3 APIC 0x6
[76093.413603] [33] intel_pstate:intel_pstate_cpu_online:3063: intel_pstate: CPU 3 going online
[76093.419339] [33] intel_pstate:intel_pstate_update_perf_limits:2913: intel_pstate: cpu:3 min_policy_perf:8 max_policy_perf:34
[76093.425340] [33] intel_pstate:intel_pstate_update_perf_limits:2929: intel_pstate: cpu:3 global_min:8 global_max:34
[76093.463597] [33] intel_pstate:intel_pstate_update_perf_limits:2942: intel_pstate: cpu:3 max_perf_ratio:34 min_perf_ratio:8
[76094.518737] CPU3 is up
[76094.526278] smpboot: Booting Node 0 Processor 4 APIC 0x1
[76094.532367] [39] intel_pstate:intel_pstate_cpu_online:3063: intel_pstate: CPU 4 going online
[76094.532374] [39] intel_pstate:intel_pstate_update_perf_limits:2913: intel_pstate: cpu:4 min_policy_perf:8 max_policy_perf:34
[76094.532376] [39] intel_pstate:intel_pstate_update_perf_limits:2929: intel_pstate: cpu:4 global_min:8 global_max:34
[76094.532378] [39] intel_pstate:intel_pstate_update_perf_limits:2942: intel_pstate: cpu:4 max_perf_ratio:34 min_perf_ratio:8
[76094.539072] CPU4 is up
[76094.539119] smpboot: Booting Node 0 Processor 5 APIC 0x3
[76095.914313] [45] intel_pstate:intel_pstate_cpu_online:3063: intel_pstate: CPU 5 going online
[76095.929354] [45] intel_pstate:intel_pstate_update_perf_limits:2913: intel_pstate: cpu:5 min_policy_perf:8 max_policy_perf:34
[76095.943343] [45] intel_pstate:intel_pstate_update_perf_limits:2929: intel_pstate: cpu:5 global_min:8 global_max:34
[76095.957368] [45] intel_pstate:intel_pstate_update_perf_limits:2942: intel_pstate: cpu:5 max_perf_ratio:34 min_perf_ratio:8
[76097.779174] CPU5 is up
[76097.780689] smpboot: Booting Node 0 Processor 6 APIC 0x5
[76099.241466] [51] intel_pstate:intel_pstate_cpu_online:3063: intel_pstate: CPU 6 going online
[76099.257377] [51] intel_pstate:intel_pstate_update_perf_limits:2913: intel_pstate: cpu:6 min_policy_perf:8 max_policy_perf:34
[76099.262719] [51] intel_pstate:intel_pstate_update_perf_limits:2929: intel_pstate: cpu:6 global_min:8 global_max:34
[76099.274449] [51] intel_pstate:intel_pstate_update_perf_limits:2942: intel_pstate: cpu:6 max_perf_ratio:34 min_perf_ratio:8
[76101.087682] CPU6 is up
[76101.098948] smpboot: Booting Node 0 Processor 7 APIC 0x7
[76103.057621] [57] intel_pstate:intel_pstate_cpu_online:3063: intel_pstate: CPU 7 going online
[76103.073609] [57] intel_pstate:intel_pstate_update_perf_limits:2913: intel_pstate: cpu:7 min_policy_perf:8 max_policy_perf:34
[76103.078836] [57] intel_pstate:intel_pstate_update_perf_limits:2929: intel_pstate: cpu:7 global_min:8 global_max:34
[76103.084128] [57] intel_pstate:intel_pstate_update_perf_limits:2942: intel_pstate: cpu:7 max_perf_ratio:34 min_perf_ratio:8
[76105.556226] CPU7 is up
[76105.567104] ACPI: PM: Waking up from system sleep state S3
[76105.568980] ACPI: EC: interrupt unblocked
...
[76107.093851] [92333] intel_pstate:intel_pstate_resume:1459: intel_pstate: CPU 7 resuming
[76107.093857] [92333] intel_pstate:intel_pstate_resume:1459: intel_pstate: CPU 6 resuming
[76107.093859] [92333] intel_pstate:intel_pstate_resume:1459: intel_pstate: CPU 5 resuming
[76107.093861] [92333] intel_pstate:intel_pstate_resume:1459: intel_pstate: CPU 4 resuming
[76107.093863] [92333] intel_pstate:intel_pstate_resume:1459: intel_pstate: CPU 3 resuming
[76107.093865] [92333] intel_pstate:intel_pstate_resume:1459: intel_pstate: CPU 2 resuming
[76107.093866] [92333] intel_pstate:intel_pstate_resume:1459: intel_pstate: CPU 1 resuming
[76107.093868] [92333] intel_pstate:intel_pstate_resume:1459: intel_pstate: CPU 0 resuming
```
Notably the timestamps while coming online show the CPUs to be slow both before and after the debug lines from intel_pstate. They also show that once again CPU4 appears to be running much faster.

The output from [`intel_pstate_update_perf_limits()`](https://elixir.bootlin.com/linux/v6.16.6/source/drivers/cpufreq/intel_pstate.c#L2884), presumably called by [`intel_cpufreq_verify_policy()`](https://elixir.bootlin.com/linux/v6.16.6/source/drivers/cpufreq/intel_pstate.c#L3160), is informative, because the code between the 3 debug lines is just calculations, with no I/O or locking, etc. The differences between CPU4 and the rest is therefore a simple reflection of different CPU execution rates.

The difference between the 1st & 3rd debug line on CPU4 is 4uS. On CPU5 it's 28mS.

## Speed recovery
When does the system start running at full speed?
### User space test
We can see that after fully resuming, all CPUs are running at approximately the same speed. This is tested using a user space benchmark written in pure bash, and using taskset to run on each CPU:
```
$ for i in $(seq 0 7); do time taskset -c $i bash -c 'fib() { if [ $1 -eq 0 ]; then echo 0; else echo $(( $1 + $(fib $(( $1 - 1 ))) )); fi; }; fib 400'; done
80200

real	0m2.190s
user	0m0.204s
sys	0m1.980s
80200

real	0m1.962s
user	0m0.176s
sys	0m1.781s
80200

real	0m1.937s
user	0m0.194s
sys	0m1.738s
80200

real	0m1.937s
user	0m0.173s
sys	0m1.757s
80200

real	0m2.222s
user	0m0.233s
sys	0m1.983s
80200

real	0m1.918s
user	0m0.196s
sys	0m1.718s
80200

real	0m1.925s
user	0m0.183s
sys	0m1.737s
80200

real	0m1.907s
user	0m0.192s
sys	0m1.710s
```
### Kernel speed test
We can modify the kernel to test how fast the CPUs are running during resume by re-using the delay loop routine taken from the loops_per_jiffy speed callibration routine. Initially we add a test just after the CPUs are brought up, and when the resume completes:
```
$ git diff
diff --git a/kernel/cpu.c b/kernel/cpu.c
index a59e009e0be4..d214afe00be9 100644
--- a/kernel/cpu.c
+++ b/kernel/cpu.c
@@ -1962,6 +1962,45 @@ void __weak arch_thaw_secondary_cpus_end(void)
 {
 }

+/* simple loop based delay - copied from arch/x86/lib/delay.c: */
+static void delay_loop(u64 __loops)
+{
+       unsigned long loops = (unsigned long)__loops;
+
+       asm volatile(
+               "       test %0,%0      \n"
+               "       jz 3f           \n"
+               "       jmp 1f          \n"
+
+               ".align 16              \n"
+               "1:     jmp 2f          \n"
+
+               ".align 16              \n"
+               "2:     dec %0          \n"
+               "       jnz 2b          \n"
+               "3:     dec %0          \n"
+
+               : "+a" (loops)
+               :
+       );
+}
+
+static void observe_cpu_speed(void *info)
+{
+       const char *when = info;
+       int cpu = smp_processor_id();
+
+       pr_info("CPU %d when %s, looping...\n", cpu, when);
+       delay_loop(loops_per_jiffy * 2);
+       pr_info("CPU %d ...done\n", cpu);
+}
+
+extern void observe_cpu_speeds(const char *when);
+void observe_cpu_speeds(const char *when)
+{
+       on_each_cpu(observe_cpu_speed, (void *)when, 1);
+}
+
 void thaw_secondary_cpus(void)
 {
        int cpu, error;
@@ -1986,6 +2025,7 @@ void thaw_secondary_cpus(void)
                }
                pr_warn("Error taking CPU%d up: %d\n", cpu, error);
        }
+       observe_cpu_speeds("thaw_secondary_cpus: all up");

        arch_thaw_secondary_cpus_end();

diff --git a/kernel/power/suspend.c b/kernel/power/suspend.c
index b4ca17c2fecf..86885338750a 100644
--- a/kernel/power/suspend.c
+++ b/kernel/power/suspend.c
@@ -616,6 +616,8 @@ static int enter_state(suspend_state_t state)
        return error;
 }

+extern void observe_cpu_speeds(const char *when);
+
 /**
  * pm_suspend - Externally visible function for suspending the system.
  * @state: System sleep state to enter.
@@ -634,6 +636,7 @@ int pm_suspend(suspend_state_t state)
        error = enter_state(state);
        dpm_save_errno(error);
        pr_info("suspend exit\n");
+       observe_cpu_speeds("pm_suspend: done");
        return error;
 }
 EXPORT_SYMBOL(pm_suspend);
```
The CPUs are slow just after the CPUs are brought up by [`thaw_secondary_cpus()`](https://elixir.bootlin.com/linux/v6.16.6/source/kernel/cpu.c#L1965):
```
[195274.866077] CPU 0 when thaw_secondary_cpus: all up, looping...
[195274.866079] CPU 4 when thaw_secondary_cpus: all up, looping...
[195274.866241] CPU 3 when thaw_secondary_cpus: all up, looping...
[195274.866242] CPU 6 when thaw_secondary_cpus: all up, looping...
[195274.866386] CPU 5 when thaw_secondary_cpus: all up, looping...
[195274.866386] CPU 1 when thaw_secondary_cpus: all up, looping...
[195274.866435] CPU 2 when thaw_secondary_cpus: all up, looping...
[195274.866863] CPU 7 when thaw_secondary_cpus: all up, looping...
[195274.925253] CPU 0 ...done
[195274.930399] CPU 4 ...done
[195277.662283] CPU 2 ...done
[195277.662477] CPU 6 ...done
[195277.692165] CPU 5 ...done
[195277.692182] CPU 1 ...done
[195277.720104] CPU 3 ...done
[195277.720635] CPU 7 ...done
```
Although it's notable that in this case CPUs 0 & 4 are much faster than the others.

By the end of the resume in [`pm_suspend()`](https://elixir.bootlin.com/linux/v6.16.6/source/kernel/power/suspend.c#L626), all CPUs are equally fast, and all faster than they were earlier:
```
[195281.073798] CPU 4 when pm_suspend: done, looping...
[195281.073799] CPU 0 when pm_suspend: done, looping...
[195281.073799] CPU 3 when pm_suspend: done, looping...
[195281.073799] CPU 2 when pm_suspend: done, looping...
[195281.073799] CPU 7 when pm_suspend: done, looping...
[195281.073799] CPU 6 when pm_suspend: done, looping...
[195281.073799] CPU 1 when pm_suspend: done, looping...
[195281.073799] CPU 5 when pm_suspend: done, looping...
[195281.076551] CPU 0 ...done
[195281.076551] CPU 4 ...done
[195281.076553] CPU 7 ...done
[195281.076554] CPU 1 ...done
[195281.076554] CPU 5 ...done
[195281.076554] CPU 3 ...done
[195281.076554] CPU 6 ...done
[195281.076554] CPU 2 ...done
```
### thaw_processes()
Looking at the dmesg output between these, it appears that lines are going past faster by this point:
```
[195281.052269] Restarting tasks: Starting
[195281.052386] usb 1-5: USB disconnect, device number 10
[195281.052394] usb 1-5.1: USB disconnect, device number 11
[195281.052399] usb 1-5.1.1: USB disconnect, device number 13
[195281.052402] usb 1-5.1.1.2: USB disconnect, device number 14
[195281.052408] usb 2-2: USB disconnect, device number 6
[195281.052414] usb 2-2.1: USB disconnect, device number 7
[195281.052417] usb 2-2.1.1: USB disconnect, device number 8
[195281.072525] Restarting tasks: Done
```
This could be misleading though, as we don't know how much code is running that is not printing anything. Let's observe the CPU speed around this block, which is in [`thaw_processes()`](https://elixir.bootlin.com/linux/v6.16.6/source/kernel/power/process.c#L179):
```
$ git diff kernel/power/process.c
diff --git a/kernel/power/process.c b/kernel/power/process.c
index dc0dfc349f22..edc44cc85d81 100644
--- a/kernel/power/process.c
+++ b/kernel/power/process.c
@@ -176,11 +176,14 @@ int freeze_kernel_threads(void)
        return error;
 }

+extern void observe_cpu_speeds(const char *when);
+
 void thaw_processes(void)
 {
        struct task_struct *g, *p;
        struct task_struct *curr = current;

+       observe_cpu_speeds("thaw_processes: entry");
        trace_suspend_resume(TPS("thaw_processes"), 0, true);
        if (pm_freezing)
                static_branch_dec(&freezer_active);
@@ -189,6 +192,7 @@ void thaw_processes(void)

        oom_killer_enable();

+       observe_cpu_speeds("thaw_processes: starting");
        pr_info("Restarting tasks: Starting\n");

        __usermodehelper_set_disable_depth(UMH_FREEZING);
@@ -210,6 +214,7 @@ void thaw_processes(void)
        schedule();
        pr_info("Restarting tasks: Done\n");
        trace_suspend_resume(TPS("thaw_processes"), 0, false);
+       observe_cpu_speeds("thaw_processes: exit");
 }

 void thaw_kernel_threads(void)
```
It is indeed fast at all 3 of these new observation points:
```
[   92.721765] CPU 7 when thaw_processes: entry, looping...
[   92.721764] CPU 3 when thaw_processes: entry, looping...
[   92.721778] CPU 2 when thaw_processes: entry, looping...
[   92.721778] CPU 6 when thaw_processes: entry, looping...
[   92.721783] CPU 0 when thaw_processes: entry, looping...
[   92.721783] CPU 4 when thaw_processes: entry, looping...
[   92.721783] CPU 1 when thaw_processes: entry, looping...
[   92.721783] CPU 5 when thaw_processes: entry, looping...
[   92.724811] CPU 7 ...done
[   92.724812] CPU 3 ...done
[   92.724819] CPU 6 ...done
[   92.724821] CPU 2 ...done
[   92.724823] CPU 0 ...done
[   92.724824] CPU 5 ...done
[   92.724824] CPU 1 ...done
[   92.724824] CPU 4 ...done
```
### acpi_pm_finish()
The first thing that happens in dmesg after the CPUs are brought up - we've observed they are still slow in thaw_secondary_cpus() - is the ACPI sleep code exiting the S3 sleep state:
```
[   88.961157] ACPI: PM: Waking up from system sleep state S3
```
Let's observe this, which is in [`acpi_pm_finish()`](https://elixir.bootlin.com/linux/v6.16.6/source/drivers/acpi/sleep.c#L477):
```
$ git diff drivers/acpi/sleep.c
diff --git a/drivers/acpi/sleep.c b/drivers/acpi/sleep.c
index c8ee8e42b0f6..1e13cd6ca480 100644
--- a/drivers/acpi/sleep.c
+++ b/drivers/acpi/sleep.c
@@ -468,6 +468,8 @@ static int acpi_pm_prepare(void)
        return error;
 }

+extern void observe_cpu_speeds(const char *when);
+
 /**
  *     acpi_pm_finish - Instruct the platform to leave a sleep state.
  *
@@ -479,6 +481,7 @@ static void acpi_pm_finish(void)
        struct acpi_device *pwr_btn_adev;
        u32 acpi_state = acpi_target_sleep_state;

+       observe_cpu_speeds("acpi_pm_finish: entry");
        acpi_ec_unblock_transactions();
        suspend_nvs_free();

@@ -488,6 +491,7 @@ static void acpi_pm_finish(void)
        pr_info("Waking up from system sleep state S%d\n", acpi_state);
        acpi_disable_wakeup_devices(acpi_state);
        acpi_leave_sleep_state(acpi_state);
+       observe_cpu_speeds("acpi_pm_finish: left sleep state");

        /* reset firmware waking vector */
        acpi_set_waking_vector(0);
@@ -512,6 +516,7 @@ static void acpi_pm_finish(void)
                pm_wakeup_event(&pwr_btn_adev->dev, 0);
                acpi_dev_put(pwr_btn_adev);
        }
+       observe_cpu_speeds("acpi_pm_finish: exit");
 }

 /**
```
It's fast on entry:
```
[   97.438039] CPU 0 when acpi_pm_finish: entry, looping...
[   97.438039] CPU 3 when acpi_pm_finish: entry, looping...
[   97.438040] CPU 5 when acpi_pm_finish: entry, looping...
[   97.438039] CPU 4 when acpi_pm_finish: entry, looping...
[   97.438042] CPU 7 when acpi_pm_finish: entry, looping...
[   97.438043] CPU 1 when acpi_pm_finish: entry, looping...
[   97.438045] CPU 6 when acpi_pm_finish: entry, looping...
[   97.438048] CPU 2 when acpi_pm_finish: entry, looping...
[   97.440793] CPU 5 ...done
[   97.440794] CPU 0 ...done
[   97.440794] CPU 4 ...done
[   97.440794] CPU 7 ...done
[   97.440795] CPU 3 ...done
[   97.440795] CPU 1 ...done
[   97.440798] CPU 6 ...done
[   97.440800] CPU 2 ...done
```
### pm_sleep_enable_secondary_cpus()
The call to [`acpi_pm_finish()`](https://elixir.bootlin.com/linux/v6.16.6/source/drivers/acpi/sleep.c#L477), which we now know to be fast on entry is invoked as the `wake` suspend op from [`platform_resume_noirq()`](https://elixir.bootlin.com/linux/v6.16.6/source/kernel/power/suspend.c#L285), which is called by [`suspend_enter()`](https://elixir.bootlin.com/linux/v6.16.6/source/kernel/power/suspend.c#L413), just after it called [`pm_sleep_enable_secondary_cpus()`](https://elixir.bootlin.com/linux/v6.16.6/source/kernel/power/power.h#L347)

Let's observe that:
```
$ git diff kernel/power/power.h
diff --git a/kernel/power/power.h b/kernel/power/power.h
index 7ccd709af93f..483d0a982720 100644
--- a/kernel/power/power.h
+++ b/kernel/power/power.h
@@ -344,10 +344,15 @@ static inline int pm_sleep_disable_secondary_cpus(void)
        return suspend_disable_secondary_cpus();
 }

+extern void observe_cpu_speeds(const char *when);
+
 static inline void pm_sleep_enable_secondary_cpus(void)
 {
+       observe_cpu_speeds("pm_sleep_enable_secondary_cpus: entry");
        suspend_enable_secondary_cpus();
+       observe_cpu_speeds("pm_sleep_enable_secondary_cpus: CPUs enabled, resuming cpuidle");
        cpuidle_resume();
+       observe_cpu_speeds("pm_sleep_enable_secondary_cpus: exit");
 }

 void dpm_save_errno(int err);
```
On entry there is only CPU0, which doesn't look slow:
```
[   60.574303] CPU 0 when pm_sleep_enable_secondary_cpus: entry, looping...
[   60.577831] CPU 0 ...done
```
Then [`suspend_enable_secondary_cpus()`](https://elixir.bootlin.com/linux/v6.16.6/source/include/linux/cpu.h#L157) is called, which only calls [`thaw_secondary_cpus()`](https://elixir.bootlin.com/linux/v6.16.6/source/kernel/cpu.c#L1965), which we've already seen as slow after all CPUs are up:
```
[   75.737254] CPU 0 when thaw_secondary_cpus: all up, looping...
[   75.737255] CPU 4 when thaw_secondary_cpus: all up, looping...
[   75.737385] CPU 7 when thaw_secondary_cpus: all up, looping...
[   75.737421] CPU 2 when thaw_secondary_cpus: all up, looping...
[   75.737429] CPU 5 when thaw_secondary_cpus: all up, looping...
[   75.737429] CPU 1 when thaw_secondary_cpus: all up, looping...
[   75.737523] CPU 3 when thaw_secondary_cpus: all up, looping...
[   75.737575] CPU 6 when thaw_secondary_cpus: all up, looping...
[   75.796443] CPU 0 ...done
[   75.801575] CPU 4 ...done
[   78.545083] CPU 2 ...done
[   78.545160] CPU 6 ...done
[   78.566431] CPU 1 ...done
[   78.566445] CPU 5 ...done
[   78.586205] CPU 7 ...done
[   78.586601] CPU 3 ...done
```
But by the time [`suspend_enable_secondary_cpus()`](https://elixir.bootlin.com/linux/v6.16.6/source/include/linux/cpu.h#L157) has returned, the CPUs are already fast:
```
[   78.645081] CPU 2 when pm_sleep_enable_secondary_cpus: CPUs enabled, resuming cpuidle, looping...
[   78.645080] CPU 5 when pm_sleep_enable_secondary_cpus: CPUs enabled, resuming cpuidle, looping...
[   78.645081] CPU 7 when pm_sleep_enable_secondary_cpus: CPUs enabled, resuming cpuidle, looping...
[   78.645081] CPU 4 when pm_sleep_enable_secondary_cpus: CPUs enabled, resuming cpuidle, looping...
[   78.645081] CPU 0 when pm_sleep_enable_secondary_cpus: CPUs enabled, resuming cpuidle, looping...
[   78.645082] CPU 3 when pm_sleep_enable_secondary_cpus: CPUs enabled, resuming cpuidle, looping...
[   78.645081] CPU 1 when pm_sleep_enable_secondary_cpus: CPUs enabled, resuming cpuidle, looping...
[   78.645081] CPU 6 when pm_sleep_enable_secondary_cpus: CPUs enabled, resuming cpuidle, looping...
[   78.647844] CPU 3 ...done
[   78.647844] CPU 2 ...done
[   78.647844] CPU 4 ...done
[   78.647845] CPU 0 ...done
[   78.647845] CPU 1 ...done
[   78.647845] CPU 6 ...done
[   78.647846] CPU 7 ...done
[   78.647847] CPU 5 ...done
```
### arch_thaw_secondary_cpus_end()
Let's add a new observation to [`thaw_secondary_cpus()`](https://elixir.bootlin.com/linux/v6.16.6/source/kernel/cpu.c#L1965), after the call to [`arch_thaw_secondary_cpus_end()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/smpboot.c#L1089):
```
diff --git a/kernel/cpu.c b/kernel/cpu.c
index a59e009e0be4..9c9035da4494 100644
--- a/kernel/cpu.c
+++ b/kernel/cpu.c
...
@@ -1986,9 +2025,12 @@ void thaw_secondary_cpus(void)
                }
                pr_warn("Error taking CPU%d up: %d\n", cpu, error);
        }
+       observe_cpu_speeds("thaw_secondary_cpus: all up");

        arch_thaw_secondary_cpus_end();

+       observe_cpu_speeds("thaw_secondary_cpus: arch-end done");
+
        cpumask_clear(frozen_cpus);
 out:
        cpu_maps_update_done();
```
On the first & second suspend/resume after reboot, the CPU bringup delay has all but gone:
```
[  105.552803] Enabling non-boot CPUs ...
[  105.552962] smpboot: Booting Node 0 Processor 1 APIC 0x2
[  105.703796] CPU1 is up
[  105.704011] smpboot: Booting Node 0 Processor 2 APIC 0x4
[  105.841891] CPU2 is up
[  105.842623] smpboot: Booting Node 0 Processor 3 APIC 0x6
[  106.043853] CPU3 is up
[  106.044578] smpboot: Booting Node 0 Processor 4 APIC 0x1
[  106.053261] CPU4 is up
[  106.053317] smpboot: Booting Node 0 Processor 5 APIC 0x3
[  106.128786] CPU5 is up
[  106.128963] smpboot: Booting Node 0 Processor 6 APIC 0x5
[  106.224905] CPU6 is up
[  106.225211] smpboot: Booting Node 0 Processor 7 APIC 0x7
[  106.349227] CPU7 is up
[  106.349234] CPU 0 when thaw_secondary_cpus: all up, looping...
[  106.349237] CPU 4 when thaw_secondary_cpus: all up, looping...
[  106.349264] CPU 2 when thaw_secondary_cpus: all up, looping...
[  106.349278] CPU 1 when thaw_secondary_cpus: all up, looping...
[  106.349279] CPU 3 when thaw_secondary_cpus: all up, looping...
[  106.349293] CPU 6 when thaw_secondary_cpus: all up, looping...
[  106.349293] CPU 5 when thaw_secondary_cpus: all up, looping...
[  106.349329] CPU 7 when thaw_secondary_cpus: all up, looping...
[  106.352071] CPU 0 ...done
[  106.352074] CPU 4 ...done
[  106.352129] CPU 1 ...done
[  106.352141] CPU 2 ...done
[  106.352175] CPU 6 ...done
[  106.352175] CPU 5 ...done
[  106.352183] CPU 3 ...done
[  106.352209] CPU 7 ...done
```
The whole resume is taking about 5s, which is where we're trying to get to. All resumes in this boot are working.

Will rebooting make it fail again?... I think not. The machine wasn't used for a while and powered off. It was resuming okay after powering up again.

I used the machine in this working state for quite a while, with many suspend/resume cycles. However, after a system update replaced the custom kernel with a new Pop!OS kernel, the problem returned. The new kernel is the Ubuntu version of 6.17.9:
```
$ sudo kernelstub -p
kernelstub.Config    : INFO     Looking for configuration...
kernelstub           : INFO     System information:

    OS:..................Pop!_OS 24.04
    Root partition:....../dev/sda5
    Root FS UUID:........02acbbe0-e8ad-4851-80d5-937164789c41
    ESP Path:............/boot/efi
    ESP Partition:......./dev/sda3
    ESP Partition #:.....3
    NVRAM entry #:.......-1
    Boot Variable #:.....0000
    Kernel Boot Options:.quiet loglevel=0 systemd.show_status=false splash  psi=0
    Kernel Image Path:.../boot/vmlinuz-6.17.9-76061709-generic
    Initrd Image Path:.../boot/initrd.img-6.17.9-76061709-generic
    Force-overwrite:.....False

kernelstub           : INFO     Configuration details:

   ESP Location:................../boot/efi
   Management Mode:...............True
   Install Loader configuration:..True
   Configuration version:.........3
```

Reverting to our own kernel again:
```
$ sudo bash -c 'rm /boot/efi/EFI/Pop_OS-*/*-previous*'
$ sudo kernelstub -k /boot/vmlinuz-6.16.7-custom-dirty -i /boot/initrd.img-6.16.7-custom-dirty
$ reboot
```

The first resume with this kernel is slow again.
```
[   32.774359] ACPI: PM: Low-level resume complete
[   32.774398] ACPI: EC: +++++ Starting EC +++++
[   32.774403] ACPI: EC: +++++ EC started +++++
[   32.774405] ACPI: PM: Restoring platform NVS memory
[   32.774885] CPU 0 when pm_sleep_enable_secondary_cpus: entry, looping...
[   32.778394] CPU 0 ...done
[   32.778546] Enabling non-boot CPUs ...
[   32.778677] smpboot: Booting Node 0 Processor 1 APIC 0x2
[   32.934307] hrtimer: interrupt took 3279326 ns
[   33.489012] CPU1 is up
[   33.505749] smpboot: Booting Node 0 Processor 2 APIC 0x4
[   34.123526] CPU2 is up
[   34.125498] smpboot: Booting Node 0 Processor 3 APIC 0x6
[   36.068022] CPU3 is up
[   36.074821] smpboot: Booting Node 0 Processor 4 APIC 0x1
[   36.089264] CPU4 is up
[   36.089324] smpboot: Booting Node 0 Processor 5 APIC 0x3
[   39.492444] CPU5 is up
[   39.512590] smpboot: Booting Node 0 Processor 6 APIC 0x5
[   43.465506] CPU6 is up
[   43.471819] smpboot: Booting Node 0 Processor 7 APIC 0x7
[   48.081264] CPU7 is up
[   48.081270] CPU 0 when thaw_secondary_cpus: all up, looping...
[   48.081272] CPU 4 when thaw_secondary_cpus: all up, looping...
[   48.081401] CPU 7 when thaw_secondary_cpus: all up, looping...
[   48.081467] CPU 6 when thaw_secondary_cpus: all up, looping...
[   48.081483] CPU 5 when thaw_secondary_cpus: all up, looping...
[   48.081438] CPU 1 when thaw_secondary_cpus: all up, looping...
[   48.081510] CPU 2 when thaw_secondary_cpus: all up, looping...
[   48.081624] CPU 3 when thaw_secondary_cpus: all up, looping...
[   48.141705] CPU 0 ...done
[   48.141762] CPU 4 ...done
[   50.918308] CPU 6 ...done
[   50.918445] CPU 2 ...done
[   50.946208] CPU 7 ...done
[   50.946603] CPU 3 ...done
[   51.054860] CPU 1 ...done
[   51.055143] CPU 5 ...done
[   51.069060] CPU 7 when thaw_secondary_cpus: arch-end done, looping...
[   51.069062] CPU 1 when thaw_secondary_cpus: arch-end done, looping...
[   51.069060] CPU 0 when thaw_secondary_cpus: arch-end done, looping...
[   51.069062] CPU 5 when thaw_secondary_cpus: arch-end done, looping...
[   51.069060] CPU 4 when thaw_secondary_cpus: arch-end done, looping...
[   51.069064] CPU 2 when thaw_secondary_cpus: arch-end done, looping...
[   51.069064] CPU 6 when thaw_secondary_cpus: arch-end done, looping...
[   51.069066] CPU 3 when thaw_secondary_cpus: arch-end done, looping...
[   51.071814] CPU 7 ...done
[   51.071815] CPU 1 ...done
[   51.071815] CPU 5 ...done
[   51.071816] CPU 4 ...done
[   51.071816] CPU 0 ...done
[   51.071817] CPU 3 ...done
[   51.071816] CPU 6 ...done
[   51.071817] CPU 2 ...done
```
Furthermore, it's slow in the same place as before, with the speed up after the call to [`arch_thaw_secondary_cpus_end()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/smpboot.c#L1089).
### cache_aps_init() / cache_cpu_init()
The only thing that [`arch_thaw_secondary_cpus_end()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/smpboot.c#L1089) does is call [`cache_aps_init()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/cacheinfo.c#L807). What it does depends on the value of [`memory_caching_control`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/cacheinfo.c#L35), which is initialised by [`mtrr_bp_init()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/mtrr/mtrr.c#L553) and [`pat_bt_init()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/mm/pat/memtype.c#L241). The dmesg output at boot shows that both are enabled:
```
[    0.000116] MTRR map: 5 entries (4 fixed + 1 variable; max 24), built from 10 variable MTRRs
[    0.000118] x86/PAT: Configuration [0-7]: WB  WC  UC- UC  WB  WP  UC- WT
```
`cache_aps_init()` runs [`cache_rendezvous_handler()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/cacheinfo.c#L749) on each CPU using [`stop_machine()`](https://elixir.bootlin.com/linux/v6.16.6/source/include/linux/stop_machine.h#L98). `cache_rendezvous_handler()` calls [`cache_cpu_init()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/cacheinfo.c#L719). We can instrument it with this change:
```
$ git diff arch/x86/
diff --git a/arch/x86/kernel/cpu/cacheinfo.c b/arch/x86/kernel/cpu/cacheinfo.c
index adfa7e8bb865..4827c1f68220 100644
--- a/arch/x86/kernel/cpu/cacheinfo.c
+++ b/arch/x86/kernel/cpu/cacheinfo.c
@@ -716,6 +716,8 @@ void cache_enable(void) __releases(cache_disable_lock)
        raw_spin_unlock(&cache_disable_lock);
 }

+extern void observe_cpu_speed(const char *when);
+
 static void cache_cpu_init(void)
 {
        unsigned long flags;
@@ -726,10 +728,13 @@ static void cache_cpu_init(void)
                cache_disable();
                mtrr_generic_set_state();
                cache_enable();
+               observe_cpu_speed("cache_cpu_init - after MTRR");
        }

-       if (memory_caching_control & CACHE_PAT)
+       if (memory_caching_control & CACHE_PAT) {
                pat_cpu_init();
+               observe_cpu_speed("cache_cpu_init - after PAT");
+       }

        local_irq_restore(flags);
 }
```
This requires modification of the changes to kernel/cpu.c to export `observe_cpu_speed()`:
```
+extern void observe_cpu_speed(const char *when);
+void observe_cpu_speed(const char *when)
+{
+       int cpu = smp_processor_id();
+
+       pr_info("CPU %d when %s, looping...\n", cpu, when);
+       delay_loop(loops_per_jiffy * 2);
+       pr_info("CPU %d ...done\n", cpu);
+}
+
+static void observe_cpu_speed_(void *info)
+{
+       const char *when = info;
+
+       observe_cpu_speed(when);
+}
+
+extern void observe_cpu_speeds(const char *when);
+void observe_cpu_speeds(const char *when)
+{
+       on_each_cpu(observe_cpu_speed_, (void *)when, 1);
+}
+
```
The CPUs seem to be fast after the MTRR phase:
```
[  106.287596] CPU 0 when cache_cpu_init - after MTRR, looping...
[  106.287994] CPU 4 when cache_cpu_init - after MTRR, looping...
[  106.288713] CPU 5 when cache_cpu_init - after MTRR, looping...
[  106.289108] CPU 1 when cache_cpu_init - after MTRR, looping...
[  106.289822] CPU 6 when cache_cpu_init - after MTRR, looping...
[  106.290227] CPU 2 when cache_cpu_init - after MTRR, looping...
[  106.290762] CPU 0 ...done
[  106.290762] CPU 4 ...done
[  106.290765] CPU 4 when cache_cpu_init - after PAT, looping...
[  106.290765] CPU 0 when cache_cpu_init - after PAT, looping...
[  106.290935] CPU 3 when cache_cpu_init - after MTRR, looping...
[  106.291339] CPU 7 when cache_cpu_init - after MTRR, looping...
[  106.291875] CPU 5 ...done
[  106.291876] CPU 1 ...done
[  106.291877] CPU 5 when cache_cpu_init - after PAT, looping...
[  106.291878] CPU 1 when cache_cpu_init - after PAT, looping...
[  106.292993] CPU 6 ...done
[  106.292995] CPU 6 when cache_cpu_init - after PAT, looping...
[  106.292996] CPU 2 ...done
[  106.292997] CPU 2 when cache_cpu_init - after PAT, looping...
[  106.293532] CPU 4 ...done
[  106.293532] CPU 0 ...done
[  106.294099] CPU 3 ...done
[  106.294100] CPU 7 ...done
[  106.294101] CPU 3 when cache_cpu_init - after PAT, looping...
[  106.294101] CPU 7 when cache_cpu_init - after PAT, looping...
[  106.294637] CPU 5 ...done
[  106.294638] CPU 1 ...done
[  106.295754] CPU 6 ...done
[  106.295756] CPU 2 ...done
[  106.296852] CPU 3 ...done
[  106.296853] CPU 7 ...done
```
### cache_disable()/cache_enable()
Does the speed recover because the cache was disabled and enabled, or because the MTRRs were updated? We can test this by adding additional calls to [`cache_disable()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/cacheinfo.c#L667)/[`cache_enable()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/cacheinfo.c#L700) prior to the ones surrounding the call to [`mtrr_generic_set_state()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/mtrr/generic.c#L964).
```
$ git diff arch/x86
diff --git a/arch/x86/kernel/cpu/cacheinfo.c b/arch/x86/kernel/cpu/cacheinfo.c
index adfa7e8bb865..31166bebb4ed 100644
--- a/arch/x86/kernel/cpu/cacheinfo.c
+++ b/arch/x86/kernel/cpu/cacheinfo.c
@@ -716,6 +716,8 @@ void cache_enable(void) __releases(cache_disable_lock)
        raw_spin_unlock(&cache_disable_lock);
 }

+extern void observe_cpu_speed(const char *when);
+
 static void cache_cpu_init(void)
 {
        unsigned long flags;
@@ -723,13 +725,20 @@ static void cache_cpu_init(void)
        local_irq_save(flags);

        if (memory_caching_control & CACHE_MTRR) {
+               observe_cpu_speed("cache_cpu_init - before cache disable/enable");
+               cache_disable();
+               cache_enable();
+               observe_cpu_speed("cache_cpu_init - before MTRR");
                cache_disable();
                mtrr_generic_set_state();
                cache_enable();
+               observe_cpu_speed("cache_cpu_init - after MTRR");
        }

-       if (memory_caching_control & CACHE_PAT)
+       if (memory_caching_control & CACHE_PAT) {
                pat_cpu_init();
+               observe_cpu_speed("cache_cpu_init - after PAT");
+       }

        local_irq_restore(flags);
 }
```
The CPUs are slow prior to the new cache disable/enable, with CPUs 3 & 7 completing last, overlapped with the subsequent operations on the other CPUs:
```
[  140.398398] CPU 0 when cache_cpu_init - before cache disable/enable, looping...
[  140.398398] CPU 4 when cache_cpu_init - before cache disable/enable, looping...
[  140.398468] CPU 3 when cache_cpu_init - before cache disable/enable, looping...
[  140.398472] CPU 7 when cache_cpu_init - before cache disable/enable, looping...
[  140.398468] CPU 1 when cache_cpu_init - before cache disable/enable, looping...
[  140.398475] CPU 2 when cache_cpu_init - before cache disable/enable, looping...
[  140.398469] CPU 5 when cache_cpu_init - before cache disable/enable, looping...
[  140.398473] CPU 6 when cache_cpu_init - before cache disable/enable, looping...
[  140.455418] CPU 0 ...done
[  140.455529] CPU 0 when cache_cpu_init - before MTRR, looping...
[  140.459974] CPU 4 ...done
[  140.460084] CPU 4 when cache_cpu_init - before MTRR, looping...
[  140.562171] CPU 0 ...done
[  140.562810] CPU 0 when cache_cpu_init - after MTRR, looping...
[  140.575815] CPU 4 ...done
[  140.576487] CPU 4 when cache_cpu_init - after MTRR, looping...
[  140.669944] CPU 0 ...done
[  140.669946] CPU 0 when cache_cpu_init - after PAT, looping...
[  140.691624] CPU 4 ...done
[  140.691627] CPU 4 when cache_cpu_init - after PAT, looping...
[  140.776480] CPU 0 ...done
[  140.812993] CPU 4 ...done
[  143.277756] CPU 2 ...done
[  143.277793] CPU 6 ...done
[  143.278700] CPU 2 when cache_cpu_init - before MTRR, looping...
[  143.278793] CPU 6 when cache_cpu_init - before MTRR, looping...
[  143.307506] CPU 5 ...done
[  143.307554] CPU 1 ...done
[  143.308449] CPU 5 when cache_cpu_init - before MTRR, looping...
[  143.308576] CPU 1 when cache_cpu_init - before MTRR, looping...
[  143.333465] CPU 3 ...done
[  143.334052] CPU 7 ...done
```
After CPUs 3 & 7 complete the cache disable/enable, they still appear to be slow:
```
[  143.334427] CPU 3 when cache_cpu_init - before MTRR, looping...
[  143.334965] CPU 7 when cache_cpu_init - before MTRR, looping...
[  146.070465] CPU 2 ...done
[  146.070711] CPU 6 ...done
[  146.071918] CPU 2 when cache_cpu_init - after MTRR, looping...
[  146.072318] CPU 6 when cache_cpu_init - after MTRR, looping...
[  146.124162] CPU 5 ...done
[  146.124286] CPU 1 ...done
[  146.125632] CPU 5 when cache_cpu_init - after MTRR, looping...
[  146.126028] CPU 1 when cache_cpu_init - after MTRR, looping...
[  146.129336] CPU 2 ...done
[  146.129340] CPU 2 when cache_cpu_init - after PAT, looping...
[  146.130072] CPU 6 ...done
[  146.130074] CPU 6 when cache_cpu_init - after PAT, looping...
[  146.140753] CPU 1 ...done
[  146.140755] CPU 1 when cache_cpu_init - after PAT, looping...
[  146.141226] CPU 5 ...done
[  146.141228] CPU 5 when cache_cpu_init - after PAT, looping...
[  146.142863] CPU 2 ...done
[  146.143868] CPU 6 ...done
[  146.152909] CPU 1 ...done
[  146.153282] CPU 5 ...done
[  146.167999] CPU 3 ...done
[  146.168491] CPU 7 ...done
```
After CPUs 3 & 7 complete the MTRR updates, they are fast:
```
[  146.169410] CPU 3 when cache_cpu_init - after MTRR, looping...
[  146.169811] CPU 7 when cache_cpu_init - after MTRR, looping...
[  146.172572] CPU 7 ...done
[  146.172572] CPU 3 ...done
[  146.172574] CPU 3 when cache_cpu_init - after PAT, looping...
[  146.172574] CPU 7 when cache_cpu_init - after PAT, looping...
[  146.175325] CPU 7 ...done
[  146.175325] CPU 3 ...done
```
So it seems that setting the MTRRs is what recovers the speed. The speeds of the CPUs doesn't appear to be recovered independently. They appear to speed up after the MTRRs are written, but not to full speed until this has happened on all CPUs.

### MTRR inconsistency?

Maybe the underlying problem is that the MTRRs become inconsistent on suspend/resume, and only once they have been updated on all CPUs does the system run at full speed.

The MTRRs don't appear to be inconsistent at boot, as there are no warnings printed by [`mtrr_state_warn()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/mtrr/generic.c#L730), called by [`mtrr_init_finalize()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/mtrr/mtrr.c#L625).

### Speed recovery end note
Some of the time we've been running with this change to the ACPI code:
```
diff --git a/drivers/acpi/ec.c b/drivers/acpi/ec.c
index 7855bbf752b1..bc7ec0bb637e 100644
--- a/drivers/acpi/ec.c
+++ b/drivers/acpi/ec.c
@@ -13,7 +13,7 @@
  */

 /* Uncomment next line to get verbose printout */
-/* #define DEBUG */
+#define DEBUG
 #define pr_fmt(fmt) "ACPI: EC: " fmt

 #include <linux/kernel.h>
```
It can bloat the dmesg output.

## Kernel version

Looking at the history of `arch/x86/kernel/cpu/mtrr/`, there were quite a few changes in the 6.2 & 6.5 cycles. It might therefore be worth trying older kernels to see if they behave any differently (if they work at all on my hardware, and on my modern user space).

We could try 6.1.104 (the most recent 6.1 kernel before some mtrr fixes were backported in 6.1.105 & 6.1.142).

### Shorter compile time
To shorten compile times, and to try and fix the large size of the genereated initrd filling up the /boot/efi system, we can remove unused modules from the configuration. After ensuring all hardware we need is plugged in:
```
$ cp .config .config-6.16.7-pre-localmodconfig
$ make localmodconfig
using config: '.config'
brcmfmac_wcc config not found!!
system76_io config not found!!
*
* Restart config...
...

$ make -j$(nproc)
$ sudo make modules_install
$ sudo make install
$ sudo bash -c 'rm /boot/efi/EFI/Pop_OS-*/*-previous*'
$ sudo kernelstub -k /boot/vmlinuz-6.16.7-custom-dirty -i /boot/initrd.img-6.16.7-custom-dirty
$ reboot
```
We don't need to worry about brcmfmac_wcc. We can see that it is built as part of the overall brcmfmac driver. See the brcmfmac top level [`Makefile`](https://elixir.bootlin.com/linux/v6.16.6/source/drivers/net/wireless/broadcom/brcm80211/brcmfmac/Makefile#L55), and the `wcc` subdirectory [`Makefile`](https://elixir.bootlin.com/linux/v6.16.6/source/drivers/net/wireless/broadcom/brcm80211/brcmfmac/wcc/Makefile).

The system76-io module appears to be provided by [DKMS](https://en.wikipedia.org/wiki/Dynamic_Kernel_Module_Support), and so should be rebuilt automatically:
```
$ cat /usr/share/initramfs-tools/modules.d/system76-io-dkms.conf
system76-io
```
The new kernel boots okay, has only 162 modules built (previously it was >6000), of which 124 are loaded. It still shows the delayed resume problem.

### v6.1.104
We're going to build our own kernel as we've already been doing (currently for v6.16.7). Where do we get a kernel config for v6.1.104? Ubuntu has a mainline kernel build [archive](https://kernel.ubuntu.com/mainline/?C=N;O=D), which has a [v6.1.100](https://kernel.ubuntu.com/mainline/v6.1.100/) build.

The kernel-headers-x.y.z-nnnn-generic package contains the kernel configuration file. We can download the package and extract it:
```
$ dpkg --contents ~/Downloads/linux-headers-6.1.100-0601100-generic_6.1.100-0601100.202407181146_amd64.deb | grep '\.config'
-rw-r--r-- root/root    273294 2024-07-18 12:46 ./usr/src/linux-headers-6.1.100-0601100-generic/.config
dpkg --fsys-tarfile ~/Downloads/linux-headers-6.1.100-0601100-generic_6.1.100-0601100.202407181146_amd64.deb | tar -xO ./usr/src/linux-headers-6.1.100-0601100-generic/.config > .config-6.1.100-ubuntu
```
Save our existing experiments and configure & build v6.1.104:
```
$ mv .config .config-6.16.7-localmodconfig
$ git stash push -m 6.16.7-debug
$ git checkout v6.1.104
$ make mrproper
$ cp .config-6.1.100-ubuntu .config
$ make oldconfig
$ make menuconfig
$ grep -e LOCALVERSION -e SYSTEM_TRUSTED_KEYS -e SYSTEM_REVOCATION_KEYS .config
CONFIG_LOCALVERSION="-custom"
CONFIG_LOCALVERSION_AUTO=y
CONFIG_SYSTEM_TRUSTED_KEYS=""
CONFIG_SYSTEM_REVOCATION_KEYS=""
$ make localmodconfig
$ make -j$(nproc)
$ sudo make modules_install
$ sudo make install
$ sudo kernelstub -k /boot/vmlinuz-6.1.104-custom -i /boot/initrd.img-6.1.104-custom
$ reboot
...
$ uname -r
6.1.104-custom
```
The delay bringing up the secondary CPUs is there, but not *as* slow on the first suspend/resume:
```
[  131.836126] Enabling non-boot CPUs ...
[  131.836487] x86: Booting SMP configuration:
[  131.836489] smpboot: Booting Node 0 Processor 1 APIC 0x2
[  132.182794] CPU1 is up
[  132.186686] smpboot: Booting Node 0 Processor 2 APIC 0x4
[  132.550331] CPU2 is up
[  132.553422] smpboot: Booting Node 0 Processor 3 APIC 0x6
[  132.908839] CPU3 is up
[  132.910797] smpboot: Booting Node 0 Processor 4 APIC 0x1
[  132.918616] CPU4 is up
[  132.918771] smpboot: Booting Node 0 Processor 5 APIC 0x3
[  133.350633] CPU5 is up
[  133.352848] smpboot: Booting Node 0 Processor 6 APIC 0x5
[  133.876797] CPU6 is up
[  133.878276] smpboot: Booting Node 0 Processor 7 APIC 0x7
[  134.543099] CPU7 is up
```
We've seen variable behaviour like this before, but it does seem to be consistently about 6s from lid open to the unlock password prompt, which is a lot better than the previous 20s.

We'd have to run with this kernel for a while and/or switch between kernel versions a few times to determine if the change in version triggers a *consistent* change in behavior.

If we determine that there *is* a consistent difference, then we could bisect for the change that made it worse, which might shed light on why it's bad in the first place.

### v4.19.12
Attempting to go further back, I tried 4.19.12 (the most recent 4.19 kernel before some mtrr fixes were backported in 4.19.13, 4.19.167 & 4.19.320).

We can take the config from the Ubuntu mainline build archive for [4.19.10](https://kernel.ubuntu.com/mainline/v4.19.10/).

```
$ dpkg --contents ~/Downloads/linux-headers-4.19.10-041910-generic_4.19.10-041910.201812170433_amd64.deb | grep '\.config'
-rw-r--r-- root/root    219485 2018-12-17 09:53 ./usr/src/linux-headers-4.19.10-041910-generic/.config
-rw-r--r-- root/root    219609 2018-12-17 09:53 ./usr/src/linux-headers-4.19.10-041910-generic/.config.old
$ dpkg --fsys-tarfile ~/Downloads/linux-headers-4.19.10-041910-generic_4.19.10-041910.201812170433_amd64.deb | tar -xO ./usr/src/linux-headers-4.19.10-041910-generic/.config > .config-4.19.10-ubuntu
$ mv .config .config-6.1.104-localmodconfig
$ git checkout v4.19.12
$ make mrproper
$ cp .config-4.19.10-ubuntu .config
$ make oldconfig
$ make menuconfig
$ grep -e LOCALVERSION -e SYSTEM_TRUSTED_KEYS -e SYSTEM_REVOCATION_KEYS .config
CONFIG_LOCALVERSION="-custom"
CONFIG_LOCALVERSION_AUTO=y
CONFIG_SYSTEM_TRUSTED_KEYS=""
$ make localmodconfig
$ make -j$(nproc)
```
However, it didn't build with the gcc-13.3.0 toolchain, and it's not worth trying to backport the fixes required to make it work.

### v6.16.7
I recompiled v6.16.7 without the previous modification for a fairer comparison with v6.1.104.
```
$ mv .config .config-4.19.12-localmodconfig
$ git checkout v6.16.7
$ make mrproper
$ cp .config-6.16.7-localmodconfig .config
$ make oldconfig
$ make -j$(nproc)
$ sudo make modules_install
$ sudo make install
$ sudo kernelstub -k /boot/vmlinuz-6.16.7-custom -i /boot/initrd.img-6.16.7-custom
$ reboot
...
$ uname -r
6.16.7-custom
```
The resume delay is back to 17-20s, so the level of delay does seem to be affected by the kernel version.

### v6.6.45
Attempting to bisect, I tried v6.6.45 (the most recent 6.6 kernel before some mtrr fixes were backported in 6.6.46 & 6.6.94).

We can take the config from the Ubuntu mainline build archive for [6.6.45](https://kernel.ubuntu.com/mainline/v6.6.45/).

```
$ dpkg --contents ~/Downloads/linux-headers-6.6.45-060645-generic_6.6.45-060645.202408111159_amd64.deb | grep '\.config'
-rw-r--r-- root/root    282451 2024-08-11 12:59 ./usr/src/linux-headers-6.6.45-060645-generic/.config
$ dpkg --fsys-tarfile ~/Downloads/linux-headers-6.6.45-060645-generic_6.6.45-060645.202408111159_amd64.deb | tar -xO ./usr/src/linux-headers-6.6.45-060645-generic/.config > .config-6.6.45-ubuntu
$ git checkout v6.6.45
$ make mrproper
$ cp .config-6.6.45-ubuntu .config
$ make oldconfig
$ make menuconfig
$ grep -e LOCALVERSION -e SYSTEM_TRUSTED_KEYS -e SYSTEM_REVOCATION_KEYS .config
CONFIG_LOCALVERSION="-custom"
CONFIG_LOCALVERSION_AUTO=y
CONFIG_SYSTEM_TRUSTED_KEYS=""
CONFIG_SYSTEM_REVOCATION_KEYS=""
$ make localmodconfig
$ cp .config .config-6.6.45-localmodconfig
$ make -j$(nproc)
$ sudo make modules_install
$ sudo make install
$ sudo kernelstub -k /boot/vmlinuz-6.6.45-custom -i /boot/initrd.img-6.6.45-custom
$ reboot
...
$ uname -r
6.6.45-custom
```
The resume time is about 6s, as for 6.1.104.

### v6.11.11
To further bisect, we need to look at the changes *after* the large changes to arch/x86/kernel/cpu/mtrr/ in 6.2.x & 6.5.x.

The remaining changes between 6.6.x (we've just shown 6.6.45 has a 6s resume time) and 6.16.7 (with the ~20s resume time) are spread out over versions 6.8.x, 6.9.x, 6.11.x, 6.12.x, 6.14.x, 6.15.x.

Looking at them and taking an educated guess as to which is most significant, the next target is v6.11.x, and specifically v6.11.11.

We can take the config from the Ubuntu mainline build archive for [6.11.11](https://kernel.ubuntu.com/mainline/v6.11.11/).

```
$ dpkg --contents ~/Downloads/linux-headers-6.11.11-061111-generic_6.11.11-061111.202412051415_amd64.deb | grep '\.config'
-rw-r--r-- root/root    291143 2024-12-05 14:15 ./usr/src/linux-headers-6.11.11-061111-generic/.config
$ dpkg --fsys-tarfile ~/Downloads/linux-headers-6.11.11-061111-generic_6.11.11-061111.202412051415_amd64.deb | tar -xO ./usr/src/linux-headers-6.11.11-061111-generic/.config > .config-6.11.11-ubuntu
$ git checkout v6.11.11
$ make mrproper
$ cp .config-6.11.11-ubuntu .config
$ make oldconfig
$ make menuconfig
$ grep -e LOCALVERSION -e SYSTEM_TRUSTED_KEYS -e SYSTEM_REVOCATION_KEYS .config
CONFIG_LOCALVERSION="-custom"
CONFIG_LOCALVERSION_AUTO=y
CONFIG_SYSTEM_TRUSTED_KEYS=""
CONFIG_SYSTEM_REVOCATION_KEYS=""
$ make localmodconfig
$ cp .config .config-6.11.11-localmodconfig
$ make -j$(nproc)
$ sudo make modules_install
$ sudo make install
$ sudo kernelstub -k /boot/vmlinuz-6.11.11-custom -i /boot/initrd.img-6.11.11-custom
$ reboot
...
$ uname -r
6.11.11-custom
```
The resume time is 18-20s, as for 6.16.7.

### v6.9.12

The next bisection target is v6.9.x, specifically v6.9.12.

We can take the config from the Ubuntu mainline build archive for [6.9.12](https://kernel.ubuntu.com/mainline/v6.9.12/).

```
$ dpkg --contents ~/Downloads/linux-headers-6.9.12-060912-generic_6.9.12-060912.202502190936_amd64.deb | grep '\.config'
-rw-r--r-- root/root    287954 2025-02-19 09:36 ./usr/src/linux-headers-6.9.12-060912-generic/.config
$ dpkg --fsys-tarfile ~/Downloads/linux-headers-6.9.12-060912-generic_6.9.12-060912.202502190936_amd64.deb | tar -xO ./usr/src/linux-headers-6.9.12-060912-generic/.config > .config-6.9.12-ubuntu
$ git checkout v6.9.12
$ make mrproper
$ cp .config-6.9.12-ubuntu .config
$ make oldconfig
$ make menuconfig
$ grep -e LOCALVERSION -e SYSTEM_TRUSTED_KEYS -e SYSTEM_REVOCATION_KEYS .config
CONFIG_LOCALVERSION="-custom"
CONFIG_LOCALVERSION_AUTO=y
CONFIG_SYSTEM_TRUSTED_KEYS=""
CONFIG_SYSTEM_REVOCATION_KEYS=""
$ make localmodconfig
$ cp .config .config-6.9.12-localmodconfig
$ make -j$(nproc)
$ sudo make modules_install
$ sudo make install
$ sudo kernelstub -k /boot/vmlinuz-6.9.12-custom -i /boot/initrd.img-6.9.12-custom
$ reboot
...
$ uname -r
6.9.12-custom
```
The resume time is 35-40s, even *worse* than the ~20s for 6.11.11 and later.

### v6.8.12

The next bisection target is v6.8.x, specifically v6.8.12.

We can take the config from the Ubuntu mainline build archive for [6.8.12](https://kernel.ubuntu.com/mainline/v6.8.12/).

```
$ dpkg --contents ~/Downloads/linux-headers-6.8.12-060812-generic_6.8.12-060812.202501300202_amd64.deb | grep '\.config'
-rw-r--r-- root/root    286395 2025-01-30 02:02 ./usr/src/linux-headers-6.8.12-060812-generic/.config
$ dpkg --fsys-tarfile ~/Downloads/linux-headers-6.8.12-060812-generic_6.8.12-060812.202501300202_amd64.deb | tar -xO ./usr/src/linux-headers-6.8.12-060812-generic/.config > .config-6.8.12-ubuntu
$ git checkout v6.8.12
$ make mrproper
$ cp .config-6.8.12-ubuntu .config
$ make oldconfig
$ make menuconfig
$ grep -e LOCALVERSION -e SYSTEM_TRUSTED_KEYS -e SYSTEM_REVOCATION_KEYS .config
CONFIG_LOCALVERSION="-custom"
CONFIG_LOCALVERSION_AUTO=y
CONFIG_SYSTEM_TRUSTED_KEYS=""
CONFIG_SYSTEM_REVOCATION_KEYS=""
$ make localmodconfig
$ cp .config .config-6.8.12-localmodconfig
$ make -j$(nproc)
$ sudo make modules_install
$ sudo make install
$ sudo kernelstub -k /boot/vmlinuz-6.8.12-custom -i /boot/initrd.img-6.8.12-custom
$ reboot
...
$ uname -r
6.8.12-custom
```
The resume time is 4s, even *better* than the 6s for 6.6.45 and earlier.

The CPU bring up is still delayed, but not as much as in *say* 6.1.104:
```
[   62.505591] Enabling non-boot CPUs ...
[   62.505997] smpboot: Booting Node 0 Processor 1 APIC 0x2
[   62.637618] CPU1 is up
[   62.638449] smpboot: Booting Node 0 Processor 2 APIC 0x4
[   62.763161] CPU2 is up
[   62.764060] smpboot: Booting Node 0 Processor 3 APIC 0x6
[   62.947180] CPU3 is up
[   62.948075] smpboot: Booting Node 0 Processor 4 APIC 0x1
[   62.955234] CPU4 is up
[   62.955390] smpboot: Booting Node 0 Processor 5 APIC 0x3
[   63.019158] CPU5 is up
[   63.019424] smpboot: Booting Node 0 Processor 6 APIC 0x5
[   63.106233] CPU6 is up
[   63.106541] smpboot: Booting Node 0 Processor 7 APIC 0x7
[   63.215537] CPU7 is up
[   63.220436] ACPI: PM: Waking up from system sleep state S3
```

Notably, the time to *suspend* (they time from closing the lid until the apple logo goes dark) seems to be longer than in other runs, although I'd have to revisit v6.1.104 to confirm the suspend behaviour of that version.

### Version summary

This table includes the results to date, and additional results from bisections described below:

| Version | Resume time (s) |
| - | - |
| v4.19.12 | *doesn't compile* |
| v6.1.104 | 6 |
| v6.6.45 | 6, later 6 |
| v6.8.0 | 28-35 |
| v6.8.5-80-g59813a65b721 | 30-36 |
| v6.8.9-116-gb6735bfe9414 | 19-38 |
| v6.8.11-123-gfea4aea0fbc3 | 21-37 |
| v6.8.11-308-g5e71608df742 | 19-31 |
| v6.8.11-401-gb77620730f61 | 21-38 |
| v6.8.11-447-g8c0a305b7603 | 26-34 |
| v6.8.11-470-g8607fbc7bb7f | 31-36 |
| v6.8.11-482-gb86dcd478264 | 20-37 |
| v6.8.11-488-gf2205e838c31 | 29-38 |
| v6.8.11-491-g32db10471618 | 25-32 |
| v6.8.11-493-g40503e604570 | 32-35 |
| v6.8.12 | 4s, later 29 |
| v6.9.12 | 35-40 |
| v6.11.11 | 18-20 |
| v6.16.7 | 17-20, but 5 for a period |

## Regression bisection : 6.8.0 - 6.8.12

The changes between v6.8.12 and v6.9.12 in arch/x86/kernel/cpu/mtrr/ don't seem likely to be the likely cause of the regression in resume time from 4s to 35-40s. It's more likely to be something else in the arch/x86 tree, perhaps a change that makes much more code run during CPU bringup and before the MTRRs are restored to a consistent state.

We can try a full kernel bisection between v6.8 and v6.9.12.

### v6.8.0

First we have to test v6.8 to verify that it behaves like v6.8.12.

For this bisection we'll use the same kernel config as we used for v6.8.12.

```
$ git checkout v6.8
$ make mrproper
$ cp .config-6.8.12-localmodconfig .config
$ make oldconfig
$ cp .config .config-6.8-localmodconfig
$ make -j$(nproc)
$ sudo make modules_install
$ sudo make install
$ sudo kernelstub -k /boot/vmlinuz-6.8.0-custom -i /boot/initrd.img-6.8.0-custom
$ reboot
...
$ uname -r
6.8.0-custom
```

The resume time is 28-35s, worse than the 4s for 6.8.12, so perhaps there is something that reduced the resume time in the 6.8.x series.

We can try bisecting for that change now.

### v6.8.5-80-g59813a65b721

```
$ git bisect start --term-old=slow --term-new=fast
status: waiting for both good and bad commits
$ git bisect slow
status: waiting for bad commit, 1 good commit known
$ git bisect fast v6.8.12
Bisecting: 1483 revisions left to test after this (roughly 11 steps)
[59813a65b72197884ba26f723dab5248419d0f9e] ext4: add a hint for block bitmap corrupt state in mb_groups
$ gitk
Follows: v6.8.5
Precedes: v6.8.6
$ git describe
v6.8.5-80-g59813a65b721
$ make oldconfig
$ cp .config .config-v6.8.5-80-g59813a65b721-localmodconfig
$ make -j$(nproc)
$ sudo make modules_install
$ sudo make install
$ sudo kernelstub -k /boot/vmlinuz-6.8.5-custom-00080-g59813a65b721 -i /boot/initrd.img-6.8.5-custom-00080-g59813a65b721
$ reboot
...
$ uname -r
6.8.5-custom-00080-g59813a65b721
```

Resume time is 30-36s.

### v6.8.9-116-gb6735bfe9414
```
$ git bisect slow
Bisecting: 741 revisions left to test after this (roughly 10 steps)
[b6735bfe941486c5dfc9c3085d2d75d4923f9449] drm/amdkfd: range check cp bad op exception interrupts
$ gitk
Follows: v6.8.9
Precedes: v6.8.10
$ git describe
v6.8.9-116-gb6735bfe9414
$ make oldconfig
$ cp .config .config-v6.8.9-116-gb6735bfe9414-localmodconfig
$ make -j$(nproc)
$ sudo make modules_install
$ sudo make install
$ sudo kernelstub -k /boot/vmlinuz-6.8.9-custom-00116-gb6735bfe9414 -i /boot/initrd.img-6.8.9-custom-00116-gb6735bfe9414
$ reboot
...
$ uname -r
6.8.9-custom-00116-gb6735bfe9414
```

Resume time is 19-38s.

### v6.8.11-123-gfea4aea0fbc3
```
$ git bisect slow
Bisecting: 370 revisions left to test after this (roughly 9 steps)
[fea4aea0fbc3d1481f689e8dbe0cac52102da755] io_uring: use the right type for work_llist empty check
$ gitk
Follows: v6.8.11
Precedes: v6.8.12
$ git describe
v6.8.11-123-gfea4aea0fbc3
$ make oldconfig
$ cp .config .config-v6.8.11-123-gfea4aea0fbc3-localmodconfig
$ make -j$(nproc)
$ sudo make modules_install
$ sudo make install
$ sudo kernelstub -k /boot/vmlinuz-6.8.11-custom-00123-gfea4aea0fbc3 -i /boot/initrd.img-6.8.11-custom-00123-gfea4aea0fbc3
$ reboot
...
$ uname -r
6.8.11-custom-00123-gfea4aea0fbc3
```

Resume time is 21-37s.

### v6.8.11-308-g5e71608df742


```
$ git bisect slow
Bisecting: 185 revisions left to test after this (roughly 8 steps)
[5e71608df742d09fcd601cff0435a68259a58904] net: ethernet: cortina: Locking fixes
$ gitk
Follows: v6.8.11
Precedes: v6.8.12
$ git describe
v6.8.11-308-g5e71608df742
$ make oldconfig
$ cp .config .config-v6.8.11-308-g5e71608df742-localmodconfig
$ make -j$(nproc)
$ sudo make modules_install
$ sudo make install
$ sudo kernelstub -k /boot/vmlinuz-6.8.11-custom-00308-g5e71608df742 -i /boot/initrd.img-6.8.11-custom-00308-g5e71608df742
$ reboot
...
$ uname -r
6.8.11-custom-00308-g5e71608df742
```

Resume time is 19-31s.

### v6.8.11-401-gb77620730f61

```
$ git bisect slow
Bisecting: 92 revisions left to test after this (roughly 7 steps)
[b77620730f614059db2470e8ebab3e725280fc6d] drm/arm/malidp: fix a possible null pointer dereference
$ gitk
Follows: v6.8.11
Precedes: v6.8.12
$ git describe
v6.8.11-401-gb77620730f61
$ make oldconfig
$ cp .config .config-v6.8.11-401-gb77620730f61-localmodconfig
$ make -j$(nproc)
$ sudo make modules_install
$ sudo make install
$ sudo kernelstub -k /boot/vmlinuz-6.8.11-custom-00401-gb77620730f61 -i /boot/initrd.img-6.8.11-custom-00401-gb77620730f61
$ reboot
...
$ uname -r
6.8.11-custom-00401-gb77620730f61
```

Resume time is 21-38s.

### v6.8.11-447-g8c0a305b7603
```
$ git bisect slow
Bisecting: 46 revisions left to test after this (roughly 6 steps)
[8c0a305b76035eaba9c81187b78e5b1a334d0171] clk: qcom: mmcc-msm8998: fix venus clock issue
$ gitk
Follows: v6.8.11
Precedes: v6.8.12
$ git describe
v6.8.11-447-g8c0a305b7603
$ make oldconfig
$ cp .config .config-v6.8.11-447-g8c0a305b7603-localmodconfig
$ make -j$(nproc)
$ sudo make modules_install
$ sudo make install
$ sudo kernelstub -k /boot/vmlinuz-6.8.11-custom-00447-g8c0a305b7603 -i /boot/initrd.img-6.8.11-custom-00447-g8c0a305b7603
$ reboot
...
$ uname -r
6.8.11-custom-00447-g8c0a305b7603
```

Resume time is 26-34s.

### v6.8.11-470-g8607fbc7bb7f
```
$ git bisect slow
Bisecting: 23 revisions left to test after this (roughly 5 steps)
[8607fbc7bb7fb0c7f7ba045c09b6d87dca49a2a6] samples/landlock: Fix incorrect free in populate_ruleset_net
$ gitk
Follows: v6.8.11
Precedes: v6.8.12
$ git describe
v6.8.11-470-g8607fbc7bb7f
$ make oldconfig
$ cp .config .config-v6.8.11-470-g8607fbc7bb7f-localmodconfig
$ make -j$(nproc)
$ sudo make modules_install
$ sudo make install
$ sudo kernelstub -k /boot/vmlinuz-6.8.11-custom-00470-g8607fbc7bb7f -i /boot/initrd.img-6.8.11-custom-00470-g8607fbc7bb7f
$ reboot
...
$ uname -r
6.8.11-custom-00470-g8607fbc7bb7f
```

Resume time is 31-36s.

### v6.8.11-482-gb86dcd478264
```
$ git bisect slow
Bisecting: 11 revisions left to test after this (roughly 4 steps)
[b86dcd478264ea4a03a2e91b2fa50d58eb4d513f] sched/fair: Allow disabling sched_balance_newidle with sched_relax_domain_level
$ gitk
Follows: v6.8.11
Precedes: v6.8.12
$ git describe
v6.8.11-482-gb86dcd478264
$ make oldconfig
$ cp .config .config-v6.8.11-482-gb86dcd478264-localmodconfig
$ make -j$(nproc)
$ sudo make modules_install
$ sudo make install
$ sudo kernelstub -k /boot/vmlinuz-6.8.11-custom-00482-gb86dcd478264 -i /boot/initrd.img-6.8.11-custom-00482-gb86dcd478264
$ reboot
...
$ uname -r
6.8.11-custom-00482-gb86dcd478264
```

Resume time is 20-37s.

### v6.8.11-488-gf2205e838c31
```
$ git bisect slow
Bisecting: 5 revisions left to test after this (roughly 3 steps)
[f2205e838c31e8ed690c0acefc909c56e3b19e1c] net: txgbe: fix to control VLAN strip
$ gitk
Follows: v6.8.11
Precedes: v6.8.12
$ git describe
v6.8.11-488-gf2205e838c31
$ make oldconfig
$ cp .config .config-v6.8.11-488-gf2205e838c31-localmodconfig
$ make -j$(nproc)
$ sudo make modules_install
$ sudo make install
$ sudo kernelstub -k /boot/vmlinuz-6.8.11-custom-00488-gf2205e838c31 -i /boot/initrd.img-6.8.11-custom-00488-gf2205e838c31
$ reboot
...
$ uname -r
6.8.11-custom-00482-gb86dcd478264
```

Resume time is 29-38s.

### v6.8.11-491-g32db10471618
```
$ git bisect slow
Bisecting: 2 revisions left to test after this (roughly 2 steps)
[32db1047161806460b4b4066da1b32c9a8b46e4d] pwm: Fix setting period with #pwm-cells = <1> and of_pwm_single_xlate()
$ gitk
Follows: v6.8.11
Precedes: v6.8.12
$ git describe
v6.8.11-491-g32db10471618
$ make oldconfig
$ cp .config .config-v6.8.11-491-g32db10471618-localmodconfig
$ make -j$(nproc)
$ sudo make modules_install
$ sudo make install
$ sudo kernelstub -k /boot/vmlinuz-6.8.11-custom-00491-g32db10471618 -i /boot/initrd.img-6.8.11-custom-00491-g32db10471618
$ reboot
...
$ uname -r
6.8.11-custom-00491-g32db10471618
```

Resume time is 25-32s.

### v6.8.11-493-g40503e604570
```
$ git bisect slow
Bisecting: 0 revisions left to test after this (roughly 1 step)
[40503e6045704b0a4f071568ffaabb2eee3d9431] net: txgbe: fix GPIO interrupt blocking
$ gitk
Follows: v6.8.11
Precedes: v6.8.12
$ git describe
v6.8.11-493-g40503e604570
$ make oldconfig
$ cp .config .config-v6.8.11-493-g40503e604570-localmodconfig
$ make -j$(nproc)
$ sudo make modules_install
$ sudo make install
$ sudo kernelstub -k /boot/vmlinuz-6.8.11-custom-00493-g40503e604570 -i /boot/initrd.img-6.8.11-custom-00493-g40503e604570
$ reboot
...
$ uname -r
6.8.11-custom-00493-g40503e604570
```

Resume time is 32-35s.

### Implausible result:

```
$ git bisect slow
632428373bea7581869cb05dce40bef0d37793e3 is the first fast commit
commit 632428373bea7581869cb05dce40bef0d37793e3
Author: Greg Kroah-Hartman <gregkh@linuxfoundation.org>
Date:   Thu May 30 09:49:53 2024 +0200

    Linux 6.8.12

    Link: https://lore.kernel.org/r/20240527185626.546110716@linuxfoundation.org
    Tested-by: Salvatore Bonaccorso <carnil@debian.org>
    Tested-by: Bagas Sanjaya <bagasdotme@gmail.com>
    Tested-by: Jon Hunter <jonathanh@nvidia.com>
    Tested-by: Mark Brown <broonie@kernel.org>
    Tested-by: SeongJae Park <sj@kernel.org>
    Tested-by: Florian Fainelli <florian.fainelli@broadcom.com>
    Tested-by: Shuah Khan <skhan@linuxfoundation.org>
    Tested-by: Ron Economos <re@w6rz.net>
    Tested-by: Linux Kernel Functional Testing <lkft@linaro.org>
    Signed-off-by: Greg Kroah-Hartman <gregkh@linuxfoundation.org>

 Makefile | 2 +-
 1 file changed, 1 insertion(+), 1 deletion(-)
 ```
 That's unlikely!
 ```
 $ git bisect log
git bisect start '--term-old=slow' '--term-new=fast'
# status: waiting for both good and bad commits
# slow: [e8f897f4afef0031fe618a8e94127a0934896aba] Linux 6.8
git bisect slow e8f897f4afef0031fe618a8e94127a0934896aba
# status: waiting for bad commit, 1 good commit known
# fast: [632428373bea7581869cb05dce40bef0d37793e3] Linux 6.8.12
git bisect fast 632428373bea7581869cb05dce40bef0d37793e3
# slow: [59813a65b72197884ba26f723dab5248419d0f9e] ext4: add a hint for block bitmap corrupt state in mb_groups
git bisect slow 59813a65b72197884ba26f723dab5248419d0f9e
# slow: [b6735bfe941486c5dfc9c3085d2d75d4923f9449] drm/amdkfd: range check cp bad op exception interrupts
git bisect slow b6735bfe941486c5dfc9c3085d2d75d4923f9449
# slow: [fea4aea0fbc3d1481f689e8dbe0cac52102da755] io_uring: use the right type for work_llist empty check
git bisect slow fea4aea0fbc3d1481f689e8dbe0cac52102da755
# slow: [5e71608df742d09fcd601cff0435a68259a58904] net: ethernet: cortina: Locking fixes
git bisect slow 5e71608df742d09fcd601cff0435a68259a58904
# slow: [b77620730f614059db2470e8ebab3e725280fc6d] drm/arm/malidp: fix a possible null pointer dereference
git bisect slow b77620730f614059db2470e8ebab3e725280fc6d
# slow: [8c0a305b76035eaba9c81187b78e5b1a334d0171] clk: qcom: mmcc-msm8998: fix venus clock issue
git bisect slow 8c0a305b76035eaba9c81187b78e5b1a334d0171
# slow: [8607fbc7bb7fb0c7f7ba045c09b6d87dca49a2a6] samples/landlock: Fix incorrect free in populate_ruleset_net
git bisect slow 8607fbc7bb7fb0c7f7ba045c09b6d87dca49a2a6
# slow: [b86dcd478264ea4a03a2e91b2fa50d58eb4d513f] sched/fair: Allow disabling sched_balance_newidle with sched_relax_domain_level
git bisect slow b86dcd478264ea4a03a2e91b2fa50d58eb4d513f
# slow: [f2205e838c31e8ed690c0acefc909c56e3b19e1c] net: txgbe: fix to control VLAN strip
git bisect slow f2205e838c31e8ed690c0acefc909c56e3b19e1c
# slow: [32db1047161806460b4b4066da1b32c9a8b46e4d] pwm: Fix setting period with #pwm-cells = <1> and of_pwm_single_xlate()
git bisect slow 32db1047161806460b4b4066da1b32c9a8b46e4d
# slow: [40503e6045704b0a4f071568ffaabb2eee3d9431] net: txgbe: fix GPIO interrupt blocking
git bisect slow 40503e6045704b0a4f071568ffaabb2eee3d9431
# first fast commit: [632428373bea7581869cb05dce40bef0d37793e3] Linux 6.8.12
```

### retest
#### 6.8.12
```
$ sudo kernelstub -k /boot/vmlinuz-6.8.12-custom -i /boot/initrd.img-6.8.12-custom
$ reboot
...
$ uname -r
6.8.12-custom
```

Resume time is 29s.

Although I tested 6.8.12 several times, including after a reboot, it's behaving differently now.

#### 6.6.45
```
$ sudo kernelstub -k /boot/vmlinuz-6.6.45-custom -i /boot/initrd.img-6.6.45-custom
$ reboot
...
$ uname -r
6.6.45-custom
```

Resume time is 6s.

So it's only 6.8.12 at 4s that was an anomoly, the 6s for 6.6.45 is consistent. 

The *suspend* time is noticeably longer, as it was for 6.8.12 when it was resuming in 4s, but not when it was resuming in 29s.

## New Bisection?

We have a choice of what to bisect next.

### Version summary

| Version | Resume time (s) |
| - | - |
| v4.19.12 | *doesn't compile* |
| v6.1.104 | 6 |
| v6.6.45 | 6, later 6 |
| v6.7.0 | 3 |
| v6.8.0-12 | 19-38, but 4 for a period |
| v6.9.12 | 35-40 |
| v6.11.11 | 18-20 |
| v6.16.7 | 17-20, but 5 for a period |

We could bisect the 6s to ~30s change between 6.6.45 and 6.8.0.

We could bisect the >30s to <=20s change between 6.9.12 and 6.16.7.

### v6.7.0

```
$ git checkout v6.7
$ make mrproper
$ cp .config-6.6.45-localmodconfig .config
$ make oldconfig
$ cp .config .config-6.7-localmodconfig
$ make -j$(nproc)
$ sudo make modules_install
$ sudo make install
$ sudo kernelstub -k /boot/vmlinuz-6.7.0-custom -i /boot/initrd.img-6.7.0-custom
$ reboot
...
$ uname -r
6.7.0-custom
```

The resume time is 3s, but retesting 6.8.0 afterwards also yields 4s, unlike the previously observed ~30s.

Maybe some of the versions have short resume times, but they can be poisoned by running a later version that *mostly* have longer resume times.

It looks like trying to bisect is not going to be feasible.

## MTRR values

We might do better by instrumenting the values of the MTRRs, and hopefully being able to see differences between the shorter and longer resume times.

### Values on boot

First, let's see the effect of putting `mtrr=debug` on the kernel command line (setting [`mtrr_debug`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/mtrr/generic.c#L46)) in mtrr/generic.c:

```
$ sudo kernelstub -k /boot/vmlinuz-6.16.7-custom -i /boot/initrd.img-6.16.7-custom -a "mtrr=debug"
$ reboot
...
$ sudo dmesg
...
[    0.000112] MTRR default type: write-back
[    0.000113] MTRR fixed ranges enabled:
[    0.000114]   00000-9FFFF write-back
[    0.000115]   A0000-BFFFF uncachable
[    0.000116]   C0000-DFFFF write-protect
[    0.000117]   E0000-FFFFF uncachable
[    0.000117] MTRR variable ranges enabled:
[    0.000118]   0 base 0000080000000 mask 0007F80000000 uncachable
[    0.000120]   1 base 000007C000000 mask 0007FFC000000 uncachable
[    0.000121]   2 base 000007A000000 mask 0007FFE000000 uncachable
[    0.000121]   3 base 0000079000000 mask 0007FFF000000 uncachable
[    0.000122]   4 disabled
[    0.000123]   5 disabled
[    0.000123]   6 disabled
[    0.000124]   7 disabled
[    0.000124]   8 disabled
[    0.000125]   9 disabled
[    0.000126] MTRR map: 5 entries (4 fixed + 1 variable; max 24), built from 10 variable MTRRs
[    0.000128]   0: 0000000000000000-000000000009ffff write-back
[    0.000129]   1: 00000000000a0000-00000000000bffff uncachable
[    0.000130]   2: 00000000000c0000-00000000000dffff write-protect
[    0.000130]   3: 00000000000e0000-00000000000fffff uncachable
[    0.000131]   4: 0000000079000000-00000000ffffffff uncachable
```

It's showing us the state of the MTRRs on the boot CPU when [`mtrr_bp_init()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/mtrr/mtrr.c#L559) is called. Presumably as set by the BIOS.

It is clear that the call to [`mtrr_cleanup()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/mtrr/cleanup.c#L670) doesn't do anything, because setting `mtrr_debug` would make it print something if it did, as you can see from how [`Dprintk()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/mtrr/mtrr.h#L14) works. This means [`changed_by_mtrr_cleanup`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/mtrr/mtrr.c#L551) will be 0, ensuring that [`mtrr_state_warn()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/mtrr/generic.c#L730) will later be called by [`mtrr_init_finalize()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/mtrr/mtrr.c#L625), diagnosing any MTRRs changes made on boot.

The output starting with "`MTRR map:`" comes from [`mtrr_build_map()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/mtrr/generic.c#L351). It shows the result of converting the MTTRs into ranges, with overlapping entries of the same type coalesced. Note that variable MTRR 3 overlaps 0, 1 & 2.

The caller of [`mtrr_bp_init()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/mtrr/mtrr.c#L559) is [`cache_bp_init()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/cacheinfo.c#L757), which also calls - directly, on the boot CPU - [`cache_cpu_init()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/cacheinfo.c#L719), which calls [`mtrr_generic_set_state()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/mtrr/generic.c#L964). We can tell it does nothing (why would it, as it's setting the MTRRs based on the read MTRR state), because the later call to [`mtrr_state_warn()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/mtrr/generic.c#L730) doesn't print anything.

The lack of output from [`mtrr_state_warn()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/mtrr/generic.c#L730) also tells us that no changes were made to the MTRRs of the other processors. If there had been changes, then [`mtrr_generic_set_state()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/mtrr/generic.c#L964) - called from [`cache_cpu_init()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/cacheinfo.c#L719) from [`cache_rendezvous_handler()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/cacheinfo.c#L749) on *all* CPUs by [`cache_aps_init()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/cacheinfo.c#L807) from [`native_smp_cpus_done()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/smpboot.c#L1108) from [`smp_cpus_done()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/include/asm/smp.h#L66) from [`smp_init()`](https://elixir.bootlin.com/linux/v6.16.6/source/kernel/smp.c#L1001) - would have set [`smp_changes_mask`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/mtrr/generic.c#L87), causing [`mtrr_state_warn()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/mtrr/generic.c#L730) to print when called from [`mtrr_init_finalize()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/mtrr/mtrr.c#L625).

### Changes after boot

Are any other MTRR changes made after boot, before suspend? We can get a diagnostic of the variable MTRRs, which appear unchanged:
```
$ sudo cat /proc/mtrr
reg00: base=0x080000000 ( 2048MB), size= 2048MB, count=1: uncachable
reg01: base=0x07c000000 ( 1984MB), size=   64MB, count=1: uncachable
reg02: base=0x07a000000 ( 1952MB), size=   32MB, count=1: uncachable
reg03: base=0x079000000 ( 1936MB), size=   16MB, count=1: uncachable
```

This doesn't print the fixed MTRRs, but there doesn't seem to be a code path for changing them - only restoring them with [`set_fixed_ranged()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/mtrr/generic.c#L865), called from [`set_mtrr_state()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/mtrr/generic.c#L920) from [`mtrr_generic_set_state()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/mtrr/generic.c#L964).

### Changes after resume

What changes are made after resume?

What about [`cache_bp_restore()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/cacheinfo.c#L766) calling [`cache_cpu_init()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/cacheinfo.c#L719) on resume? Can we tell if it made changes? I don't think so, because `mtrr_state_warn()` is only called from [`mtrr_init_finalize()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/mtrr/mtrr.c#L625). We could change that.

Similarly, we can't tell about changes on the other CPUs. They would be made when [`arch_thaw_secondary_cpus_end()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/smpboot.c#L1089) calls [`cache_aps_init()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/cacheinfo.c#L807), in the same way that [`native_smp_cpus_done()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/smpboot.c#L1108) did on boot.

We may as well patch the code to show all changes made by [`set_mtrr_state()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/mtrr/generic.c#L920), called from [`mtrr_generic_set_state()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/mtrr/generic.c#L964), from [`cache_cpu_init()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/cacheinfo.c#L719).

Revert to 6.16.7:
```
$ git checkout v6.16.7
$ make mrproper
```
Add debug printing when changes are made, as well as a summary line of the change mask, so that we can easily see for each CPU whether changes were made.

changes from [commit](https://github.com/cgpe-a/linux/commit/884399682120f1864b7c183796991d29deddc3cc):
```
$ git diff
diff --git a/arch/x86/kernel/cpu/mtrr/generic.c b/arch/x86/kernel/cpu/mtrr/generic.c
index 8c18327eb10b..f0793919a708 100644
--- a/arch/x86/kernel/cpu/mtrr/generic.c
+++ b/arch/x86/kernel/cpu/mtrr/generic.c
@@ -764,7 +764,7 @@ void mtrr_wrmsr(unsigned msr, unsigned a, unsigned b)
  * @changed: pointer which indicates whether the MTRR needed to be changed
  * @msrwords: pointer to the MSR values which the MSR should have
  */
-static void set_fixed_range(int msr, bool *changed, unsigned int *msrwords)
+static void set_fixed_range(int block, int range, int msr, bool *changed, unsigned int *msrwords)
 {
 	unsigned lo, hi;
 
@@ -773,6 +773,13 @@ static void set_fixed_range(int msr, bool *changed, unsigned int *msrwords)
 	if (lo != msrwords[0] || hi != msrwords[1]) {
 		mtrr_wrmsr(msr, msrwords[0], msrwords[1]);
 		*changed = true;
+		if (mtrr_debug) {
+			int cpu = smp_processor_id();
+
+			pr_info("MTRR cpu %d fixed %d,%d: %08x%08x -> %08x%08x [bits %08x%08x]\n",
+				cpu, block, range, hi, lo, msrwords[1], msrwords[0],
+				hi ^ msrwords[1], lo ^ msrwords[0]);
+		}
 	}
 }
 
@@ -872,7 +879,8 @@ static int set_fixed_ranges(mtrr_type *frs)
 
 	while (fixed_range_blocks[++block].ranges) {
 		for (range = 0; range < fixed_range_blocks[block].ranges; range++)
-			set_fixed_range(fixed_range_blocks[block].base_msr + range,
+			set_fixed_range(block, range,
+					fixed_range_blocks[block].base_msr + range,
 					&changed, (unsigned int *)saved++);
 	}
 
@@ -886,6 +894,7 @@ static int set_fixed_ranges(mtrr_type *frs)
 static bool set_mtrr_var_ranges(unsigned int index, struct mtrr_var_range *vr)
 {
 	unsigned int lo, hi;
+	unsigned int blo, bhi;
 	bool changed = false;
 
 	rdmsr(MTRRphysBase_MSR(index), lo, hi);
@@ -895,6 +904,8 @@ static bool set_mtrr_var_ranges(unsigned int index, struct mtrr_var_range *vr)
 		mtrr_wrmsr(MTRRphysBase_MSR(index), vr->base_lo, vr->base_hi);
 		changed = true;
 	}
+	blo = lo;
+	bhi = hi;
 
 	rdmsr(MTRRphysMask_MSR(index), lo, hi);
 
@@ -903,6 +914,17 @@ static bool set_mtrr_var_ranges(unsigned int index, struct mtrr_var_range *vr)
 		mtrr_wrmsr(MTRRphysMask_MSR(index), vr->mask_lo, vr->mask_hi);
 		changed = true;
 	}
+
+	if (mtrr_debug && changed) {
+		int cpu = smp_processor_id();
+
+		pr_info("MTRR cpu %d variable %u %08x%08x/%08x%08x -> %08x%08x/%08x%08x [bits %08x%08x/%08x%08x]\n",
+			cpu, index,
+			bhi, blo, hi, lo,
+			vr->base_hi, vr->base_lo, vr->mask_hi, vr->mask_lo,
+			bhi ^ vr->base_hi, blo ^ vr->base_lo, hi ^ vr->mask_hi, lo ^ vr->mask_lo);
+	}
+
 	return changed;
 }
 
@@ -943,6 +965,15 @@ static unsigned long set_mtrr_state(void)
 		change_mask |= MTRR_CHANGE_MASK_DEFTYPE;
 	}
 
+	if (mtrr_debug) {
+		int cpu = smp_processor_id();
+
+		pr_info("MTRR cpu %d changes:%c%c%c\n", cpu,
+			change_mask & MTRR_CHANGE_MASK_VARIABLE ? 'V' : '_',
+			change_mask & MTRR_CHANGE_MASK_FIXED ? 'F' : '_',
+			change_mask & MTRR_CHANGE_MASK_DEFTYPE ? 'T' : '_');
+	}
+
 	return change_mask;
 }
 
@@ -953,12 +984,35 @@ void mtrr_disable(void)
 
 	/* Disable MTRRs, and set the default type to uncached */
 	mtrr_wrmsr(MSR_MTRRdefType, deftype_lo & MTRR_DEF_TYPE_DISABLE, deftype_hi);
+
+	if (mtrr_debug && !(deftype_lo & MTRR_DEF_TYPE_DISABLE)) {
+		int cpu = smp_processor_id();
+		u32 hi = deftype_hi;
+		u32 lo = deftype_lo & MTRR_DEF_TYPE_DISABLE;
+
+		pr_info("MTRR cpu %d disable: %08x%08x -> %08x%08x [bits %08x%08x]\n", cpu,
+			deftype_hi, deftype_lo, hi, lo,
+			deftype_hi ^ hi, deftype_lo ^ lo);
+	}
 }
 
 void mtrr_enable(void)
 {
+       u32 hi, lo;
+
+       if (mtrr_debug)
+               rdmsr(MSR_MTRRdefType, lo, hi);
+
        /* Intel (P6) standard MTRRs */
        mtrr_wrmsr(MSR_MTRRdefType, deftype_lo, deftype_hi);
+
+       if (mtrr_debug && (lo != deftype_lo || hi != deftype_hi)) {
+               int cpu = smp_processor_id();
+
+               pr_info("MTRR cpu %d enable: %08x%08x -> %08x%08x [bits %08x%08x]\n", cpu,
+                       hi, lo, deftype_hi, deftype_lo,
+                       hi ^ deftype_hi, lo ^ deftype_lo);
+       }
 }
 
 void mtrr_generic_set_state(void)
```
Build and run:
```
$ cp .config-6.16.7-localmodconfig .config
$ make -j$(nproc)
$ sudo make modules_install
$ sudo make install
$ sudo kernelstub -k /boot/vmlinuz-6.16.7-custom-dirty -i /boot/initrd.img-6.16.7-custom-dirty
$ reboot
...
$ uname -r
6.16.7-custom-dirty
```
As expected, there are no changes on boot (no-op writing what was read for CPU 0, then the no-op writes to all CPUs):
```
[    0.000235] MTRR cpu 0 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[    0.001757] MTRR cpu 0 changes:___
[    0.002502] MTRR cpu 0 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
...
[    0.185127] MTRR cpu 6 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[    0.186006] MTRR cpu 6 changes:___
[    0.186006] MTRR cpu 6 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[    0.186006] MTRR cpu 4 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[    0.186006] MTRR cpu 4 changes:___
[    0.186006] MTRR cpu 4 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[    0.186006] MTRR cpu 3 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[    0.186006] MTRR cpu 3 changes:___
[    0.186006] MTRR cpu 3 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[    0.186006] MTRR cpu 1 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[    0.186006] MTRR cpu 1 changes:___
[    0.186006] MTRR cpu 1 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[    0.186006] MTRR cpu 0 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[    0.186006] MTRR cpu 0 changes:___
[    0.186006] MTRR cpu 0 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[    0.186006] MTRR cpu 5 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[    0.186006] MTRR cpu 5 changes:___
[    0.186006] MTRR cpu 5 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[    0.186006] MTRR cpu 7 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[    0.186006] MTRR cpu 7 changes:___
[    0.186006] MTRR cpu 7 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[    0.186006] MTRR cpu 2 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[    0.186006] MTRR cpu 2 changes:___
[    0.186006] MTRR cpu 2 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
```
The enable/disable output is coming from the [`mtrr_disable()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/mtrr/generic.c#L949)/[`mtrr_enable()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/mtrr/generic.c#L958) calls made as the cache is disabled and re-enabled.

On resume, which took ~18s, we can see changes on CPUs 0,1,6 & 7 (on a subsequent resume it was 0,2,3 & 5, and on another reboot & resume it was 0,5,6 & 7, so it's not consistent):
```
[  231.833596] MTRR cpu 0 disable: 0000000000000c00 -> 0000000000000000 [bits 0000000000000c00]
[  231.836708] MTRR cpu 0 variable 0 00000000ff800005/0000007fff800800 -> 0000000080000000/0000007f80000800 [bits 000000007f800005/000000007f800000]
[  231.842166] MTRR cpu 0 variable 1 0000000000000006/0000007fc0000800 -> 000000007c000000/0000007ffc000800 [bits 000000007c000006/000000003c000000]
[  231.847381] MTRR cpu 0 variable 2 0000000040000006/0000007fe0000800 -> 000000007a000000/0000007ffe000800 [bits 000000003a000006/000000001e000000]
[  231.852625] MTRR cpu 0 variable 3 0000000060000006/0000007ff0000800 -> 0000000079000000/0000007fff000800 [bits 0000000019000006/000000000f000000]
[  231.857789] MTRR cpu 0 variable 4 0000000070000006/0000007ff8000800 -> 0000000000000000/0000000000000000 [bits 0000000070000006/0000007ff8000800]
[  231.862765] MTRR cpu 0 variable 5 0000000078000006/0000007fff000800 -> 0000000000000000/0000000000000000 [bits 0000000078000006/0000007fff000800]
[  231.867820] MTRR cpu 0 fixed 0,0: 0000000000000005 -> 0606060606060606 [bits 0606060606060603]
[  231.871151] MTRR cpu 0 fixed 1,0: 0000000000000000 -> 0606060606060606 [bits 0606060606060606]
[  231.874509] MTRR cpu 0 fixed 2,0: 0000000000000000 -> 0505050505050505 [bits 0505050505050505]
[  231.877842] MTRR cpu 0 fixed 2,1: 0000000000000000 -> 0505050505050505 [bits 0505050505050505]
[  231.881175] MTRR cpu 0 fixed 2,2: 0000000000000000 -> 0505050505050505 [bits 0505050505050505]
[  231.884517] MTRR cpu 0 fixed 2,3: 0000000000000000 -> 0505050505050505 [bits 0505050505050505]
[  231.887891] MTRR cpu 0 changes:VFT
[  231.889583] MTRR cpu 0 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[  231.892329] ACPI: PM: Low-level resume complete
[  231.892360] ACPI: EC: EC started
[  231.892362] ACPI: PM: Restoring platform NVS memory
[  231.892809] Enabling non-boot CPUs ...
[  231.892926] smpboot: Booting Node 0 Processor 1 APIC 0x2
[  232.010058] hrtimer: interrupt took 3589040 ns
[  232.578533] CPU1 is up
[  232.583613] smpboot: Booting Node 0 Processor 2 APIC 0x4
[  233.241925] CPU2 is up
[  233.247477] smpboot: Booting Node 0 Processor 3 APIC 0x6
[  235.249031] CPU3 is up
[  235.250156] smpboot: Booting Node 0 Processor 4 APIC 0x1
[  235.263574] CPU4 is up
[  235.263627] smpboot: Booting Node 0 Processor 5 APIC 0x3
[  238.799081] CPU5 is up
[  238.804890] smpboot: Booting Node 0 Processor 6 APIC 0x5
[  242.771566] CPU6 is up
[  242.822485] smpboot: Booting Node 0 Processor 7 APIC 0x7
[  248.146442] CPU7 is up
[  248.152352] MTRR cpu 4 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[  248.154653] MTRR cpu 4 changes:___
[  248.155805] MTRR cpu 4 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[  248.157807] MTRR cpu 0 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[  248.160108] MTRR cpu 0 changes:___
[  248.161255] MTRR cpu 0 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[  248.163279] MTRR cpu 6 disable: 0000000000000c00 -> 0000000000000000 [bits 0000000000000c00]
[  248.165276] MTRR cpu 6 variable 0 00000000ff800005/0000007fff800800 -> 0000000080000000/0000007f80000800 [bits 000000007f800005/000000007f800000]
[  248.168835] MTRR cpu 6 variable 1 0000000000000006/0000007fc0000800 -> 000000007c000000/0000007ffc000800 [bits 000000007c000006/000000003c000000]
[  248.172320] MTRR cpu 6 variable 2 0000000040000006/0000007fe0000800 -> 000000007a000000/0000007ffe000800 [bits 000000003a000006/000000001e000000]
[  248.175869] MTRR cpu 6 variable 3 0000000060000006/0000007ff0000800 -> 0000000079000000/0000007fff000800 [bits 0000000019000006/000000000f000000]
[  248.179415] MTRR cpu 6 variable 4 0000000070000006/0000007ff8000800 -> 0000000000000000/0000000000000000 [bits 0000000070000006/0000007ff8000800]
[  248.182825] MTRR cpu 6 variable 5 0000000078000006/0000007fff000800 -> 0000000000000000/0000000000000000 [bits 0000000078000006/0000007fff000800]
[  248.186242] MTRR cpu 6 fixed 0,0: 0000000000000000 -> 0606060606060606 [bits 0606060606060606]
[  248.188464] MTRR cpu 6 fixed 1,0: 0000000000000000 -> 0606060606060606 [bits 0606060606060606]
[  248.190705] MTRR cpu 6 fixed 2,0: 0000000000000000 -> 0505050505050505 [bits 0505050505050505]
[  248.192926] MTRR cpu 6 fixed 2,1: 0000000000000000 -> 0505050505050505 [bits 0505050505050505]
[  248.195150] MTRR cpu 6 fixed 2,2: 0000000000000000 -> 0505050505050505 [bits 0505050505050505]
[  248.197371] MTRR cpu 6 fixed 2,3: 0000000000000000 -> 0505050505050505 [bits 0505050505050505]
[  248.199616] MTRR cpu 6 changes:VFT
[  248.200731] MTRR cpu 6 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[  248.202678] MTRR cpu 1 disable: 0000000000000c00 -> 0000000000000000 [bits 0000000000000c00]
[  248.204630] MTRR cpu 1 variable 0 00000000ff800005/0000007fff800800 -> 0000000080000000/0000007f80000800 [bits 000000007f800005/000000007f800000]
[  248.208107] MTRR cpu 1 variable 1 0000000000000006/0000007fc0000800 -> 000000007c000000/0000007ffc000800 [bits 000000007c000006/000000003c000000]
[  248.211522] MTRR cpu 1 variable 2 0000000040000006/0000007fe0000800 -> 000000007a000000/0000007ffe000800 [bits 000000003a000006/000000001e000000]
[  248.215005] MTRR cpu 1 variable 3 0000000060000006/0000007ff0000800 -> 0000000079000000/0000007fff000800 [bits 0000000019000006/000000000f000000]
[  248.218467] MTRR cpu 1 variable 4 0000000070000006/0000007ff8000800 -> 0000000000000000/0000000000000000 [bits 0000000070000006/0000007ff8000800]
[  248.221806] MTRR cpu 1 variable 5 0000000078000006/0000007fff000800 -> 0000000000000000/0000000000000000 [bits 0000000078000006/0000007fff000800]
[  248.225220] MTRR cpu 1 fixed 0,0: 0000000000000000 -> 0606060606060606 [bits 0606060606060606]
[  248.227473] MTRR cpu 1 fixed 1,0: 0000000000000000 -> 0606060606060606 [bits 0606060606060606]
[  248.229723] MTRR cpu 1 fixed 2,0: 0000000000000000 -> 0505050505050505 [bits 0505050505050505]
[  248.231977] MTRR cpu 1 fixed 2,1: 0000000000000000 -> 0505050505050505 [bits 0505050505050505]
[  248.234202] MTRR cpu 1 fixed 2,2: 0000000000000000 -> 0505050505050505 [bits 0505050505050505]
[  248.236436] MTRR cpu 1 fixed 2,3: 0000000000000000 -> 0505050505050505 [bits 0505050505050505]
[  248.238687] MTRR cpu 1 changes:VFT
[  248.239807] MTRR cpu 1 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[  248.241762] MTRR cpu 7 disable: 0000000000000c00 -> 0000000000000000 [bits 0000000000000c00]
[  248.243773] MTRR cpu 7 variable 0 00000000ff800005/0000007fff800800 -> 0000000080000000/0000007f80000800 [bits 000000007f800005/000000007f800000]
[  248.247336] MTRR cpu 7 variable 1 0000000000000006/0000007fc0000800 -> 000000007c000000/0000007ffc000800 [bits 000000007c000006/000000003c000000]
[  248.250846] MTRR cpu 7 variable 2 0000000040000006/0000007fe0000800 -> 000000007a000000/0000007ffe000800 [bits 000000003a000006/000000001e000000]
[  248.254413] MTRR cpu 7 variable 3 0000000060000006/0000007ff0000800 -> 0000000079000000/0000007fff000800 [bits 0000000019000006/000000000f000000]
[  248.257975] MTRR cpu 7 variable 4 0000000070000006/0000007ff8000800 -> 0000000000000000/0000000000000000 [bits 0000000070000006/0000007ff8000800]
[  248.261403] MTRR cpu 7 variable 5 0000000078000006/0000007fff000800 -> 0000000000000000/0000000000000000 [bits 0000000078000006/0000007fff000800]
[  248.264898] MTRR cpu 7 fixed 0,0: 0000000000000000 -> 0606060606060606 [bits 0606060606060606]
[  248.267197] MTRR cpu 7 fixed 1,0: 0000000000000000 -> 0606060606060606 [bits 0606060606060606]
[  248.269508] MTRR cpu 7 fixed 2,0: 0000000000000000 -> 0505050505050505 [bits 0505050505050505]
[  248.271802] MTRR cpu 7 fixed 2,1: 0000000000000000 -> 0505050505050505 [bits 0505050505050505]
[  248.274104] MTRR cpu 7 fixed 2,2: 0000000000000000 -> 0505050505050505 [bits 0505050505050505]
[  248.276403] MTRR cpu 7 fixed 2,3: 0000000000000000 -> 0505050505050505 [bits 0505050505050505]
[  248.278724] MTRR cpu 7 changes:VFT
[  248.279859] MTRR cpu 7 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[  248.281860] MTRR cpu 5 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[  248.284307] MTRR cpu 5 changes:___
[  248.285521] MTRR cpu 5 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[  248.287629] MTRR cpu 3 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[  248.290044] MTRR cpu 3 changes:___
[  248.291245] MTRR cpu 3 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[  248.293331] MTRR cpu 2 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[  248.295739] MTRR cpu 2 changes:___
[  248.296941] MTRR cpu 2 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[  248.298940] ACPI: PM: Waking up from system sleep state S3
```
Note that CPU 0 is considered twice, and it appears that the changes made the first time slows down the resume.

Another anomaly is that the disable/enable of MTTRs on CPU 0 resume does not change `MSR_MTRRdefType` from 0xc06 -> 0x000 -> 0xc06 as usual, but rather 0xc00 -> 0x000 -> 0xc06. That is, between suspend & resume the CPU 0 `MSR_MTRRdefType` changed from 0xc06 to 0xc00. So from ["writeback" to "uncachable"](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/mtrr/if.c#L19).

### Split of resume changes

Why is CPU 0 considered twice on resume? This results in an extended period when its MTRRs have been restored, but those on the other CPUs have not - until after all CPUs are brought up and the MTRRs restored.

The first CPU 0 MTRR restore comes from:

[`do_suspend_lowlevel`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/acpi/wakeup_64.S#L49), [`restore_processor_state()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/power/cpu.c#L297), [`restore_processor_state_()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/power/cpu.c#L197), [`cache_bp_restore()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/cacheinfo.c#L766), [`cache_cpu_init()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/cacheinfo.c#L719), [`mtrr_generic_set_state()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/mtrr/generic.c#L964).

The all-CPUs MTRR restore comes from:

[`suspend_enter()`](https://elixir.bootlin.com/linux/v6.16.6/source/kernel/power/suspend.c#L413), [`pm_sleep_enable_secondary_cpus()`](https://elixir.bootlin.com/linux/v6.16.6/source/kernel/power/power.h#L347), [`suspend_enable_secondary_cpus()`](https://elixir.bootlin.com/linux/v6.16.6/source/include/linux/cpu.h#L157), [`thaw_secondary_cpus()`](https://elixir.bootlin.com/linux/v6.16.6/source/kernel/cpu.c#L1965), [`arch_thaw_secondary_cpus_end()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/smpboot.c#L1089), [`cache_aps_init()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/cacheinfo.c#L807), [`stop_machine()`](https://elixir.bootlin.com/linux/v6.16.6/source/kernel/stop_machine.c#L623),... [`cache_rendezvous_handler()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/cacheinfo.c#L749), [`cache_cpu_init()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/cacheinfo.c#L719), [`mtrr_generic_set_state()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/mtrr/generic.c#L964).

### Delay of resume changes on secondary CPUs

The confusing [`cache_aps_delayed_init`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/cacheinfo.c#L737) variable is set to `true` on resume by [`arch_thaw_secondary_cpus_begin()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/smpboot.c#L1084), [`set_cache_aps_delayed_init()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/cacheinfo.c#L739). This supresses the *other* call to stop_machine/cache_rendezvous_handler - from [`cache_ap_online()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/cacheinfo.c#L772), registered as a callback with CPU hotplug state [`CPUHP_AP_CACHECTRL_STARTING`](https://elixir.bootlin.com/linux/v6.16.6/source/include/linux/cpuhotplug.h#L135). The state callback is invoked by [`secondary_startup_64`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/head_64.S#L147), [`initial_code`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/smpboot.c#L856) == [`start_secondary()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/smpboot.c#L232), [`ap_starting()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/smpboot.c#L173), [`notify_cpu_starting()`](https://elixir.bootlin.com/linux/v6.16.6/source/kernel/cpu.c#L1589), [`cpuhp_invoke_callback_range_nofail()`](https://elixir.bootlin.com/linux/v6.16.6/source/kernel/cpu.c#L994), [`__cpuhp_invoke_callback_range()`](https://elixir.bootlin.com/linux/v6.16.6/source/kernel/cpu.c#L955), [`cpuhp_invoke_callback()`](https://elixir.bootlin.com/linux/v6.16.6/source/kernel/cpu.c#L169).

So, restoring the MTRRs on the secondary cpus is delayed from the bringup implemented by the hotplug state machine until after all the CPUs are thawed - in [`arch_thaw_secondary_cpus_end()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/smpboot.c#L1089).

Why? the `cache_aps_delayed_init` variable was originally added as `mtrr_aps_delayed_init` by commit [d0af9eed](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=d0af9eed5aa91b6b7b5049cae69e5ea956fd85c3) "x86, pat/mtrr: Rendezvous all the cpus for MTRR/PAT init". This commit co-ordinates MTRR updates across CPUs as required by the CPU documentation. Why wait until all CPUs are up? The commit message doesn't say, but I assume that the co-ordination has an overhead, and so it makes sense to wait until all CPUs are up to minimise the overhead by only doing it once for all CPUs.

The code has changed since that commit. The changes in commit [30f89e52](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=30f89e524becdbaa483b34902b079c9d4dfaa4a3) "x86/cacheinfo: Switch cache_ap_init() to hotplug callback" move the cache initialisation into a CPU hotplug callback, reusing `cache_aps_delayed_init` to make the callback a no-op in the resume case, preserving the cache init delay until [`arch_thaw_secondary_cpus_end()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/smpboot.c#L1089).

Unfortunately for us, the CPUs seem to run slowly with inconsistent MTRRs, and so much so that the delayed cache init causes the whole bring up to take tens of seconds.

### Reducing MTRR restore delay

Can we reduce the delay by allowing [`cache_ap_online()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/cacheinfo.c#L772) to call [`stop_machine_from_inactive_cpu`](https://elixir.bootlin.com/linux/v6.16.6/source/kernel/stop_machine.c#L678)`(`[`cache_rendezvous_handler`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/cacheinfo.c#L749)`),...)`? This would minimise the time the secondary CPUs are running with inconsistent MTRRs, but increase the number of time all CPUs have to call the rendevzous handler, and therefore all the code in [`cache_cpu_init()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/cacheinfo.c#L719).

We can try it with this temporary change:
```
$ git diff arch/x86/kernel/smpboot.c
diff --git a/arch/x86/kernel/smpboot.c b/arch/x86/kernel/smpboot.c
index 58ede3fa6a75..cb98426f3212 100644
--- a/arch/x86/kernel/smpboot.c
+++ b/arch/x86/kernel/smpboot.c
@@ -1083,11 +1083,12 @@ void __init native_smp_prepare_cpus(unsigned int max_cpus)

 void arch_thaw_secondary_cpus_begin(void)
 {
-       set_cache_aps_delayed_init(true);
+       //set_cache_aps_delayed_init(true);
 }

 void arch_thaw_secondary_cpus_end(void)
 {
+       set_cache_aps_delayed_init(true);
        cache_aps_init();
 }
```

This does seem to solve the problem. The resume (lid open to lock screen) takes ~3s, which is as good as it's ever been.

As before, on resume the CPU 0 MTRRs are restored by [`cache_bp_restore()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/cacheinfo.c#L766):
```
[   63.743168] MTRR cpu 0 disable: 0000000000000c00 -> 0000000000000000 [bits 0000000000000c00]
[   63.746259] MTRR cpu 0 variable 0 00000000ff800005/0000007fff800800 -> 0000000080000000/0000007f80000800 [bits 000000007f800005/000000007f800000]
[   63.751740] MTRR cpu 0 variable 1 0000000000000006/0000007fc0000800 -> 000000007c000000/0000007ffc000800 [bits 000000007c000006/000000003c000000]
[   63.756946] MTRR cpu 0 variable 2 0000000040000006/0000007fe0000800 -> 000000007a000000/0000007ffe000800 [bits 000000003a000006/000000001e000000]
[   63.762173] MTRR cpu 0 variable 3 0000000060000006/0000007ff0000800 -> 0000000079000000/0000007fff000800 [bits 0000000019000006/000000000f000000]
[   63.767359] MTRR cpu 0 variable 4 0000000070000006/0000007ff8000800 -> 0000000000000000/0000000000000000 [bits 0000000070000006/0000007ff8000800]
[   63.772345] MTRR cpu 0 variable 5 0000000078000006/0000007fff000800 -> 0000000000000000/0000000000000000 [bits 0000000078000006/0000007fff000800]
[   63.777453] MTRR cpu 0 fixed 0,0: 0000000000000005 -> 0606060606060606 [bits 0606060606060603]
[   63.780793] MTRR cpu 0 fixed 1,0: 0000000000000000 -> 0606060606060606 [bits 0606060606060606]
[   63.784166] MTRR cpu 0 fixed 2,0: 0000000000000000 -> 0505050505050505 [bits 0505050505050505]
[   63.787507] MTRR cpu 0 fixed 2,1: 0000000000000000 -> 0505050505050505 [bits 0505050505050505]
[   63.790850] MTRR cpu 0 fixed 2,2: 0000000000000000 -> 0505050505050505 [bits 0505050505050505]
[   63.794207] MTRR cpu 0 fixed 2,3: 0000000000000000 -> 0505050505050505 [bits 0505050505050505]
[   63.797590] MTRR cpu 0 changes:VFT
[   63.799257] MTRR cpu 0 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   63.801998] ACPI: PM: Low-level resume complete
[   63.802030] ACPI: EC: EC started
[   63.802032] ACPI: PM: Restoring platform NVS memory
```
While the secondary CPUs are brought up, we can now see the MTRRs being checked and fixed by the [`cache_ap_online()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/cacheinfo.c#L772) CPU hotplug calls:
```
[   63.802494] Enabling non-boot CPUs ...
[   63.802609] smpboot: Booting Node 0 Processor 1 APIC 0x2
[   63.816594] MTRR cpu 1 disable: 0000000000000c00 -> 0000000000000000 [bits 0000000000000c00]
[   63.819331] MTRR cpu 1 variable 0 00000000ff800005/0000007fff800800 -> 0000000080000000/0000007f80000800 [bits 000000007f800005/000000007f800000]
[   63.824179] MTRR cpu 1 variable 1 0000000000000006/0000007fc0000800 -> 000000007c000000/0000007ffc000800 [bits 000000007c000006/000000003c000000]
[   63.828942] MTRR cpu 1 variable 2 0000000040000006/0000007fe0000800 -> 000000007a000000/0000007ffe000800 [bits 000000003a000006/000000001e000000]
[   63.833831] MTRR cpu 1 variable 3 0000000060000006/0000007ff0000800 -> 0000000079000000/0000007fff000800 [bits 0000000019000006/000000000f000000]
[   63.838656] MTRR cpu 1 variable 4 0000000070000006/0000007ff8000800 -> 0000000000000000/0000000000000000 [bits 0000000070000006/0000007ff8000800]
[   63.843307] MTRR cpu 1 variable 5 0000000078000006/0000007fff000800 -> 0000000000000000/0000000000000000 [bits 0000000078000006/0000007fff000800]
[   63.848041] MTRR cpu 1 fixed 0,0: 0000000000000000 -> 0606060606060606 [bits 0606060606060606]
[   63.851159] MTRR cpu 1 fixed 1,0: 0000000000000000 -> 0606060606060606 [bits 0606060606060606]
[   63.854284] MTRR cpu 1 fixed 2,0: 0000000000000000 -> 0505050505050505 [bits 0505050505050505]
[   63.857400] MTRR cpu 1 fixed 2,1: 0000000000000000 -> 0505050505050505 [bits 0505050505050505]
[   63.860513] MTRR cpu 1 fixed 2,2: 0000000000000000 -> 0505050505050505 [bits 0505050505050505]
[   63.863619] MTRR cpu 1 fixed 2,3: 0000000000000000 -> 0505050505050505 [bits 0505050505050505]
[   63.866773] MTRR cpu 1 changes:VFT
[   63.868327] MTRR cpu 1 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   63.871679] CPU1 is up
[   63.871774] smpboot: Booting Node 0 Processor 2 APIC 0x4
[   63.884824] MTRR cpu 2 disable: 0000000000000c00 -> 0000000000000000 [bits 0000000000000c00]
[   63.887182] MTRR cpu 2 variable 0 00000000ff800005/0000007fff800800 -> 0000000080000000/0000007f80000800 [bits 000000007f800005/000000007f800000]
[   63.891369] MTRR cpu 2 variable 1 0000000000000006/0000007fc0000800 -> 000000007c000000/0000007ffc000800 [bits 000000007c000006/000000003c000000]
[   63.895476] MTRR cpu 2 variable 2 0000000040000006/0000007fe0000800 -> 000000007a000000/0000007ffe000800 [bits 000000003a000006/000000001e000000]
[   63.899672] MTRR cpu 2 variable 3 0000000060000006/0000007ff0000800 -> 0000000079000000/0000007fff000800 [bits 0000000019000006/000000000f000000]
[   63.903846] MTRR cpu 2 variable 4 0000000070000006/0000007ff8000800 -> 0000000000000000/0000000000000000 [bits 0000000070000006/0000007ff8000800]
[   63.907890] MTRR cpu 2 variable 5 0000000078000006/0000007fff000800 -> 0000000000000000/0000000000000000 [bits 0000000078000006/0000007fff000800]
[   63.912012] MTRR cpu 2 fixed 0,0: 0000000000000000 -> 0606060606060606 [bits 0606060606060606]
[   63.914723] MTRR cpu 2 fixed 1,0: 0000000000000000 -> 0606060606060606 [bits 0606060606060606]
[   63.917440] MTRR cpu 2 fixed 2,0: 0000000000000000 -> 0505050505050505 [bits 0505050505050505]
[   63.920145] MTRR cpu 2 fixed 2,1: 0000000000000000 -> 0505050505050505 [bits 0505050505050505]
[   63.922843] MTRR cpu 2 fixed 2,2: 0000000000000000 -> 0505050505050505 [bits 0505050505050505]
[   63.925539] MTRR cpu 2 fixed 2,3: 0000000000000000 -> 0505050505050505 [bits 0505050505050505]
[   63.928275] MTRR cpu 2 changes:VFT
[   63.929622] MTRR cpu 2 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   63.932558] CPU2 is up
[   63.932658] smpboot: Booting Node 0 Processor 3 APIC 0x6
[   63.944895] MTRR cpu 3 disable: 0000000000000c00 -> 0000000000000000 [bits 0000000000000c00]
[   63.946872] MTRR cpu 3 variable 0 00000000ff800005/0000007fff800800 -> 0000000080000000/0000007f80000800 [bits 000000007f800005/000000007f800000]
[   63.950386] MTRR cpu 3 variable 1 0000000000000006/0000007fc0000800 -> 000000007c000000/0000007ffc000800 [bits 000000007c000006/000000003c000000]
[   63.953832] MTRR cpu 3 variable 2 0000000040000006/0000007fe0000800 -> 000000007a000000/0000007ffe000800 [bits 000000003a000006/000000001e000000]
[   63.957353] MTRR cpu 3 variable 3 0000000060000006/0000007ff0000800 -> 0000000079000000/0000007fff000800 [bits 0000000019000006/000000000f000000]
[   63.960864] MTRR cpu 3 variable 4 0000000070000006/0000007ff8000800 -> 0000000000000000/0000000000000000 [bits 0000000070000006/0000007ff8000800]
[   63.964241] MTRR cpu 3 variable 5 0000000078000006/0000007fff000800 -> 0000000000000000/0000000000000000 [bits 0000000078000006/0000007fff000800]
[   63.967685] MTRR cpu 3 fixed 0,0: 0000000000000000 -> 0606060606060606 [bits 0606060606060606]
[   63.969956] MTRR cpu 3 fixed 1,0: 0000000000000000 -> 0606060606060606 [bits 0606060606060606]
[   63.972237] MTRR cpu 3 fixed 2,0: 0000000000000000 -> 0505050505050505 [bits 0505050505050505]
[   63.974506] MTRR cpu 3 fixed 2,1: 0000000000000000 -> 0505050505050505 [bits 0505050505050505]
[   63.976774] MTRR cpu 3 fixed 2,2: 0000000000000000 -> 0505050505050505 [bits 0505050505050505]
[   63.979060] MTRR cpu 3 fixed 2,3: 0000000000000000 -> 0505050505050505 [bits 0505050505050505]
[   63.981352] MTRR cpu 3 changes:VFT
[   63.982478] MTRR cpu 3 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   63.984977] CPU3 is up
[   63.985039] smpboot: Booting Node 0 Processor 4 APIC 0x1
[   63.991806] MTRR cpu 4 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[   63.994226] MTRR cpu 4 changes:___
[   63.995415] MTRR cpu 4 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   63.997985] CPU4 is up
[   63.998042] smpboot: Booting Node 0 Processor 5 APIC 0x3
[   64.004802] MTRR cpu 5 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[   64.007245] MTRR cpu 5 changes:___
[   64.008463] MTRR cpu 5 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   64.010992] CPU5 is up
[   64.011043] smpboot: Booting Node 0 Processor 6 APIC 0x5
[   64.017804] MTRR cpu 6 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[   64.020268] MTRR cpu 6 changes:___
[   64.021489] MTRR cpu 6 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   64.024057] CPU6 is up
[   64.024110] smpboot: Booting Node 0 Processor 7 APIC 0x7
[   64.030804] MTRR cpu 7 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[   64.033207] MTRR cpu 7 changes:___
[   64.034393] MTRR cpu 7 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   64.036912] CPU7 is up
```
In this case CPUs 1,2 & 3 needed MTRR restoration (as well as CPU 0).

The subsequent MTRR check from [`arch_thaw_secondary_cpus_end()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/smpboot.c#L1089), [`cache_aps_init()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/cacheinfo.c#L807) no longer makes any changes:
```
[   64.037037] MTRR cpu 4 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[   64.039333] MTRR cpu 4 changes:___
[   64.040470] MTRR cpu 4 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   64.042459] MTRR cpu 3 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[   64.044764] MTRR cpu 3 changes:___
[   64.045899] MTRR cpu 3 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   64.047893] MTRR cpu 5 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[   64.050180] MTRR cpu 5 changes:___
[   64.051305] MTRR cpu 5 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   64.053291] MTRR cpu 7 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[   64.055700] MTRR cpu 7 changes:___
[   64.056891] MTRR cpu 7 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   64.058959] MTRR cpu 2 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[   64.061185] MTRR cpu 2 changes:___
[   64.062287] MTRR cpu 2 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   64.064235] MTRR cpu 1 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[   64.066671] MTRR cpu 1 changes:___
[   64.067882] MTRR cpu 1 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   64.069990] MTRR cpu 6 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[   64.072525] MTRR cpu 6 changes:___
[   64.073761] MTRR cpu 6 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   64.075908] MTRR cpu 0 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[   64.078298] MTRR cpu 0 changes:___
[   64.079484] MTRR cpu 0 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   64.081462] ACPI: PM: Waking up from system sleep state S3
```

### Correctness of change

There are fewer MTRR checks than I was expecting. As each additional CPU is brought up, why is there not additional diagnostic output showing cache-disable/no-mtrr-changes/cache-enable for the CPUs that are already up?

That is, why is the [`cache_ap_online()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/cacheinfo.c#L772) call of [`stop_machine_from_inactive_cpu`](https://elixir.bootlin.com/linux/v6.16.6/source/kernel/stop_machine.c#L678)`(`[`cache_rendezvous_handler`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/cacheinfo.c#L749)`),...)` not calling [`cache_cpu_init()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/cacheinfo.c#L719) on all CPUs brought up so far?

The call to `cache_cpu_init()` in `cache_rendezvous_handler()` is guarded:
```
if (get_cache_aps_delayed_init() || !cpu_online(smp_processor_id()))
		cache_cpu_init();
```
We have disabled `cache_aps_delayed_init`, so we are relying on the current CPU not being online. The CPUs already brought up must already be regarded as online, making their rendezvous calls a no-op.

I don't know if this is okay. Maybe the required synchronisation is achieved by the `stop_machine` mechanism, and doesn't require the cache to be disabled & enabled on the other CPUs when the current CPU has its MTRRs set - as happens for [`arch_thaw_secondary_cpus_end()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/smpboot.c#L1089), [`cache_aps_init()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/cacheinfo.c#L807).

### Conditional change

Given my doubts about the correctness, and to preserve the existing behaviour on other machines, can we make this change conditional - so that it only happens when the MTRRs on CPU 0 have been changed on resume?

Reverting the temporary change to `smpboot.c`, and applying the following changes instead should both make the new behaviour conditional, and also avoid the change in the value of `cache_aps_delayed_init` that caused the change to the rendezvous behaviour.

changes from [commit](https://github.com/cgpe-a/linux/commit/d161fde1e9712eb2de01959c04db419190522372):
```
diff --git a/arch/x86/include/asm/mtrr.h b/arch/x86/include/asm/mtrr.h
index c69e269937c5..7a592e931530 100644
--- a/arch/x86/include/asm/mtrr.h
+++ b/arch/x86/include/asm/mtrr.h
@@ -73,7 +73,7 @@ extern int mtrr_trim_uncached_memory(unsigned long end_pfn);
 extern int amd_special_default_mtrr(void);
 void mtrr_disable(void);
 void mtrr_enable(void);
-void mtrr_generic_set_state(void);
+bool mtrr_generic_set_state(void);
 #  else
 static inline void guest_force_mtrr_state(struct mtrr_var_range *var,
 					  unsigned int num_var,
@@ -118,7 +118,10 @@ static inline int mtrr_trim_uncached_memory(unsigned long end_pfn)
 #define mtrr_bp_init() do {} while (0)
 #define mtrr_disable() do {} while (0)
 #define mtrr_enable() do {} while (0)
-#define mtrr_generic_set_state() do {} while (0)
+static inline bool mtrr_generic_set_state(void)
+{
+	return false;
+}
 #  endif
 
 #ifdef CONFIG_COMPAT
diff --git a/arch/x86/kernel/cpu/cacheinfo.c b/arch/x86/kernel/cpu/cacheinfo.c
index adfa7e8bb865..2508ecefcc6a 100644
--- a/arch/x86/kernel/cpu/cacheinfo.c
+++ b/arch/x86/kernel/cpu/cacheinfo.c
@@ -716,15 +716,16 @@ void cache_enable(void) __releases(cache_disable_lock)
 	raw_spin_unlock(&cache_disable_lock);
 }
 
-static void cache_cpu_init(void)
+static bool cache_cpu_init(void)
 {
 	unsigned long flags;
+	bool changed = false;
 
 	local_irq_save(flags);
 
 	if (memory_caching_control & CACHE_MTRR) {
 		cache_disable();
-		mtrr_generic_set_state();
+		changed = mtrr_generic_set_state();
 		cache_enable();
 	}
 
@@ -732,6 +733,8 @@ static void cache_cpu_init(void)
 		pat_cpu_init();
 
 	local_irq_restore(flags);
+
+	return changed;
 }
 
 static bool cache_aps_delayed_init = true;
@@ -749,7 +752,7 @@ bool get_cache_aps_delayed_init(void)
 static int cache_rendezvous_handler(void *unused)
 {
 	if (get_cache_aps_delayed_init() || !cpu_online(smp_processor_id()))
-		cache_cpu_init();
+		(void)cache_cpu_init();
 
 	return 0;
 }
@@ -760,20 +763,31 @@ void __init cache_bp_init(void)
 	pat_bp_init();
 
 	if (memory_caching_control)
-		cache_cpu_init();
+		(void)cache_cpu_init();
 }
 
+static bool cache_bp_changed_on_restore;
+
 void cache_bp_restore(void)
 {
 	if (memory_caching_control)
-		cache_cpu_init();
+		cache_bp_changed_on_restore = cache_cpu_init();
 }
 
 static int cache_ap_online(unsigned int cpu)
 {
 	cpumask_set_cpu(cpu, cpu_cacheinfo_mask);
 
-	if (!memory_caching_control || get_cache_aps_delayed_init())
+	if (!memory_caching_control)
+		return 0;
+
+	/*
+	 * Normally we delay MTRR (and PAT) init until cache_aps_init(), but if
+	 * the MTRRs had to be restored on the boot processor on resume, then
+	 * delaying any required MTTR restore on the APs can lead to very slow
+	 * CPU execution during the period when the MTRRs are inconsistent.
+	 */
+	if (get_cache_aps_delayed_init() && !cache_bp_changed_on_restore)
 		return 0;
 
 	/*
@@ -811,6 +825,7 @@ void cache_aps_init(void)
 
 	stop_machine(cache_rendezvous_handler, NULL, cpu_online_mask);
 	set_cache_aps_delayed_init(false);
+	cache_bp_changed_on_restore = false;
 }
 
 static int __init cache_ap_register(void)
diff --git a/arch/x86/kernel/cpu/mtrr/generic.c b/arch/x86/kernel/cpu/mtrr/generic.c
index e67b96a053f7..dc6231df272f 100644
--- a/arch/x86/kernel/cpu/mtrr/generic.c
+++ b/arch/x86/kernel/cpu/mtrr/generic.c
@@ -1015,12 +1015,14 @@ void mtrr_enable(void)
 	}
 }
 
-void mtrr_generic_set_state(void)
+bool mtrr_generic_set_state(void)
 {
 	unsigned long mask, count;
+	bool changed;
 
 	/* Actually set the state */
 	mask = set_mtrr_state();
+	changed = mask != 0;
 
 	/* Use the atomic bitops to update the global mask */
 	for (count = 0; count < sizeof(mask) * 8; ++count) {
@@ -1028,6 +1030,8 @@ void mtrr_generic_set_state(void)
 			set_bit(count, &smp_changes_mask);
 		mask >>= 1;
 	}
+
+	return changed;
 }
 
 /**
```

This also works, and has the extra MTRR checks on the processors already brought up. The whole process seems to take about 0.5s:
```
[   55.624629] MTRR cpu 0 disable: 0000000000000c00 -> 0000000000000000 [bits 0000000000000c00]
[   55.627729] MTRR cpu 0 variable 0 00000000ff800005/0000007fff800800 -> 0000000080000000/0000007f80000800 [bits 000000007f800005/000000007f800000]
[   55.633188] MTRR cpu 0 variable 1 0000000000000006/0000007fc0000800 -> 000000007c000000/0000007ffc000800 [bits 000000007c000006/000000003c000000]
[   55.638408] MTRR cpu 0 variable 2 0000000040000006/0000007fe0000800 -> 000000007a000000/0000007ffe000800 [bits 000000003a000006/000000001e000000]
[   55.643621] MTRR cpu 0 variable 3 0000000060000006/0000007ff0000800 -> 0000000079000000/0000007fff000800 [bits 0000000019000006/000000000f000000]
[   55.648800] MTRR cpu 0 variable 4 0000000070000006/0000007ff8000800 -> 0000000000000000/0000000000000000 [bits 0000000070000006/0000007ff8000800]
[   55.653763] MTRR cpu 0 variable 5 0000000078000006/0000007fff000800 -> 0000000000000000/0000000000000000 [bits 0000000078000006/0000007fff000800]
[   55.658814] MTRR cpu 0 fixed 0,0: 0000000000000005 -> 0606060606060606 [bits 0606060606060603]
[   55.662386] MTRR cpu 0 fixed 1,0: 0000000000000000 -> 0606060606060606 [bits 0606060606060606]
[   55.665986] MTRR cpu 0 fixed 2,0: 0000000000000000 -> 0505050505050505 [bits 0505050505050505]
[   55.669563] MTRR cpu 0 fixed 2,1: 0000000000000000 -> 0505050505050505 [bits 0505050505050505]
[   55.673130] MTRR cpu 0 fixed 2,2: 0000000000000000 -> 0505050505050505 [bits 0505050505050505]
[   55.676711] MTRR cpu 0 fixed 2,3: 0000000000000000 -> 0505050505050505 [bits 0505050505050505]
[   55.680340] MTRR cpu 0 changes:VFT
[   55.682008] MTRR cpu 0 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   55.684722] ACPI: PM: Low-level resume complete
[   55.684754] ACPI: EC: EC started
[   55.684755] ACPI: PM: Restoring platform NVS memory
[   55.685258] Enabling non-boot CPUs ...
[   55.685367] smpboot: Booting Node 0 Processor 1 APIC 0x2
[   55.699310] MTRR cpu 0 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[   55.702416] MTRR cpu 0 changes:___
[   55.703953] MTRR cpu 0 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   55.706643] MTRR cpu 1 disable: 0000000000000c00 -> 0000000000000000 [bits 0000000000000c00]
[   55.709372] MTRR cpu 1 variable 0 00000000ff800005/0000007fff800800 -> 0000000080000000/0000007f80000800 [bits 000000007f800005/000000007f800000]
[   55.714230] MTRR cpu 1 variable 1 0000000000000006/0000007fc0000800 -> 000000007c000000/0000007ffc000800 [bits 000000007c000006/000000003c000000]
[   55.718972] MTRR cpu 1 variable 2 0000000040000006/0000007fe0000800 -> 000000007a000000/0000007ffe000800 [bits 000000003a000006/000000001e000000]
[   55.723803] MTRR cpu 1 variable 3 0000000060000006/0000007ff0000800 -> 0000000079000000/0000007fff000800 [bits 0000000019000006/000000000f000000]
[   55.728638] MTRR cpu 1 variable 4 0000000070000006/0000007ff8000800 -> 0000000000000000/0000000000000000 [bits 0000000070000006/0000007ff8000800]
[   55.733312] MTRR cpu 1 variable 5 0000000078000006/0000007fff000800 -> 0000000000000000/0000000000000000 [bits 0000000078000006/0000007fff000800]
[   55.738054] MTRR cpu 1 fixed 0,0: 0000000000000000 -> 0606060606060606 [bits 0606060606060606]
[   55.741417] MTRR cpu 1 fixed 1,0: 0000000000000000 -> 0606060606060606 [bits 0606060606060606]
[   55.744799] MTRR cpu 1 fixed 2,0: 0000000000000000 -> 0505050505050505 [bits 0505050505050505]
[   55.748186] MTRR cpu 1 fixed 2,1: 0000000000000000 -> 0505050505050505 [bits 0505050505050505]
[   55.751528] MTRR cpu 1 fixed 2,2: 0000000000000000 -> 0505050505050505 [bits 0505050505050505]
[   55.754863] MTRR cpu 1 fixed 2,3: 0000000000000000 -> 0505050505050505 [bits 0505050505050505]
[   55.758254] MTRR cpu 1 changes:VFT
[   55.759807] MTRR cpu 1 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   55.763117] CPU1 is up
[   55.763211] smpboot: Booting Node 0 Processor 2 APIC 0x4
[   55.776468] MTRR cpu 1 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[   55.779425] MTRR cpu 1 changes:___
[   55.780902] MTRR cpu 1 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   55.783466] MTRR cpu 0 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[   55.786296] MTRR cpu 0 changes:___
[   55.787714] MTRR cpu 0 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   55.790183] MTRR cpu 2 disable: 0000000000000c00 -> 0000000000000000 [bits 0000000000000c00]
[   55.792538] MTRR cpu 2 variable 0 00000000ff800005/0000007fff800800 -> 0000000080000000/0000007f80000800 [bits 000000007f800005/000000007f800000]
[   55.796725] MTRR cpu 2 variable 1 0000000000000006/0000007fc0000800 -> 000000007c000000/0000007ffc000800 [bits 000000007c000006/000000003c000000]
[   55.800829] MTRR cpu 2 variable 2 0000000040000006/0000007fe0000800 -> 000000007a000000/0000007ffe000800 [bits 000000003a000006/000000001e000000]
[   55.805011] MTRR cpu 2 variable 3 0000000060000006/0000007ff0000800 -> 0000000079000000/0000007fff000800 [bits 0000000019000006/000000000f000000]
[   55.809186] MTRR cpu 2 variable 4 0000000070000006/0000007ff8000800 -> 0000000000000000/0000000000000000 [bits 0000000070000006/0000007ff8000800]
[   55.813227] MTRR cpu 2 variable 5 0000000078000006/0000007fff000800 -> 0000000000000000/0000000000000000 [bits 0000000078000006/0000007fff000800]
[   55.817323] MTRR cpu 2 fixed 0,0: 0000000000000000 -> 0606060606060606 [bits 0606060606060606]
[   55.820258] MTRR cpu 2 fixed 1,0: 0000000000000000 -> 0606060606060606 [bits 0606060606060606]
[   55.823181] MTRR cpu 2 fixed 2,0: 0000000000000000 -> 0505050505050505 [bits 0505050505050505]
[   55.826088] MTRR cpu 2 fixed 2,1: 0000000000000000 -> 0505050505050505 [bits 0505050505050505]
[   55.828995] MTRR cpu 2 fixed 2,2: 0000000000000000 -> 0505050505050505 [bits 0505050505050505]
[   55.831911] MTRR cpu 2 fixed 2,3: 0000000000000000 -> 0505050505050505 [bits 0505050505050505]
[   55.834847] MTRR cpu 2 changes:VFT
[   55.836203] MTRR cpu 2 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   55.839180] CPU2 is up
[   55.839284] smpboot: Booting Node 0 Processor 3 APIC 0x6
[   55.851534] MTRR cpu 2 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[   55.853881] MTRR cpu 2 changes:___
[   55.855054] MTRR cpu 2 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   55.857090] MTRR cpu 1 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[   55.859432] MTRR cpu 1 changes:___
[   55.860604] MTRR cpu 1 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   55.862635] MTRR cpu 0 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[   55.865099] MTRR cpu 0 changes:___
[   55.866322] MTRR cpu 0 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   55.868445] MTRR cpu 3 disable: 0000000000000c00 -> 0000000000000000 [bits 0000000000000c00]
[   55.870433] MTRR cpu 3 variable 0 00000000ff800005/0000007fff800800 -> 0000000080000000/0000007f80000800 [bits 000000007f800005/000000007f800000]
[   55.873944] MTRR cpu 3 variable 1 0000000000000006/0000007fc0000800 -> 000000007c000000/0000007ffc000800 [bits 000000007c000006/000000003c000000]
[   55.877384] MTRR cpu 3 variable 2 0000000040000006/0000007fe0000800 -> 000000007a000000/0000007ffe000800 [bits 000000003a000006/000000001e000000]
[   55.880894] MTRR cpu 3 variable 3 0000000060000006/0000007ff0000800 -> 0000000079000000/0000007fff000800 [bits 0000000019000006/000000000f000000]
[   55.884398] MTRR cpu 3 variable 4 0000000070000006/0000007ff8000800 -> 0000000000000000/0000000000000000 [bits 0000000070000006/0000007ff8000800]
[   55.887793] MTRR cpu 3 variable 5 0000000078000006/0000007fff000800 -> 0000000000000000/0000000000000000 [bits 0000000078000006/0000007fff000800]
[   55.891238] MTRR cpu 3 fixed 0,0: 0000000000000000 -> 0606060606060606 [bits 0606060606060606]
[   55.893696] MTRR cpu 3 fixed 1,0: 0000000000000000 -> 0606060606060606 [bits 0606060606060606]
[   55.896154] MTRR cpu 3 fixed 2,0: 0000000000000000 -> 0505050505050505 [bits 0505050505050505]
[   55.898601] MTRR cpu 3 fixed 2,1: 0000000000000000 -> 0505050505050505 [bits 0505050505050505]
[   55.901045] MTRR cpu 3 fixed 2,2: 0000000000000000 -> 0505050505050505 [bits 0505050505050505]
[   55.903492] MTRR cpu 3 fixed 2,3: 0000000000000000 -> 0505050505050505 [bits 0505050505050505]
[   55.905960] MTRR cpu 3 changes:VFT
[   55.907093] MTRR cpu 3 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   55.909590] CPU3 is up
[   55.909650] smpboot: Booting Node 0 Processor 4 APIC 0x1
[   55.916427] MTRR cpu 4 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[   55.918822] MTRR cpu 4 changes:___
[   55.920014] MTRR cpu 4 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   55.922087] MTRR cpu 0 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[   55.924365] MTRR cpu 0 changes:___
[   55.925491] MTRR cpu 0 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   55.927460] MTRR cpu 3 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[   55.929726] MTRR cpu 3 changes:___
[   55.930853] MTRR cpu 3 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   55.932802] MTRR cpu 1 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[   55.935051] MTRR cpu 1 changes:___
[   55.936175] MTRR cpu 1 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   55.938122] MTRR cpu 2 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[   55.940382] MTRR cpu 2 changes:___
[   55.941508] MTRR cpu 2 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   55.943953] CPU4 is up
[   55.944009] smpboot: Booting Node 0 Processor 5 APIC 0x3
[   55.950421] MTRR cpu 1 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[   55.952702] MTRR cpu 1 changes:___
[   55.953848] MTRR cpu 1 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   55.955833] MTRR cpu 2 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[   55.958079] MTRR cpu 2 changes:___
[   55.959199] MTRR cpu 2 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   55.961145] MTRR cpu 4 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[   55.963406] MTRR cpu 4 changes:___
[   55.964527] MTRR cpu 4 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   55.966489] MTRR cpu 3 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[   55.968728] MTRR cpu 3 changes:___
[   55.969854] MTRR cpu 3 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   55.971797] MTRR cpu 0 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[   55.974140] MTRR cpu 0 changes:___
[   55.975318] MTRR cpu 0 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   55.977347] MTRR cpu 5 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[   55.979759] MTRR cpu 5 changes:___
[   55.980958] MTRR cpu 5 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   55.983448] CPU5 is up
[   55.983504] smpboot: Booting Node 0 Processor 6 APIC 0x5
[   55.990424] MTRR cpu 1 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[   55.992715] MTRR cpu 1 changes:___
[   55.993853] MTRR cpu 1 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   55.995833] MTRR cpu 3 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[   55.998000] MTRR cpu 3 changes:___
[   55.999081] MTRR cpu 3 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   56.000958] MTRR cpu 6 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[   56.003220] MTRR cpu 6 changes:___
[   56.004339] MTRR cpu 6 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   56.006291] MTRR cpu 4 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[   56.008561] MTRR cpu 4 changes:___
[   56.009687] MTRR cpu 4 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   56.011658] MTRR cpu 2 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[   56.014098] MTRR cpu 2 changes:___
[   56.015301] MTRR cpu 2 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   56.017376] MTRR cpu 0 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[   56.019732] MTRR cpu 0 changes:___
[   56.020904] MTRR cpu 0 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   56.022935] MTRR cpu 5 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[   56.025340] MTRR cpu 5 changes:___
[   56.026537] MTRR cpu 5 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   56.029041] CPU6 is up
[   56.029093] smpboot: Booting Node 0 Processor 7 APIC 0x7
[   56.035427] MTRR cpu 3 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[   56.037818] MTRR cpu 3 changes:___
[   56.039003] MTRR cpu 3 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   56.041070] MTRR cpu 7 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[   56.043402] MTRR cpu 7 changes:___
[   56.044563] MTRR cpu 7 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   56.046580] MTRR cpu 1 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[   56.048869] MTRR cpu 1 changes:___
[   56.050005] MTRR cpu 1 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   56.051987] MTRR cpu 0 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[   56.054281] MTRR cpu 0 changes:___
[   56.055419] MTRR cpu 0 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   56.057403] MTRR cpu 2 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[   56.059629] MTRR cpu 2 changes:___
[   56.060733] MTRR cpu 2 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   56.062668] MTRR cpu 4 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[   56.065104] MTRR cpu 4 changes:___
[   56.066300] MTRR cpu 4 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   56.068373] MTRR cpu 5 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[   56.070794] MTRR cpu 5 changes:___
[   56.071995] MTRR cpu 5 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   56.074082] MTRR cpu 6 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[   56.076502] MTRR cpu 6 changes:___
[   56.077710] MTRR cpu 6 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   56.080239] CPU7 is up
[   56.080375] MTRR cpu 1 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[   56.082661] MTRR cpu 1 changes:___
[   56.083798] MTRR cpu 1 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   56.085786] MTRR cpu 0 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[   56.088084] MTRR cpu 0 changes:___
[   56.089221] MTRR cpu 0 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   56.091203] MTRR cpu 6 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[   56.093471] MTRR cpu 6 changes:___
[   56.094591] MTRR cpu 6 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   56.096551] MTRR cpu 4 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[   56.098930] MTRR cpu 4 changes:___
[   56.100114] MTRR cpu 4 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   56.102174] MTRR cpu 7 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[   56.104468] MTRR cpu 7 changes:___
[   56.105609] MTRR cpu 7 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   56.107604] MTRR cpu 2 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[   56.110075] MTRR cpu 2 changes:___
[   56.111283] MTRR cpu 2 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   56.113370] MTRR cpu 3 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[   56.115803] MTRR cpu 3 changes:___
[   56.117012] MTRR cpu 3 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   56.119109] MTRR cpu 5 disable: 0000000000000c06 -> 0000000000000000 [bits 0000000000000c06]
[   56.121520] MTRR cpu 5 changes:___
[   56.122718] MTRR cpu 5 enable: 0000000000000000 -> 0000000000000c06 [bits 0000000000000c06]
[   56.124707] ACPI: PM: Waking up from system sleep state S3
```

### Cost of extra checks

Now the problem is solved, and the MTRR updates appear correct. However, if this code were triggered on a system with many CPUs, then the number of times the cache rendezvous handler would be executed would be large - the multiplier being [triangular](https://en.wikipedia.org/wiki/Triangular_number): n(n+1)/2.

I don't see a way to avoid this. We can't minimise the time the CPU runs with inconsistent MTRRs without incurring the cost of synchronising the CPUs to update them. Breaking the synchronisation rules might appear to work, but have unforeseen consequences.

### Add diagnostic

We can see what's happening because we have `mtrr=debug` on the kernel command line. If we take it off, we'd like to have something in the kernel output that indicates that the MTRRs were corrected.

The easiest thing to do is to add a diagnostic to [`cache_aps_init()`](https://elixir.bootlin.com/linux/v6.16.6/source/arch/x86/kernel/cpu/cacheinfo.c#L807), and at the same time skip the now unneccessary delayed cache init.

changes from [commit](https://github.com/cgpe-a/linux/commit/d161fde1e9712eb2de01959c04db419190522372):
```
@@ -809,8 +823,13 @@ void cache_aps_init(void)
        if (!memory_caching_control || !get_cache_aps_delayed_init())
                return;

-       stop_machine(cache_rendezvous_handler, NULL, cpu_online_mask);
+       if (cache_bp_changed_on_restore)
+               pr_warn("mtrr: your CPUs had inconsistent MTRR settings on resume\n");
+       else
+               stop_machine(cache_rendezvous_handler, NULL, cpu_online_mask);
+
        set_cache_aps_delayed_init(false);
+       cache_bp_changed_on_restore = false;
 }

 static int __init cache_ap_register(void)
```

Test with `mtrr=debug` removed:
```
$ make -j$(nproc)
$ sudo make install
$ sudo kernelstub -k /boot/vmlinuz-6.16.7-custom-dirty -i /boot/initrd.img-6.16.7-custom-dirty -d "mtrr=debug"
$ reboot
```
Now we just get the single diagnostic on resume:
```
[   60.352278] ACPI: PM: Low-level resume complete
[   60.352316] ACPI: EC: EC started
[   60.352318] ACPI: PM: Restoring platform NVS memory
[   60.352899] Enabling non-boot CPUs ...
[   60.353038] smpboot: Booting Node 0 Processor 1 APIC 0x2
[   60.370186] CPU1 is up
[   60.370293] smpboot: Booting Node 0 Processor 2 APIC 0x4
[   60.386170] CPU2 is up
[   60.386263] smpboot: Booting Node 0 Processor 3 APIC 0x6
[   60.401311] CPU3 is up
[   60.401374] smpboot: Booting Node 0 Processor 4 APIC 0x1
[   60.411310] CPU4 is up
[   60.411363] smpboot: Booting Node 0 Processor 5 APIC 0x3
[   60.421879] CPU5 is up
[   60.421932] smpboot: Booting Node 0 Processor 6 APIC 0x5
[   60.432381] CPU6 is up
[   60.432433] smpboot: Booting Node 0 Processor 7 APIC 0x7
[   60.443996] CPU7 is up
[   60.443999] mtrr: your CPUs had unexpected MTRR settings on resume
[   60.444004] ACPI: PM: Waking up from system sleep state S3
```

### Commit changes

Preserve the changes we want to keep by comitting them to a branch. Use one commit for the debug changes (that we won't submit upstream), and one for the fix (which we will):
```
$ git config --global user.name "Chris Paulson-Ellis"
$ git config --global user.email "chris@paulson-ellis.org"
$ git checkout -b mtrr-resume-fix
$ git add -p arch/x86/kernel/cpu/mtrr/generic.c
$ git commit
$ git add -p arch/x86/include/asm/mtrr.h
$ git add -p arch/x86/kernel/cpu/mtrr/generic.c
$ git add -p arch/x86/kernel/cpu/cacheinfo.c
$ git commit
$ git log --oneline -3
d161fde1e971 (HEAD -> mtrr-resume-fix) x86/mtrr: Expedite cache init if MTRRs change on resume
884399682120 x86/mtrr: Print MTRR changes when mtrr=debug set [don't merge]
131e2001572b (tag: v6.16.7, stable/linux-6.16.y) Linux 6.16.7
```

### Test on main branch

Test just the fix commit on the current mainline:
```
$ git pull -p
$ git branch --no-track mtrr-resume-fix-2 origin/master
$ git checkout mtrr-resume-fix-2
$ git cherry-pick d161fde1e971
$ git log --oneline -2
5642f778c714 (HEAD -> mtrr-resume-fix-2) x86/mtrr: Expedite cache init if MTRRs change on resume
4d310797262f (origin/master, origin/HEAD) Merge tag 'pm-6.19-rc8' of git://git.kernel.org/pub/scm/linux/kernel/git/rafael/linux-pm
$ make oldconfig
$ git describe
v6.19-rc7-109-g5642f778c714
$ cp .config .config-v6.19-rc7-109-g5642f778c714-localmodconfig
$ make -j$(nproc)
$ sudo make modules_install
$ sudo make install
$ reboot
...
$ uname -r
6.19.0-rc7-custom-00109-g5642f778c714
```
It still works.

## Patch submission

### Subsystem tree

The x86 cache & MTRR code doesn't have a specific entry in `MAINTAINERS`, so the relevent section is [X86 ARCHITECTURE (32-BIT AND 64-BIT)](https://elixir.bootlin.com/linux/v6.19-rc5/source/MAINTAINERS#L28206):
```
X86 ARCHITECTURE (32-BIT AND 64-BIT)
M:	Thomas Gleixner <tglx@kernel.org>
M:	Ingo Molnar <mingo@redhat.com>
M:	Borislav Petkov <bp@alien8.de>
M:	Dave Hansen <dave.hansen@linux.intel.com>
M:	x86@kernel.org
R:	"H. Peter Anvin" <hpa@zytor.com>
L:	linux-kernel@vger.kernel.org
S:	Maintained
T:	git git://git.kernel.org/pub/scm/linux/kernel/git/tip/tip.git x86/core
F:	Documentation/arch/x86/
F:	Documentation/devicetree/bindings/x86/
F:	arch/x86/
F:	tools/testing/selftests/x86
```

This shows us that development happens in the `x86/core` branch of the [tip tree](https://docs.kernel.org/process/maintainer-tip.html).

We therefore need to check our changes there:
```
$ git remote add tip git://git.kernel.org/pub/scm/linux/kernel/git/tip/tip.git
$ git remote -v
origin	https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git (fetch)
origin	https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git (push)
stable	git://git.kernel.org/pub/scm/linux/kernel/git/stable/linux-stable.git (fetch)
stable	git://git.kernel.org/pub/scm/linux/kernel/git/stable/linux-stable.git (push)
tip	git://git.kernel.org/pub/scm/linux/kernel/git/tip/tip.git (fetch)
tip	git://git.kernel.org/pub/scm/linux/kernel/git/tip/tip.git (push)
$ git fetch tip
$ git branch -a | grep x86/core
  remotes/tip/x86/core
```
However, the `x86/core` branch is (currently) the same as `v6.19-rc5`, which is older than we've already tested, which was `v6.19-rc7-109-g5642f778c714`:
```
$ git log --oneline -1 tip/x86/core
0f61b1860cc3 (tag: v6.19-rc5, tip/x86/core) Linux 6.19-rc5
```
So, there are no x86 specific changes pending that we need to test against.

### git send-email

As per the `MAINTAINERS` entry, we'll need to send our changes to the Linux kernel mailing list. We don't need to subscribe to send. This is good, because we don't want to receive hundreds of emails a day, even if filtered. However, we do need some way to send well-formatted plain-text emails.

As a Gmail user, I don't want to mess around with setting up an alternative [suitable MUA](https://docs.kernel.org/process/email-clients.html), so I'll use [`git send-email`](https://git-scm.com/docs/git-send-email) at the command line.

The send-email sub-command needs to be installed:
```
$ sudo apt install git-email
```

I use [smtp2go](https://www.smtp2go.com/) to send emails. Configure the SMTP settings in `~/.gitconfig`:
```
[sendemail]
        smtpServer = mail-eu.smtp2go.com
        smtpServerPort = 2525
        smtpEncryption = tls
        smtpUser = my-smtp2go-user
        smtpPass = my-smtp2go-password
```

Send a test email to myself. The text only needs a Subject: header to be accepted:
```
$ echo -e "Subject: hello\n\nHello Chris" > /tmp/hello
$ cat /tmp/hello
Subject: hello

Hello Chris
$ git send-email --to=chris@paulson-ellis.org /tmp/hello
```

I received it (in Gmail).

Messages on [lore.kernel.org](https://lore.kernel.org/lkml) have a trailer that describes how to reply to messages using `git send-email`.

### b4

The [rules](https://www.kernel.org/doc/html/latest/process/submitting-patches.html) for proper patch submission are complex, so I'll use [b4](https://b4.docs.kernel.org/en/latest/) to help avoid making technical mistakes.

Install b4, as per the [instructions](https://b4.docs.kernel.org/en/latest/installing.html):
```
$ sudo apt install b4
$ b4 --version
0.13.0
```

Test [retrieval](https://b4.docs.kernel.org/en/latest/maintainer/mbox.html) of a recent [MTRR related thread](https://lore.kernel.org/lkml/20260130113625.599305-1-jgross@suse.com/) found on [lore](https://lore.kernel.org/lkml/):
```
$ b4 mbox 20260130113625.599305-1-jgross@suse.com
Grabbing thread from lore.kernel.org/all/20260130113625.599305-1-jgross@suse.com/t.mbox.gz
5 messages in the thread
Saved ./20260130113625.599305-1-jgross@suse.com.mbx
```

For sending, b4 doesn't need to be configured explicitly, because it uses the git-send-email SMTP config.

### Composing reply emails

We can now retrieve threads, and send emails - either arbitrary messages using [`git send-email`](https://git-scm.com/docs/git-send-email), or patches using [`b4 send`](https://b4.docs.kernel.org/en/latest/contributor/send.html).

How do we pick a message from an `mbox` file saved by b4 for a thread, and generate a reply file with the message quoted, ready for editing - prior to sending with git send-email?

I can't find a way to do this easily on the CLI, so I'll just do replies in Gmail using plain text mode. Oh well... Maybe b4 will be enhanced for this sort of thing at some point. 

### Publish commits

I'd like to reference my commits in this document, and make it public. So, I need to push to a public server. I have an account on GitHub.

First fork [torvalds/linux](https://github.com/torvalds/linux) to [cgpe-a/linux](https://github.com/cgpe-a/linux).

Set up SSH auth:
```
$ ssh-keygen -t ed25519 -C "chris@paulson-ellis.org"
$ echo "my-key-pasphrase-hint" > ~/.ssh/id_ed25519.hint
$ eval "$(ssh-agent -s)"
$ ssh-add ~/.ssh/id_ed25519
```
[Add](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account) SSH key to github account, and test:
```
ssh -T git@github.com
```

Add as a remote, and fetch to test:
```
$ git remote add github git@github.com:cgpe-a/linux.git
$ git fetch github
```

Push the original branch that includes the debug commit:
```
$ git push github mtrr-resume-fix
```
The changes are now [public](https://github.com/cgpe-a/linux/commits/mtrr-resume-fix/).

### Publish these notes (convert to HTML)

Details in this doc may be useful to refer to from the patch cover letter. For example, someone might want to know about the specific MTRR values I'm seeing. So, I need to publish it.

Install [Sphinx](https://www.sphinx-doc.org/en/master/), and [MyST Parser](https://myst-parser.readthedocs.io/en/latest/) for Markdown support:
```
$ sudo apt install python3-sphinx
$ sudo apt install python3-myst-parser
$ sphinx-build --version
sphinx-build 7.2.6
```

Create project:
```
$ sphinx-quickstart
> Separate source and build directories (y/n) [n]: y
> Project name: MacBook Pro Resume Delay
> Author name(s): Chris Paulson-Ellis
> Project release []:
```
Add [support](https://www.sphinx-doc.org/en/master/usage/markdown.html) for [Markdown](https://en.wikipedia.org/wiki/Markdown) in `source/conf.py`:
```
extensions = ['myst_parser']
```
Add this doc as "journey" to the [toctree](https://www.sphinx-doc.org/en/master/usage/restructuredtext/directives.html#directive-toctree) in `source/index.rst`:
```
.. toctree::
   :maxdepth: 2
   :caption: Contents:

   journey
```
Copy this doc in for now. We might convert it to [reStructuredText](https://en.wikipedia.org/wiki/ReStructuredText) later:
```
$ cp macbook-resume-delay.md source/journey.md
```
Build to HTML and review:
```
$ make html
$ xdg-open build/html/index.html
```

Not bad, but several modifications are required to fix problems:

To make the sidebar wide enough, to make the console output readable by using the full width of the window, and to fix the sidebar in place when the content scrolls, add to `source/conf.py`:
```
html_theme_options = {
        'body_max_width' : 'none',
        'page_width': 'auto',
        'fixed_sidebar': 'true',
        'sidebar_width': '350px'
}
```

To add the sub-headings to the sidebar, increment `maxdepth` in `source/index.rst`:
```
   :maxdepth: 3
```

To make the sidebar independently scrollable, add `source/_static/custom.css`:
```
div.sphinxsidebar {
	max-height: 100%;
	overflow-y: auto;
}
```

Shorten the main title in `source/index.rst`:
```
MacBook Pro Resume Delay
========================
```

After rebuilding, the resulting HTML is now acceptable.

### Publish these notes (Upload to CloudFlare)

Cloudflare Pages [supports](https://developers.cloudflare.com/pages/framework-guides/deploy-a-sphinx-site/) Sphinx, so it makes sense to use this facility in the ["docs as code"](https://www.writethedocs.org/guide/docs-as-code/) style, rather than uploading locally generated HTML.

The docs seem to imply that Cloudflare requires a virtualenv set up using [pipenv](https://pipenv.pypa.io/en/latest/), so although I already have Sphinx working natively, we need to re-install using pipenv:

```
$ sudo apt install pipenv
$ pipenv --version
pipenv, version 2023.12.1
$ pipenv install
$ pipenv install sphinx myst-parser
$ pipenv shell
(resume-delay) $ make html
(resume-delay) $ exit
```
Still works.

Create a git repo:
```
$ git config --global init.defaultBranch main
$ echo /build/ > .gitignore
$ echo \*.swp >> .gitignore
$ git init
$ git add .
$ git commit
```
Create a new [cgpe-a/macbook-pro-resume-delay](https://github.com/cgpe-a/macbook-pro-resume-delay) repo on GitHub, and upload:
```
$ git remote add origin git@github.com:cgpe-a/macbook-pro-resume-delay.git
$ git push origin main
```
Create a Cloudflare account, and on Dashboard, Build, Compute & AI, Workers & Pages, Create application, Connect GitHub,... (lots of faffing)... Create a worker, Import an existing Git repository, select GitHub cgpe-a/macbook-pro-resume-delay, Build command = make html, Build output directory = build/html, Environment variables, PYTHON_VERSION = 3.12.

The deploy fails. We have to continue and create the worker anyway so that we can then change the settings.

The Cloudflare Pages [build docs](https://developers.cloudflare.com/pages/configuration/build-image) say that the v3 build system doesn't support pipenv. In the Settings for the project we need to set the Build system version to Version 1, and set PYTHON_VERSION to 3.7.

Retrying the deploy still doesn't work, because the project `Pipfile` & `Pipfile.lock` were created using python 3.12. We need to remove it:
```
$ pipenv --rm
$ rm Pipenv*
```

To create a python 3.7 pipenv, we need to install python 3.7, which isn't in the Ubuntu repos. We can use the [deadsnakes ppa](https://launchpad.net/~deadsnakes/+archive/ubuntu/ppa) to install it:
```
$ sudo add-apt-repository ppa:deadsnakes/ppa
$ sudo apt update
$ sudo apt install python3.7 python3.7-distutils python3.7-venv
$ python3.7 -V
Python 3.7.17
```

We can't use our existing pipenv, because it [doesn't work](https://github.com/pypa/pipenv/issues/6265) with python 3.7. We need to install the older 2023.10.3 version of pipenv.

We also need to isolate the new pipenv installation from the system one, so we need a venv which we can use to re-install our pipenv:
```
$ python3.7 -m venv ~/python3.7venv
$ ~/python3.7venv/bin/python -m pip install pipenv==2023.10.3
$ ~/python3.7venv/bin/python -m pipenv --version
pipenv, version 2023.10.3
$ ~/python3.7venv/bin/python -m pipenv --python 3.7
$ ~/python3.7venv/bin/python -m pipenv install
$ ~/python3.7venv/bin/python -m pipenv install sphinx myst-parser
$ ~/python3.7venv/bin/python -m pipenv shell
(resume-delay) $ make html
(resume-delay) $ exit
```

Now we have a `Pipfile` & `Pipfile.lock` that is python 3.7 specific, not python 3.12. Commit & push:
```
$ git add Pipfile Pipfile.lock
$ git commit
[main 9cae7b9] Regenerate Pipenv using python 3.7
 2 files changed, 170 insertions(+), 244 deletions(-)
$ git push -u origin main
```
The push triggers an attempt to redeploy the site. It's still broken:
```
17:46:33.198	plette.models.base.ValidationError: {u'python_full_version': u'3.7.17', u'python_version': u'3.7'}
17:46:33.198	python_full_version: 'python_version' must not be present with 'python_full_version'
17:46:33.198	python_version: 'python_full_version' must not be present with 'python_version'
17:46:33.259	Error installing Pipenv dependencies
```
We need to remove the `python_full_version` lines manually:
```
$ git diff
diff --git a/Pipfile b/Pipfile
index 1d5dd5a..a760a1d 100644
--- a/Pipfile
+++ b/Pipfile
@@ -11,4 +11,3 @@ myst-parser = "*"

 [requires]
 python_version = "3.7"
-python_full_version = "3.7.17"
diff --git a/Pipfile.lock b/Pipfile.lock
index 5e2e0f6..e8f39c8 100644
--- a/Pipfile.lock
+++ b/Pipfile.lock
@@ -5,7 +5,6 @@
         },
         "pipfile-spec": 6,
         "requires": {
-            "python_full_version": "3.7.17",
             "python_version": "3.7"
         },
         "sources": [
$ git add Pipfile Pipfile.lock
$ git commit
[main 5ede887] Remove python_full_version from Pipfile & Pipfile.lock
 2 files changed, 2 deletions(-)
$ git push
```
The deploy now works, and the [site](https://macbook-pro-resume-delay.pages.dev/) is available.

### b4 prep

TODO: Create new "series", and write cover note. Base branch on latest -rc release. Test again.

### b4 send

TODO: Send!

## Patch review

Waiting...