
### 63.2 节 I/O 多路复用：select 和 poll 系统调用

I/O 多路复用是一种允许进程同时监控多个文件描述符（FD），并在其中一个或多个描述符变为就绪状态时获得通知的技术。这一技术避免了进程因阻塞在单个 I/O 操作上而导致的效率低下，尤其适用于需要同时处理多个客户端连接或多个输入源的场景（如网络服务器、终端交互程序等）。


#### 一、select 系统调用

select 是最早实现的 I/O 多路复用机制，其函数原型如下：
```c
#include <sys/select.h>
#include <sys/time.h>

int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
```

##### 1. 参数说明
- **nfds**：指定被监控的文件描述符集合中最大描述符值加 1。这是为了告知内核需要检查的描述符范围。
- **readfds、writefds、exceptfds**：分别为指向读就绪、写就绪和异常事件就绪的文件描述符集合的指针。这三个集合通过 `fd_set` 类型变量表示，可通过以下宏操作管理：
  - `FD_ZERO(fd_set *set)`：清空集合。
  - `FD_SET(int fd, fd_set *set)`：将描述符添加到集合。
  - `FD_CLR(int fd, fd_set *set)`：从集合中移除描述符。
  - `FD_ISSET(int fd, fd_set *set)`：检查描述符是否在集合中。
- **timeout**：指向 `struct timeval` 的指针，用于设置超时时间：
  - `struct timeval { long tv_sec; long tv_usec; }`：若为 `NULL`，则 select 会阻塞直到有描述符就绪；若为 `{0, 0}`，则立即返回，不阻塞。

##### 2. 返回值
- 成功时返回就绪的描述符总数。
- 若超时，返回 0。
- 错误时返回 -1，`errno` 可能为 `EBADF`（无效描述符）、`EINTR`（被信号中断）等。

##### 3. 工作原理
select 会阻塞进程，直到以下情况之一发生：
- 任一集合中的描述符变为就绪。
- 超时时间到达。
- 进程被信号中断。

就绪状态的判定规则：
- **读就绪**：描述符可读（如套接字接收缓冲区非空、管道有数据等）。
- **写就绪**：描述符可写（如套接字发送缓冲区有空间、管道未满等）。
- **异常就绪**：发生异常事件（如带外数据到达套接字）。


#### 二、poll 系统调用

poll 是 select 的改进版本，提供了更灵活的接口，函数原型如下：
```c
#include <poll.h>

int poll(struct pollfd fds[], nfds_t nfds, int timeout);

struct pollfd {
    int fd;         /* 文件描述符 */
    short events;   /* 等待的事件 */
    short revents;  /* 实际发生的事件 */
};
```

##### 1. 参数说明
- **fds**：指向 `pollfd` 结构体数组的指针，每个元素指定一个描述符及其关注的事件。
- **nfds**：数组中的元素数量。
- **timeout**：超时时间（毫秒）：
  - `-1`：永久阻塞。
  - `0`：立即返回。
  - 正数：等待指定毫秒数。

##### 2. `pollfd.events` 事件标志
- `POLLIN`：可读事件。
- `POLLOUT`：可写事件。
- `POLLERR`：错误事件。
- `POLLHUP`：挂起事件（如管道另一端关闭）。
- `POLLNVAL`：无效描述符（fd 未打开）。

##### 3. 返回值
- 成功时返回就绪的描述符数量。
- 超时返回 0。
- 错误返回 -1，`errno` 可能为 `EINTR`（被信号中断）、`EINVAL`（无效参数）等。

##### 4. 工作原理
poll 遍历 `fds` 数组，检查每个描述符的状态。与 select 不同：
- 无需手动管理描述符集合，直接通过 `pollfd` 结构体指定事件。
- 没有描述符数量的硬编码限制（仅受限于系统资源）。


#### 三、select 与 poll 的对比

| **特性**      | **select**                  | **poll**            |
| ----------- | --------------------------- | ------------------- |
| **描述符管理**   | 通过 `fd_set` 位掩码管理，跨平台需手动处理  | 结构体数组管理，更直观         |
| **最大描述符限制** | 通常受限于 `FD_SETSIZE`（默认 1024） | 仅受限于系统资源（如内存）       |
| **性能**      | 线性扫描所有描述符，大量描述符时效率低         | 同样线性扫描，但结构更紧凑       |
| **跨平台兼容性**  | 广泛支持，包括非 UNIX 系统            | 主要在 UNIX 系统中支持      |
| **就绪状态获取**  | 需手动重置 `fd_set` 集合           | 直接通过 `revents` 字段获取 |


#### 四、I/O 多路复用的应用场景

1. **网络服务器**：同时处理多个客户端连接，避免为每个连接创建单独进程/线程。
2. **终端交互程序**：同时监控键盘输入和其他 I/O 源（如网络数据）。
3. **文件服务器**：监控多个文件描述符的读写事件，优化资源利用。

##### 代码示例：使用 select 监控标准输入和套接字
```c
#include <sys/select.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

int main() {
    fd_set read_fds;
    struct timeval timeout;
    int stdin_fd = STDIN_FILENO;
    int socket_fd = /* 假设已创建的套接字描述符 */;
    char buffer[1024];

    while (1) {
        // 初始化描述符集合
        FD_ZERO(&read_fds);
        FD_SET(stdin_fd, &read_fds);
        FD_SET(socket_fd, &read_fds);

        // 设置超时时间（10 秒）
        timeout.tv_sec = 10;
        timeout.tv_usec = 0;

        // 调用 select
        int ready = select(socket_fd + 1, &read_fds, NULL, NULL, &timeout);
        if (ready == -1) {
            perror("select error");
            exit(EXIT_FAILURE);
        } else if (ready == 0) {
            printf("Timeout occurred\n");
            continue;
        }

        // 检查就绪描述符
        if (FD_ISSET(stdin_fd, &read_fds)) {
            // 处理标准输入
            fgets(buffer, sizeof(buffer), stdin);
            printf("Read from stdin: %s", buffer);
        }

        if (FD_ISSET(socket_fd, &read_fds)) {
            // 处理套接字数据
            int n = read(socket_fd, buffer, sizeof(buffer));
            if (n > 0) {
                printf("Read %d bytes from socket\n", n);
            }
        }
    }

    return 0;
}
```


#### 五、注意事项

1. **性能瓶颈**：select 和 poll 均采用线性扫描方式检查描述符，当监控大量描述符时，效率会显著下降（O(n) 复杂度）。
2. **描述符重置**：select 每次调用后会修改 `fd_set` 集合，需重新初始化；poll 则通过 `revents` 直接返回结果。
3. **超时处理**：select 的 `timeout` 参数在调用后会被修改，若需重复使用，需重新赋值。
4. **可移植性**：select 在 POSIX 和 SUSv3 中标准化，poll 则主要在 UNIX 系统中支持，部分嵌入式系统可能不支持。


#### 六、文档引用

- 关于 select 和 poll 的详细参数说明及错误处理，可参考文档中第 63 章相关内容（如至段）。
- 文档中提到，select 和 poll 均属于 POSIX 标准接口，但在处理大量描述符时存在性能限制，后续章节可能介绍更高效的 I/O 多路复用机制（如 epoll，在 Linux 中为专有扩展，见段）。