# Lab — `kmod-hello` (Day 61)

Out-of-tree kernel module against the Cuttlefish / GKI kernel.

## Files
```
kmod-hello/
├── README.md
├── hello.c
├── Makefile
└── hello_trace_events.h    (optional, used in L6A §6A.2)
```

## `hello.c`
```c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>

static int __init hello_init(void)
{
    pr_info("hello: loaded, jiffies=%lu\n", jiffies);
    return 0;
}

static void __exit hello_exit(void)
{
    pr_info("hello: unloaded\n");
}

module_init(hello_init);
module_exit(hello_exit);

MODULE_LICENSE("GPL v2");
MODULE_AUTHOR("aosp-book");
MODULE_DESCRIPTION("hello world kernel module");
MODULE_VERSION("0.1");
```

## `Makefile`
```makefile
KDIR ?= $(HOME)/aosp-kernel/common
ARCH ?= x86_64
CROSS_COMPILE ?= x86_64-linux-gnu-

obj-m := hello.o

all:
	$(MAKE) -C $(KDIR) M=$(PWD) ARCH=$(ARCH) CROSS_COMPILE=$(CROSS_COMPILE) modules
clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

## Build & load
```bash
$ # Build the GKI/CF kernel first (see L4 §4.2)
$ cd ~/aosp-kernel/common && make ARCH=x86_64 defconfig && make -j$(nproc)
$ cd /path/to/kmod-hello && make
$ adb push hello.ko /data/local/tmp/
cf:# insmod /data/local/tmp/hello.ko
cf:# dmesg | tail
cf:# rmmod hello
```

## Verifying
- `dmesg` shows `hello: loaded, jiffies=...` after `insmod` and `hello: unloaded` after `rmmod`.
- `lsmod | grep hello` lists the module.

