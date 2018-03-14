# OPTEE
## untrusted application和trusted application之间的交互
相关文献V.C部分[BOOMERANG_Exploiting_the_Semantic_Gap_in_Trusted_Execution_Environments.pdf](/uploads/80fafd577c1ecd9f97f14017029d791f  /BOOMERANG_Exploiting_the_Semantic_Gap_in_Trusted_Execution_Environments.pdf)
### 关于untrusted application和untrusted os之间
和其他的实现方式一样，untrusted os提供一个driver /dev/tee0，用来使application和TAs进行交互，同时提供一个client library——libteec.so使得applications和driver之间的通信更加便捷。所有传输到TA的参数都是强类型的，这里有两种广泛的类型，指针类型和值类型(二者均可以输入到TA中，也可以从TA中输出)。每次call the secure world只可以支持四个参数，而且必须统一为严格的类型。  
untrusted application也使用不透明的指针(即shmids)来表示一块需要和TA进行共享的内存。为了传递指针参数，untrusted application需要和/dev/tee0通信来获取一块具体长度的memory，然后kernel driver会从特定的共享memory区域(即common-memory)中分配出这块memory，同时附带一个shmid，将其返回给client。  untrusted application可以利用这个shmid将分配出来的memory映射到自己的地址空间中，在这块映射的memory中，application可以向TA写入command，从TA读取response。这块共享的memory是non-secure和secure world都可以访问的。然而由于这块共享的memory是专用的，这减少了BOOMERANG漏洞的风险。  
### 关于untrusted os和trusted os之间
在接收到来自untrusted application的command时，untrusted os将首先执行所需指针的翻译(比如PTRSAN)，接下来，os将把所有的参数打包放到optee_msg_arg结构体中，同时将该结构体复制到在common-memory中的一块空闲区域中。最后，os将使用SMC指令执行world的切换，参数设置为之前保存optee_msg_arg结构体的memory区域的物理地址。
### 关于trusted os和trusted application之间
在OP-TEE中运行的TAs是在secure world中作为非特权的application，每一个都只运行在它自己的thread里，只有来自于non-secure world的请求才会驱动该thread运行。所有的特权操作必须通过执行syscall陷入到trusted os中执行(比如supervisor call指令(SVC))。对于每一个来自于non-secure world的传递给TA的memory参数，首先检查其物理地址，确定该地址在common-memory区域中，同时确定该物理地址区域是映射到处理request的thread中。更确切地说，trusted os将把传递的过来的物理地址作为参数，并用handling thread（即TA）存储空间内的相应虚拟地址更新它。所以当TA访问任何指针参数的时候，它可以访问它像正常的指针一样（即不需要任何额外的验证调用）。然而TA必须严格确定所有的参数的类型都是预期的，否则像type-confusion attack就可以用来利用TA或者trusted kernel。比如，如果一个指针可以伪装为一个值，绕过PTRSAN，那么shared memory region外的memory regions就可以传递到TA里，这就会导致BOOMERANG漏洞。
### 需要注意
这里TA是不能够访问non-secure world memory的，TA可以访问自己的memory和common-memory，访问os kernel memory需要内陷到kernel态。
### TEE中利用common-memory进行交互的图示(基于上述)
![optee-common-memory](/uploads/33167bc10c20933b075ace9ad6cbad82/optee-common-memory.jpg)  