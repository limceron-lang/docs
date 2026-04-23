# Platform Abstraction Layer (PAL)

## Overview

Limceron targets multiple operating systems and CPU architectures from a single codebase.
The Platform Abstraction Layer isolates all OS-specific and architecture-specific code
behind stable internal interfaces.

## Supported Platforms

### Operating Systems
| OS      | System Call Interface | Binary Format | Event Loop   |
|---------|----------------------|---------------|--------------|
| Linux   | syscall (direct)     | ELF64         | epoll        |
| macOS   | libSystem.dylib      | Mach-O 64     | kqueue       |
| Windows | ntdll.dll / kernel32 | PE/COFF       | IOCP         |

### CPU Architectures
| Architecture | Register Count | Calling Convention   | Status    |
|-------------|----------------|----------------------|-----------|
| x86_64      | 16 GPR, 16 XMM| System V / Win64     | Primary   |
| AArch64     | 31 GPR, 32 NEON| AAPCS64             | Primary   |
| RISC-V 64   | 32 GPR, 32 FPR| Standard             | Secondary |

## PAL Architecture

```
Application Code
       |
       v
+---------------------------------------------+
|           Limceron Standard Library           |
|   (fs, net, async, os, time, crypto)          |
+-----------------------------------------------+
|         Platform Abstraction Layer             |
|                                               |
|  +----------+ +----------+ +---------+       |
|  |  Memory   | |   I/O    | | Thread  |       |
|  | Management| | Interface| | Control |       |
|  +----------+ +----------+ +---------+       |
|  +----------+ +----------+ +---------+       |
|  |  File    | | Network  | |  Time   |       |
|  | System   | |  Stack   | | & Clock |       |
|  +----------+ +----------+ +---------+       |
+----------+----------+------------------------+
|  Linux   |  macOS   |     Windows             |
|  (POSIX) | (Darwin) |    (Win32/NT)           |
+----------+----------+------------------------+
       |
       v
    Kernel
```

## System Call Strategy

### Linux
Direct syscall via `syscall` instruction. No libc dependency for the runtime.
This ensures Limceron binaries work on any Linux kernel >= 3.17 regardless of libc version.

```
// Pseudocode — what the compiler generates
fn sys_write(fd: i32, buf: *const u8, len: usize) -> isize {
    // x86_64: syscall number in RAX, args in RDI, RSI, RDX
    asm("mov rax, 1; syscall")
}
```

### macOS
Call through `libSystem.B.dylib` (required by Apple — direct syscalls are not stable).
The runtime dynamically loads libSystem at startup.

### Windows
Call through `kernel32.dll` and `ntdll.dll` via the standard Windows API.
Loaded at startup via the PE import table.

## Conditional Compilation

Platform-specific code uses `comptime if`:

```limceron
comptime if os.name() == "linux" {
    fn event_loop_new() -> EventLoop {
        EpollLoop.new()
    }
} else if os.name() == "darwin" {
    fn event_loop_new() -> EventLoop {
        KqueueLoop.new()
    }
} else if os.name() == "windows" {
    fn event_loop_new() -> EventLoop {
        IocpLoop.new()
    }
}
```

The compiler eliminates unreachable branches at compile time — no runtime overhead.

## Path Handling

```limceron
// fs.Path automatically handles platform differences
let config = fs.Path.join(home_dir, ".config", "myapp", "config.json")
// Linux/macOS: /home/user/.config/myapp/config.json
// Windows:     C:\Users\user\.config\myapp\config.json

// Path always uses / in source code, converted at the OS boundary
let path = fs.Path.from("data/input/file.csv")
// Works on all platforms
```

## Endianness

Limceron is little-endian by default (matching x86_64, ARM64, and RISC-V).
Network byte order (big-endian) conversion is explicit:

```limceron
let port: u16 = 8080
let network_port = port.to_be()  // to big-endian
let host_port = network_port.to_le()  // back to little-endian
```

## Cross-Compilation

The compiler contains all platform backends in a single binary:

```bash
# Build from macOS for Linux ARM64
limceron build --target=linux-arm64

# Build from Linux for Windows
limceron build --target=windows-x86_64

# List all available targets
limceron targets
```

No cross-toolchain installation needed. The compiler generates native machine code
directly for each target.
