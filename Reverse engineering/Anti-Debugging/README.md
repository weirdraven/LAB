# Linux Anti-Debugging Techniques

## Overview

Debugging is a powerful tool for reverse engineering, exploit development, and software analysis. However, in security-sensitive applications, preventing debugging can help protect intellectual property, sensitive data, and critical operations. 

In Linux, several methods can detect and mitigate debugging attempts, making it harder for attackers to analyze or modify a program.

---

## 1. ptrace-based Detection
`ptrace` is a system call used by debuggers to inspect and control a process. A process can detect if it is being traced by attempting to attach to itself.
Linux debuggers like `GDB`, `strace`, and `ltrace` use `ptrace()` to attach to a process. If a process is already being traced, calling `ptrace()` again will fail.

### Implementation:
```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/ptrace.h>

void check_ptrace() {
    if (ptrace(PTRACE_TRACEME, 0, NULL, NULL) == -1) {
        printf("Debugger detected! Exiting...\n");
        exit(1);
    }
}

int main() {
    check_ptrace();
    printf("No debugger detected. Running normally.\n");
    return 0;
}
```


### How It Works

* If `ptrace(PTRACE_TRACEME, ...)` fails, another process (likely a debugger) is already attached.

* The program exits if debugging is detected.

---

## 2. Checking `/proc/self/status`
The `/proc/self/status` file contains process information, including the `TracerPid` field, which is nonzero if a debugger is attached.

The **Linux proc filesystem** provides information about a running process. The `TracerPid` field in `/proc/self/status` reveals if the process is being debugged.

### Implementation:
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

void check_proc_status() {
    FILE *fp = fopen("/proc/self/status", "r");
    if (!fp) return;

    char line[256];
    while (fgets(line, sizeof(line), fp)) {
        if (strncmp(line, "TracerPid:", 10) == 0) {
            if (atoi(&line[10]) > 0) {
                printf("Debugger detected! Exiting...\n");
                exit(1);
            }
        }
    }
    fclose(fp);
}

int main() {
    check_proc_status();
    printf("No debugger detected. Running normally.\n");
    return 0;
}
```

### How It Works

* Reads `/proc/self/status`.

* If `TracerPid` is nonzero, the process is being debugged.

---

## 3. Using `prctl()` to Disable Tracing

Linux provides `prctl()` to control process behavior. Setting `PR_SET_DUMPABLE` to `0` prevents debugging and core dumps.

The `prctl` system call allows a process to disable debugging by setting `PR_SET_DUMPABLE` to `0`.

### Implementation:
```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/prctl.h>

void disable_tracing() {
    prctl(PR_SET_DUMPABLE, 0);
}

int main() {
    disable_tracing();
    printf("Tracing disabled.\n");
    return 0;
}
```

### How It Works

* Prevents a debugger from attaching.

* Blocks access to memory dumps.

---

## 4. Detecting `LD_PRELOAD` & `LD_LIBRARY_PATH`
Dynamic linker environment variables like `LD_PRELOAD` and `LD_LIBRARY_PATH` can be used to inject libraries for debugging.

Debugging tools inject libraries via environment variables like `LD_PRELOAD` and `LD_LIBRARY_PATH`. Checking for these can detect unwanted tracing.

### Implementation:
```c
#include <stdio.h>
#include <stdlib.h>

void check_env() {
    if (getenv("LD_PRELOAD") || getenv("LD_LIBRARY_PATH")) {
        printf("Debugging environment detected! Exiting...\n");
        exit(1);
    }
}

int main() {
    check_env();
    printf("No debugging environment detected.\n");
    return 0;
}
```

### How It Works

* LD_PRELOAD injects libraries before execution.

* LD_LIBRARY_PATH changes library loading paths.

---

## 5. Using `fork()` to Self-Debug
A process can fork itself and call `ptrace` on its child. If a debugger is already attached, the `ptrace` call will fail.

### Implementation:
```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/ptrace.h>
#include <unistd.h>

void self_debug() {
    if (fork() == 0) {
        if (ptrace(PTRACE_TRACEME, 0, NULL, NULL) == -1) {
            printf("Debugger detected! Exiting...\n");
            exit(1);
        }
        exit(0);
    }
    wait(NULL);
}

int main() {
    self_debug();
    printf("No debugger detected. Running normally.\n");
    return 0;
}
```

### How It Works

* The child process attempts `ptrace(PTRACE_TRACEME)`, failing if debugging is active.

* The parent waits for the child to complete.

---

## 6. Virtual Machine Detection
Malware and anti-reversing techniques often check for virtual machines by detecting known VM artifacts.

### Implementation:
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

void detect_vm() {
    FILE *cpuinfo = fopen("/proc/cpuinfo", "r");
    if (!cpuinfo) return;
    
    char line[256];
    while (fgets(line, sizeof(line), cpuinfo)) {
        if (strstr(line, "hypervisor")) {
            printf("Virtual machine detected! Exiting...\n");
            exit(1);
        }
    }
    fclose(cpuinfo);
}

int main() {
    detect_vm();
    printf("No VM detected. Running normally.\n");
    return 0;
}
```

### How It Works

* Reads /proc/cpuinfo to check for the hypervisor flag.

* If present, the system is likely running inside a VM.

---

## 7. Using `seccomp` to Restrict Debugging
Seccomp (Secure Computing Mode) can restrict system calls, preventing the use of debugging tools.

### Implementation:
```c
#include <stdio.h>
#include <stdlib.h>
#include <linux/seccomp.h>
#include <sys/prctl.h>
#include <unistd.h>

void enable_seccomp() {
    prctl(PR_SET_SECCOMP, SECCOMP_MODE_STRICT);
}

int main() {
    enable_seccomp();
    printf("Seccomp enabled, debugging restricted.\n");
    return 0;
}
```

### How It Works

* Enables seccomp in strict mode, allowing only read, write, _exit, and sigreturn system calls.

* Prevents debugging tools from making necessary system calls.


---

## 8. Using Timing Analysis
A process can detect debugging by measuring execution time variations caused by breakpoints.

### Implementation:
```c
#include <stdio.h>
#include <time.h>

void detect_debugger_timing() {
    struct timespec start, end;
    clock_gettime(CLOCK_MONOTONIC, &start);

    for (volatile int i = 0; i < 100000000; i++);

    clock_gettime(CLOCK_MONOTONIC, &end);

    double elapsed = (end.tv_sec - start.tv_sec) + 
                     (end.tv_nsec - start.tv_nsec) / 1e9;
    if (elapsed > 0.1) {
        printf("Debugger detected! Exiting...\n");
        exit(1);
    }
}

int main() {
    detect_debugger_timing();
    printf("Running normally...\n");
    return 0;
}
```

### How It Works

* Measures execution time of a loop.

* Debugging slows down execution, causing detection

* Breakpoints and single-step debugging introduce noticeable execution delays.


---


These techniques provide multiple layers of anti-debugging protection. They can be combined for better security, though advanced attackers can still bypass them with specialized tools. Implementing such protections increases the difficulty of debugging and reverse engineering in Linux environments.
