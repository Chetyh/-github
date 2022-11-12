# COP4610 - Project 2 Guide - Part 8 - System Calls

## Authors

Brian Poblete

## Before we begin

- This guide was written in October 2022 with Ubuntu 22.04 Server edition in
mind
- It is definitely possible to use another distro and another version.
- But I would recommend using this same version for maximum compatibility.

## What are System Calls?

- Standard interface to allow the kernel to safely handle use requests
  - Examples:
    - Read from hardware
    - Spawn a new process
    - Get current time
    - Create shared memory
- It is a technique for passing messages between:
  - The OS kernel (server)
  - User (client)

## Executing System Calls

- User program issues call
- Core kernel looks up the call in the syscall table
- *Kernel module* handles the syscall action
- Module returns the result of the system call
- Core kernel forwards results to the user

![Executing system call 1](https://developers.elementor.com/docs/assets/img/elementor-placeholder-image.png)
![Executing system call 2](https://developers.elementor.com/docs/assets/img/elementor-placeholder-image.png)
![Executing system call 3](https://developers.elementor.com/docs/assets/img/elementor-placeholder-image.png)
![Executing system call 4](https://developers.elementor.com/docs/assets/img/elementor-placeholder-image.png)

## What if the module is not loaded?

- User program issues call
- Core kernel looks up the call in the syscall table
- Kernel modules isn't loaded to handle the action
- (...)
- Where does the call go?

![Module not loaded 1](https://developers.elementor.com/docs/assets/img/elementor-placeholder-image.png)
![Module not laoded 2](https://developers.elementor.com/docs/assets/img/elementor-placeholder-image.png)

## System Call Wrappers

![Not this kind of rapper](https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcTI_GkFfaC0BDPRRAr6GeO13mSEI7Y5Ee9d_g&usqp=CAUhttps://img.thedailybeast.com/image/upload/dpr_2.0/c_crop,h_1440,w_1440,x_512,y_0/c_limit,w_128/d_placeholder_euli9k,fl_lossy,q_auto/v1512956943/171210-stereo-eminem-lede-2_pf4tpd)

`Not this kind of rapper`

- System call wrappers handle this scenario of missing modules by:
  - Calling the system call handler function *if* the module is loaded
  - Else, return an error
- Wrappers use a function pointer to point to the system call handler function
- You should add one system call wrapper for each system call you add

## Adding System Calls

- Create and add the following files to your kernel.
Note the position of the file within your kernel directory.
  - `/usr/src/test_kernel/SystemCalls/test_call.c`
  - `/usr/src/test_kernel/SystemCalls/Makefile`
  - `/usr/src/test_kernel/SystemCalls/syscallModule.c`

- Modify these files. They should already exist in your kernel directory
  - `/usr/src/test_kernel/arch/x86/entry/syscalls/syscall_64.tbl`
  - `/usr/src/test_kernel/include/linux/syscalls.h`
  - `/usr/src/test_kernel/Makefile`

## test_kernel/SystemCalls/test_call.c

```C
#include <linux/linkage.h>
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/syscalls.h>

long (*STUB_test_call)(int) = NULL;
// ^ This is the syscall pointer (function pointer).
//   Also known as a stub.

EXPORT_SYMBOL(STUB_test_call);
// ^ Exports the syscall pointer so the handler module can access it.

// System call wrapper
SYSCALL_DEFINE1(test_call, int, test_int) {
 printk(KERN_NOTICE "Inside SYSCALL_DEFINE1 block. %s: Your int is %d\n", __FUNCTION__, test_int);
 if (STUB_test_call != NULL)
  return STUB_test_call(test_int); // Execute the function if defined.
 else
  return -ENOSYS; // Return error if not defined.
}
```

## SYSCALL_DEFINE*n*

- SYSCALL_DEFINE*n* is a macro that generates the proper system call definition
with the appropriate arguments.
- SYSCALL_DEFINE1
  - *1* means the system call you are creating will take 1 argument
  - Example
    - `SYSCALL_DEFINE1(test_call, int, test_int)`
    - This creates a system call named `sys_test_call` that takes one `int`
        type argument named `test_int`
         (notice the comman between `int` and `test_int`)
    - `SYSCALL_DEFINE0` to `SYSCALL_DEFINE6` are available in this format
      - You can pass a minimum or zero arguments up to a maximum of
         6 arguments.

## test_kernel/SystemCalls/syscallModule.c

```C
#include <linux/init.h>
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/linkage.h>
MODULE_LICENSE("GPL");

extern long (*STUB_test_call)(int);
// ^ Access the syscall pointer you created earlier.

long my_test_call(int test) {
 printk(KERN_NOTICE "%s: Your int is %d\n", __FUNCTION__, test);
 return test;
}
// ^ The implementation of the system call handler.
//   This is the definition of the function that is pointed to by the function
//   pointer.

static int hello_init(void) {
 STUB_test_call = my_test_call;
 // ^ Assigns the syscall pointer to the syscall handler function when the
 //   module is loaded.

 return 0;
}
module_init(hello_init);

static void hello_exit(void) {
 STUB_test_call = NULL;
 // ^ Empties the syscall pointer when the module is unloaded.
}
module_exit(hello_exit);
```

## test_kernel/SystemCalls/Makefile

```Makefile
obj-y := test_call.o
obj-m := syscallModule.o

PWD := $(shell pwd)
KDIR := /lib/modules/`uname -r`/build

default:
 $(MAKE) -C $(KDIR) SUBDIRS=$(PWD) modules

clean:
 rm -f *.o *.ko *.mod.* Module.* modules.*
```

## test_kernel/arch/x86/entry/syscalls/syscall_64.tbl

- Note that `arch` in the line above refers to architecture. It does not refer
to the [Arch Linux](https://archlinux.org/) distribution.

```tbl
541     x32     setsockopt              sys_setsockopt
542     x32     getsockopt              sys_getsockopt
543     x32     io_setup                compat_sys_io_setup
544     x32     io_submit               compat_sys_io_submit
545     x32     execveat                compat_sys_execveat
546     x32     preadv2                 compat_sys_preadv64v2
547     x32     pwritev2                compat_sys_pwritev64v2
# This is the end of the legacy x32 range.  Numbers 548 and above are
# not special and are not to be used for x32-specific syscalls.
### END OF UNMODIFIED FILE. SYSCALL FOR THIS CLASS ADDED BELOW THIS LINE.

548     common  test_call               sys_test_call
```

- To ensure compatibility, I recommend adding your system call to the very
end of the .tbl file and assigning it the number that comes after the last used
syscall number.
- In October 2022, this number was 548. It might be different by the time you
are reading this so be aware of that.
- Format of the syscall is as follows:

    ```tbl
    548<tab>common<tab>test_call<tab><tab>sys_test_call
    ```

  - Note: replace `<tab>` with actual presses of the tab key. Using
    multiple spaces as it might not work. If the line you entered into the .tbl
    file does not look identical to the line in the large code block, it may
    not be read correctly when running `make`.

## test_kernel/include/syscalls.h

```H
long __do_semtimedop(int semid, struct sembuf *tsems, unsigned int nsops,
                     const struct timespec64 *timeout,
                     struct ipc_namespace *ns);

int __sys_getsockopt(int fd, int level, int optname, char __user *optval,
                int __user *optlen);
int __sys_setsockopt(int fd, int level, int optname, char __user *optval,
                int optlen);
// END OF UNMODIFIED FILE. CODE FOR THIS CLASS ADDED BELOW THIS LINE
asmlinkage long sys_test_call(int);

// CODE FOR THIS CLASS MUST BE ADDED ABOVE THIS #endif STATEMENT
#endif
```

- At thei very end of the `syscalls.h` document, we define the
system call prototype
- This time, we write sys_test_call. This matches the name at the end of the
line we added to the .tbl file.

## test_kernel/Makefile

```Makefile
  ### LINE NUMBERS ADDED FOR CONVENIENCE ONLY
  ### LINE NUMBERS ARE FOR THE MAKEFILE AS OF OCT. 2022
  ### LINE NUMBERS DO NOT APPEAR IN ACTUAL MAKEFILE
  ### DO NOT ADD LINE NUMBERS TO ACTUAL MAKEFILE
  1149 
  1150  ifeq ($(KBUILD_EXTMOD),)
  1151  core-y          += kernel/ certs/ mm/ fs/ ipc/ security/ crypto/ block/
  1152  core-y          += SystemCalls/
        # ^ Line to be added
  1153
  1154  vmlinux-dirs    := $(patsubst %/,%,$(filter %/, \
  1155                       $(core-y) $(core-m) $(drivers-y) $(drivers-m) \
  1156                       $(libs-y) $(libs-m)))
  1157
  1158  vmlinux-alldirs := $(sort $(vmlinux-dirs) Documentation \
  1159                       $(patsubst %/,%,$(filter %/, $(core-) \
  1160                          $(drivers-) $(libs-))))
  1161
```

## User space program to test syscall

- Before this step you should have:
  - Created the three files containing your custom syscall.
  - Modified the other three files to add your syscall to the kernel.
  - Compiled the kernel without any errors (warnings are acceptable).
  - Installed the kernel onto your machine.
  - Rebooted your machine.
- If these steps have been followed, your syscall should now be accessible
through the `sys/syscall.h` header in C.
- You can test this by compiling the file below with the `gcc` command and the
running the output file with the correct number of parameters.

```C
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/syscall.h>
#define __NR_TEST_CALL 335

int test_call(int test) {
 return syscall(__NR_TEST_CALL, test);
}

int main(int argc, char **argv) {
 if (argc != 2) {
  printf("wrong number of args\n");
  return -1;
 }
 
 int test = atoi(argv[1]);
 long ret = test_call(test);

 printf("sending this: %d\n", test);

 if (ret < 0)
  perror("system call error");
 else
  printf("Function successful. passed in: %d, returned %ld\n", test, ret);
 
 printf("Returned value: %ld\n", ret); 

 return 0;
}
```

- Sample usage:

    ```bash
    gcc main.c
    ./a.out 5
    ```

- If the syscall and module were added correctly, you should get the following
output

    ```txt
    sending this: 5
    Function successful. passed in: 5, returned 5
    Returned value: 5
    ```

- If either the syscall, module, or both were not loaded correctly, you should
get the follow output.

    ```txt
    sending this: 5
    system call error: Function not implemented
    Returned value: -1
    ```

- Double check all of the files you created/modified/added if you encounter an
error.

## Notes

- Adding a new system call requires re-compiling and reinstalling the whole
kernel
- However, a module can be added at any time without kernel reinstallation.
- Keep the system call definition function (test_call.c) really simple and
compile only once.
  - Ideally, it should only create the sys_call pointer and call it.
- Implement the actual system call handler functions as a module
(syscallModule.c).
  - You can compile the module as many times as needed.
