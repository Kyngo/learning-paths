---
title: "File I/O & System Calls"
weight: 6
---

# File I/O & System Calls

C provides two layers of file I/O: the standard library (`<stdio.h>`) for portable, buffered I/O, and POSIX system calls for low-level, unbuffered access. Understanding both is essential for systems programming.

---

## Standard I/O (`<stdio.h>`)

### Opening and Closing Files

```c
FILE *fp = fopen("data.txt", "r");
if (!fp) {
    perror("fopen");  // prints: "fopen: No such file or directory"
    return 1;
}

// ... use fp ...

fclose(fp);
```

| Mode | Meaning |
|------|---------|
| `"r"` | Read (file must exist) |
| `"w"` | Write (truncates or creates) |
| `"a"` | Append (creates if needed) |
| `"r+"` | Read + write (file must exist) |
| `"w+"` | Read + write (truncates or creates) |
| `"rb"` | Read binary |
| `"wb"` | Write binary |

### Reading

```c
// Read character by character
int ch;
while ((ch = fgetc(fp)) != EOF) {
    putchar(ch);
}

// Read line by line
char line[256];
while (fgets(line, sizeof(line), fp)) {
    printf("%s", line);  // line includes '\n'
}

// Read formatted
int id;
char name[64];
fscanf(fp, "%d %63s", &id, name);

// Read binary
size_t n = fread(buffer, sizeof(int), count, fp);
```

### Writing

```c
fprintf(fp, "User %d: %s\n", id, name);
fputs("hello\n", fp);
fputc('A', fp);
fwrite(data, sizeof(int), count, fp);
```

### Standard Streams

| Stream | File Descriptor | Purpose |
|--------|----------------|---------|
| `stdin` | 0 | Standard input |
| `stdout` | 1 | Standard output (line-buffered) |
| `stderr` | 2 | Standard error (unbuffered) |

---

## POSIX I/O (Low-Level)

Direct system calls — no buffering, no formatting, works with file descriptors (integers):

```c
#include <fcntl.h>
#include <unistd.h>

int fd = open("data.txt", O_RDONLY);
if (fd == -1) {
    perror("open");
    return 1;
}

char buf[1024];
ssize_t n = read(fd, buf, sizeof(buf));
if (n == -1) {
    perror("read");
}

write(STDOUT_FILENO, buf, n);
close(fd);
```

| `open` Flag | Meaning |
|------------|---------|
| `O_RDONLY` | Read only |
| `O_WRONLY` | Write only |
| `O_RDWR` | Read + write |
| `O_CREAT` | Create if not exists |
| `O_TRUNC` | Truncate to zero length |
| `O_APPEND` | Append mode |
| `O_NONBLOCK` | Non-blocking I/O |

### stdio vs POSIX

| Feature | stdio (`FILE*`) | POSIX (`int fd`) |
|---------|----------------|-----------------|
| Buffering | Automatic (user-space) | None (kernel only) |
| Formatting | `fprintf`, `fscanf` | Manual |
| Portability | C standard (everywhere) | POSIX (Unix/Linux/macOS) |
| Performance | Good for sequential | Better for random access |
| Use case | Text files, formatted I/O | Sockets, pipes, devices |

---

## Error Handling: `errno`

System calls set `errno` on failure:

```c
#include <errno.h>
#include <string.h>

int fd = open("missing.txt", O_RDONLY);
if (fd == -1) {
    printf("Error %d: %s\n", errno, strerror(errno));
    // Error 2: No such file or directory
    perror("open");  // shorthand: prints "open: No such file or directory"
}
```

| errno Value | Name | Meaning |
|------------|------|---------|
| `ENOENT` | 2 | No such file or directory |
| `EACCES` | 13 | Permission denied |
| `EEXIST` | 17 | File exists |
| `EINTR` | 4 | Interrupted system call |
| `ENOMEM` | 12 | Out of memory |
| `EAGAIN` | 11 | Resource temporarily unavailable |

**Always check return values of system calls.** A return of -1 means failure; check `errno` for the reason.

---

## Signals

Signals are asynchronous notifications from the OS to a process:

```c
#include <signal.h>

volatile sig_atomic_t running = 1;

void handle_sigint(int sig) {
    (void)sig;
    running = 0;  // set flag — do NOT call printf or malloc here
}

int main(void) {
    signal(SIGINT, handle_sigint);  // or use sigaction for reliability

    while (running) {
        // main loop
    }
    printf("Shutting down gracefully\n");
    return 0;
}
```

| Signal | Number | Default | Meaning |
|--------|--------|---------|---------|
| `SIGINT` | 2 | Terminate | Ctrl+C |
| `SIGTERM` | 15 | Terminate | `kill` command |
| `SIGKILL` | 9 | Terminate | Cannot be caught |
| `SIGSEGV` | 11 | Core dump | Segmentation fault |
| `SIGPIPE` | 13 | Terminate | Write to broken pipe |
| `SIGCHLD` | 17 | Ignore | Child process terminated |

### Signal Handler Rules

Signal handlers must be **async-signal-safe**. You can:
- Set a `volatile sig_atomic_t` flag
- Call `_exit()`, `write()`, `signal()`

You **cannot** safely call: `printf`, `malloc`, `free`, `fopen`, or most library functions.

---

## Process Control

```c
#include <unistd.h>
#include <sys/wait.h>

pid_t pid = fork();  // create child process

if (pid == -1) {
    perror("fork");
} else if (pid == 0) {
    // Child process
    execvp("ls", (char *[]){"ls", "-la", NULL});  // replace with new program
    perror("exec");  // only reached if exec fails
    _exit(1);
} else {
    // Parent process
    int status;
    waitpid(pid, &status, 0);  // wait for child
    if (WIFEXITED(status)) {
        printf("Child exited with %d\n", WEXITSTATUS(status));
    }
}
```

---

## Key Takeaways

- `stdio.h` provides buffered, portable file I/O. POSIX calls provide low-level, unbuffered access.
- Always check return values. System calls return -1 on failure; `errno` tells you why.
- `perror()` and `strerror(errno)` are your friends for error messages.
- Signal handlers must be minimal — set a flag and return. Do not call `printf` or `malloc`.
- `fork()` + `exec()` is the Unix process creation model. The child is a copy of the parent until `exec` replaces it.
- Use `stdio` for text processing and formatted output. Use POSIX I/O for sockets, pipes, and devices.
