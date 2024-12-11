
# Linux内核内存保护机制：aslr和canary


## ASLR


ASLR技术，全称为Address space layout randomization（地址空间布局随机化），是现代通用操作系统基本都会配备的一个功能，其确保了每次实例化进程时内存排布都是不同的。


对于某些内存段，会附加随机的offset来防止缓冲区攻击等，这是OS层面的保护，当然也可以兼容硬件层面使用ECC bit进行合法性检测的冗余保护。


更确切的说，在Linux系统下的进程模型中，aslr对于内存排布的影响如下：


* 不变：代码段/BSS/全局数据区等
* 改变：加载的依赖库的代码位置（手动链接的，最典型的是glibc，e.g. 比如使用printf的时候重放会出现链接空间地址段错误，若禁用aslr则可以进行攻击），栈空间，堆空间等（后两者视aslr的不同层级，有可能不会附加，取决于内核版本）


进程地址空间排布（来自代码随想录，仅供参考）：
![](https://img2024.cnblogs.com/blog/3061928/202412/3061928-20241210170332840-1925834555.png)


在**GDB环境**下运行程序，aslr是默认关闭的，这也便于我们进行程序的调试。


这里举出来一个典型的实用案例：


我在工作的实际需求中需要建立一个虚拟化的沙盒，底层基座依赖了一个uni\-kernel的bsd系统，出于sandbox的snapshot迁移重放需求，需要手动关闭aslr机制，并附加硬件层面的内存保护。


开启/关闭aslr执行以下代码，开启为1，关闭为0：



```
sysctl kern.elf64.aslr.stack=0  
sysctl kern.elf64.aslr.pie_enable=0  
sysctl kern.elf64.aslr.enable=0  
sysctl kern.elf64c.aslr.stack=0  
sysctl kern.elf64c.aslr.pie_enable=0  
sysctl kern.elf64c.aslr.enable=0  
sysctl -a | grep aslr  

```

如果使用gcc编译器的时候，可以附加`-fPIE`的编译选项，以支持aslr机制。


## canary


我们的栈中通常有一个magic number，用来检测该块内存空间是否被其他意外修改。它有一个好听的名字：canary，金丝雀，美丽而又脆弱。


它通常被部署在栈顶返回地址附近的某个位置，确保该处空间没有被外部缓冲区溢出修改，虽然只是一个简单的机制，但是可以防止很多比较简单的攻击或者非恶意失误，是内存保护的第一道防线。


通常在函数被调用时生成，且该段对于用户态来说是严格不可读的，所以只能用fork/提权/劫持sys函数等暴力破拆的方式探测处理。


这个思想不仅限于内核场景，在通用需求下做数据校验的时候也可以使用，或者需求可靠的TCB场景时也可用。这种情况下就是在user space中进行自定义规则的校验了。


在gcc/clang中可以使用`-fno-stack-protector`编译选项来禁用canary，但如果你不明确知道自己在干什么，不要这么做！


 本博客参考[cmespeed楚门加速器](https://77yingba.com)。转载请注明出处！
