## 3.8 带外数据

### **📌 3.8 带外数据（Out of Band, OOB）解析**

#### **🔹 带外数据（OOB）概述**

- **作用**：用于紧急通知对方，数据优先级高于普通数据（也叫带内数据）。
    
- **特点**：
    
    1. **总是立即发送**，不受发送缓冲区中排队数据的影响。
        
    2. **独立于普通数据的传输**，但仍然使用相同的 TCP 连接。
        
    3. **不占用独立的传输信道**，而是通过 TCP 头部的 **URG（紧急）标志和紧急指针** 机制实现。
        
    4. **在应用层面**，像 `telnet`、`ssh` 等远程终端程序利用带外数据处理特殊输入。
        

#### **🔹 UDP 和 TCP 处理带外数据的方式**

- **UDP** 无法真正支持带外数据，它只能通过**数据报区分紧急和非紧急数据**。
    
- **TCP** 通过头部的 **URG 标志和紧急指针**，提供真正的带外数据支持。
    

### **🔹 TCP 发送带外数据**

假设一个进程已经建立了 TCP 连接，且在发送缓冲区写入了 **"abc"**：

1. **发送普通数据 "ab"**（没有特殊标记）。
    
2. **发送带外数据 "c"**：
    
    - **设置 TCP 头部的 `URG` 标志**，告诉对方这是带外数据。
        
    - **调整 TCP 头部的紧急指针**，指向**紧急数据的下一个字节**（即 `c` 之后的位置）。
        

📌 **图 3-10** 展示了 TCP 发送缓冲区的情况：

```
  |  a  |  b  |  c(OOB)  | 
            ↑
       紧急指针指向 c 之后的位置
```

- 只有 **最后 1 字节（`c`）被视为带外数据**，其余（`a`、`b`）仍是普通数据。
    
- 如果 `TCP` 发送多个数据段，每个段都会带 `URG` 标志，但紧急指针总指向相同的位置。
    


### **🔹 TCP 接收带外数据**

1. **检查 `URG` 标志和紧急指针** 来确定带外数据的位置。
    
2. **存入“带外缓冲区”**（只能存 **1 字节**）。
    
    - 如果应用程序未读取该字节，后续带外数据会覆盖它（旧的带外数据会被丢弃）。
        
3. **默认情况下，带外数据和普通数据一起存入 TCP 缓冲区**，但可以**通过 `SO_OOBINLINE` 选项**控制：
    
    - **未设置 `SO_OOBINLINE`**（默认）：带外数据单独存储，应用程序必须使用 `recv(..., MSG_OOB);` 读取。
        
    - **设置 `SO_OOBINLINE`**：带外数据会**像普通数据一样存入 TCP 缓冲区**，可以用普通 `recv()` 读取。
        

