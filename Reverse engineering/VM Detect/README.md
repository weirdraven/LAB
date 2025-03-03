# Virtual Machine (VM) Detection Techniques

Virtual machine (VM) detection techniques are used to determine if a program is running inside a virtualized environment. This is often done for security reasons, such as preventing malware analysis, or for licensing purposes, such as restricting software to run only on physical hardware. Below are some common VM detection techniques used in Linux and other environments.
These methods work across different hypervisors such as VMware, VirtualBox, Hyper-V, and KVM/QEMU.

---

## 1. **Checking CPUID Instructions**

The `CPUID` instruction can be used to query information about the CPU, including whether it is running in a virtualized environment.

### Assembly (x86-64)
```assembly
mov eax, 1
cpuid
bt ecx, 31  ; Hypervisor present bit
jc is_vm    ; If set, running inside a VM
```

### C Code
```c
#include <stdio.h>
#include <cpuid.h>

int main() {
    unsigned int eax, ebx, ecx, edx;
    __cpuid(1, eax, ebx, ecx, edx);
    if (ecx & (1 << 31)) {
        printf("Running inside a VM\n");
    } else {
        printf("Running on bare metal\n");
    }
    return 0;
}
```

### C Code inline assembly
```c
#include <stdio.h>

void check_vm() {
    unsigned int eax, ebx, ecx, edx;
    eax = 0x1; // Function 1: Get CPU features
    __asm__("cpuid"
            : "=a"(eax), "=b"(ebx), "=c"(ecx), "=d"(edx)
            : "a"(eax));
    if (ecx & (1 << 31)) {
        printf("VM detected (Hypervisor bit set)\n");
    } else {
        printf("No VM detected\n");
    }
}

int main() {
    check_vm();
    return 0;
}		
```

---

## 2. **Checking Hypervisor Vendor String**

The CPUID instruction can also be used to retrieve the hypervisor vendor string.

### C Code
```c
#include <stdio.h>
#include <string.h>
#include <cpuid.h>

int main() {
    unsigned int eax, ebx, ecx, edx;
    char hypervisor[13] = {0};

    __cpuid(0x40000000, eax, ebx, ecx, edx);
    memcpy(hypervisor, &ebx, 4);
    memcpy(hypervisor + 4, &ecx, 4);
    memcpy(hypervisor + 8, &edx, 4);

    printf("Hypervisor: %s\n", hypervisor);
    return 0;
}
```
### C Code inline assembly

```c
#include <stdio.h>

void check_vm() {
    char vendor[13];
    unsigned int eax, ebx, ecx, edx;
    eax = 0x40000000; // Hypervisor CPUID leaf
    __asm__("cpuid"
            : "=a"(eax), "=b"(ebx), "=c"(ecx), "=d"(edx)
            : "a"(eax));
    *(unsigned int*)vendor = ebx;
    *(unsigned int*)(vendor + 4) = ecx;
    *(unsigned int*)(vendor + 8) = edx;
    vendor[12] = '\0';
    printf("Hypervisor vendor: %s\n", vendor);
    if (ebx != 0 || ecx != 0 || edx != 0) {
        printf("VM detected\n");
    } else {
        printf("No VM detected\n");
    }
}

int main() {
    check_vm();
    return 0;
}
```

### Common Hypervisor Strings
- **VMware** → `"VMwareVMware"`
- **VirtualBox** → `"VBoxVBoxVBox"`
- **Microsoft Hyper-V** → `"Microsoft Hv"`
- **QEMU/KVM** → `"KVMKVMKVM"`
- **Parallels** → `"prl hyperv"`

---

## 3. **Checking System BIOS & DMI Information**

Virtual machines often have specific BIOS/UEFI strings.

```sh
dmidecode -s system-manufacturer
dmidecode -s system-product-name
```
**Typical VM Outputs:**
- **VMware** → `"VMware, Inc."`
- **VirtualBox** → `"Oracle Corporation"`
- **QEMU/KVM** → `"QEMU"`
- **Hyper-V** → `"Microsoft Corporation"`

---

## 4. **Checking MAC Address Prefix**

Virtual machines often use specific ranges of MAC addresses.

```sh
ip link show | grep ether
```
### Common VM MAC Prefixes
- **VMware** → `00:05:69`, `00:0C:29`, `00:1C:14`
- **VirtualBox** → `08:00:27`
- **Microsoft Hyper-V** → `00:15:5D`
- **QEMU/KVM** → `52:54:00`

---

## 5. **Checking for Virtual Devices**

Virtual machines often emulate specific hardware devices. Checking for the presence of these devices can indicate a VM.

```sh
lsblk -o NAME,VENDOR,MODEL
```
### Common VM Disk Names
- **VMware** → `"VMware Virtual disk"`
- **VirtualBox** → `"VBOX HARDDISK"`
- **KVM/QEMU** → `"QEMU HARDDISK"`

---

## 6. **Checking CPU Features**
```sh
lscpu | grep "Hypervisor vendor"
```
If this field is present, the system is likely running inside a VM.

---

## 7. **Checking for Virtualized Devices (Linux)**
```sh
ls /dev | grep -i vbox
lsmod | grep -i vmw
```

---

## 8. **Checking for Virtualization-Specific Files**

Virtual machines often leave traces in the filesystem, such as specific drivers or configuration files.

### C Code
```c
#include <stdio.h>
#include <string.h>

int check_vm_files() {
    const char *files[] = {
        "/sys/class/dmi/id/product_name",
        "/sys/class/dmi/id/sys_vendor",
        "/proc/scsi/scsi",
        NULL
    };
    for (int i = 0; files[i] != NULL; i++) {
        FILE *fp = fopen(files[i], "r");
        if (fp) {
            char buffer[128];
            fgets(buffer, sizeof(buffer), fp);
            fclose(fp);
            if (strstr(buffer, "VMware") || strstr(buffer, "VirtualBox") || strstr(buffer, "QEMU")) {
                return 1;
            }
        }
    }
    return 0;
}

int main() {
    if (check_vm_files()) {
        printf("VM detected\n");
    } else {
        printf("No VM detected\n");
    }
    return 0;
}
```

---

## 9. **Checking for Virtualization-Specific Processes**

Virtual machines often run specific processes that can be detected.

```sh
ps aux | grep -i "vmtoolsd\|vboxservice\|qemu"
```

---

## 10. **Using Anti-VM Tools**

Tools like [VMDetect](https://github.com/PerryWerneck/vmdetect/) or [Pafish](https://github.com/a0rtega/pafish) can automate the detection of virtualized environments.

---

# Notes:

* VM detection techniques are often used by malware to evade analysis in sandboxes or virtualized environments.

* These techniques can be bypassed by skilled attackers or by using advanced virtualization solutions.

* Always consider the ethical implications of using VM detection techniques, especially in malware or DRM systems.

By combining multiple techniques, you can make VM detection more robust, but keep in mind that no method is 100% reliable.
