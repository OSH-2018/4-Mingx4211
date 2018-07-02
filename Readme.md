# LAB04-Meltdown Attack

### 一、漏洞 & 程序原理

当今操作系统的核心安全特性之一是内存隔离。所谓内存隔离就是操作系统要确保用户应用程序不能访问彼此的内存，此外，它也要阻止用户应用程序对内核空间的访问。而通过Meltdown漏洞进行攻击，任意用户进程都可以突破操作系统对地址空间的隔离，从而获取内核空间的数据以及其他无权获取的数据，如用户的密码等。

漏洞利用现代计算机在CPU指令执行上所使用的乱序执行以及预测执行设计，来让用户态程序绕开内存隔离机制，来访问内核空间的数据。在现代CPU的设计中，为了提高性能，在执行一些需要等待操作结果的指令时，CPU并不进入STALL状态，而是采取乱序执行的方式，继续处理后续指令并调度该指令去空闲的执行单元去执行。而在这种情况下，被攻击的CPU可以运行未授权的进程从一个需要特权访问的地址上读出数据并加载到一个临时的寄存器中。虽然在发现异常后CPU会进行回滚，消除指令执行的效果，如对寄存器的影响等。然而即使消除了指令执行对寄存器等的影响，指令执行对Cache的影响却并不会消除。现代计算机系统采用Cache的方式来提高对数据的访问速度，即将需要读取的数据从内存复制到Cache中，以获得对数据更快的访问速度。在乱序执行非法的地址访问时，CPU会将目标地址的数据加载进Cache中。即使在CPU发现异常清楚运行结果后，加载进Cache中的数据仍会保留。因此，可以观察乱序执行对cache的影响，从而根据这些cache提供的侧信道信息来发起攻击。

具体的攻击方式是，利用乱序执行的非法地址访问来将目标地址的数据加载进Cache中，然后对内存进行遍历访问，被加载进Cache中的Page访问速度会显著快于其他Page，根据[Meltdown论文[1]](https://meltdownattack.com/meltdown.pdf)中所提供的数据，两者有着接近200个Cpu cycle的速度差距。通过对不同地址访问速度的观察，即可发现被加载的数据地址。从而推测出更多的数据。

### 二、攻击示例

所使用的POC来自于Github，POC的地址为(https://github.com/paboldin/meltdown-exploit/)。POC可以在用户态获取到`linux_proc_banner`这个内核地址上的数据.

POC会去打开/proc/version这个文件。/proc/version并不是一个文本文件，如果我们的程序尝试去读这个文件，内核会从`linux_proc_banner`这个地址动态地读取系统的信息并返回给程序。这就意味着这个地址上的数据会被加载到cache里。

![Snipaste_2018-07-02_19-06-17](https://github.com/OSH-2018/4-Mingx4211/blob/master/Snipaste_2018-07-02_19-06-17.png)

因此POC会打开/proc/version这个文件，并每次在进行Meltdown攻击前，都调用pread()一次，保证内核会将这个地址上的数据读到cache中。接下来POC就开始尝试读取`linux_proc_banner`这个内核地址上的数据了。因为一个地址的数据是由8个bit组成的，POC里每次都是只猜一个bit，因此一个地址要猜8次才行。

![Snipaste_2018-07-02_19-09-40](https://github.com/OSH-2018/4-Mingx4211/blob/master/Snipaste_2018-07-02_19-09-40.png)

猜数据的攻击也就是Meltdown攻击。首先POC将用户态target_array的偏移地址传给rbx。然后进行一段rept循环操作，保证数据已经读入cache。接着，将addr上的数据读到al中，随后根据bit的值（也就是第几位）对rax进行位移和AND 1操作，从而得到目标地址上第bit位的值（0或1）。最后根据是rax的值0还是1，对target_array进行写入操作。虽然在实际情况下，执行到`"movb (%[addr]), %%al\n\t"`这一行的时候，CPU就会报出异常了，但因为推测执行的原因，随后的代码已经被执行了，虽然推测执行的结果会回滚，但是对cache的操作却是不可逆的。

![Snipaste_2018-07-02_19-11-50](https://github.com/OSH-2018/4-Mingx4211/blob/master/Snipaste_2018-07-02_19-11-50.png)

因为读cache的速度要比读内存快很多，因此我们可以通过访问target_array里数据的速度来判断哪个数据被放到了cache里，从而就能知道目标地址上某一位的数据是什么了。

![Snipaste_2018-07-02_19-12-27](https://github.com/OSH-2018/4-Mingx4211/blob/master/Snipaste_2018-07-02_19-12-27.png)

程序最终运行结果如下

![2018-07-02 20-52-43屏幕截图](https://github.com/OSH-2018/4-Mingx4211/blob/master/2018-07-02 20-52-43屏幕截图.png)

成功读取到了`linux_proc_banner`地址上的数据.



实验环境: Ubuntu 16.04 LTS , 内核版本: x86_64 linux 4.13.0-45-generic 关闭PTI

![check](https://github.com/OSH-2018/4-Mingx4211/blob/master/check.png)