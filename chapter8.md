

### <center>第八章 hello的IO管理</center>

#### 8.1 Linux的IO设备管理方法

Linux下，操作系统管理IO设备通过设备驱动程序。驱动程序在设备可理解的硬件指令和内核使用的固定编程接口之间起转换作用。驱动程序层的存在有助于内核合理地保持设备独立性。

一个设备驱动程序可以在内核模式下，也可以在用户模式下访问，在用户模式下的访问要通过位于/dev目录下的特殊设备文件，印证了Linux的“一切皆文件”的思想。

大多数硬件设备在/dev下都有对应的一个设备文件（网络设备除外）。设备文件分为两个类型：块设备文件和字符设备文件。块设备文件一次读取或写入一块数据（一组字节），而字符设备文件每次读取或写入一个字节。

在Linux下一般不需要手动创建设备文件，linux下的udev系统将动态地创建和删除设备文件，守护进程udevd监听内核传来的有关设备状态变化的消息。

设备文件作为设备驱动程序的门户，负责数据在操作系统和设备间的中转。数据从应用程序或操作系统传递到设备文件，然后设备文件将它传递给设备驱动程序，驱动程序再将它发给物理设备。反向的数据通道也可以用，从物理设备通过设备驱动程序，再到设备文件，最后到达应用程序或其他设备。

由于设备已被模型化为文件，于是对设备的数据操作（读入写出）就可以使用操作系统的IO接口来完成。

#### 8.2 简述Unix IO接口及其函数

Unix IO接口，即Unix IO系统调用，是Unix系统提供的最低层的对于文件的读写函数。Unix IO接口只有五个函数：open（打开）、close（关闭）、read（读）、write（写）和lseek（定位）。通常，由于开销较大，我们不直接调用系统IO，而是调用Rio（健壮读写）或者C语言的标准IO进行读写。

```c
int open(char *filename, int flags, mode_t mode);
```

open函数，打开一个文件并返回该文件的描述符（非负整数），文件都是用描述符来标识的，若出错则返回-1。参数filename是文件路径字符串，flags表示进程以何种方式打开文件，通常使用的三个宏定义有O_RDONLY（只读模式）、O_WRONLY（只写模式）、O_RDWR（读写模式），三者必选其一。mode参数用于在创建新文件时指定文件的权限。

```c
int close(int fd);
```

close函数，关闭一个文件，内核释放文件打开时创建的数据结构，并恢复描述符到描述符池中，若关闭成功则返回0，关闭一个不存在的文件会返回-1。参数fd为文件描述符。

```c
ssize_t read(int fd, void *buf, size_t n);
```

read函数，从描述符为fd的当前文件位置拷贝至多n个字节到存储器位置buf。若描述符fd的当前位置为k，则会将当前位置增加到k+n，若k+n大于文件的字节数，则该操作触发一个EOF条件，返回0，否则返回读到的字节数（n）。出错返回-1。

```c
ssize_t write(int fd, const void *buf, size_t n);
```

write函数，从存储器位置buf拷贝至多n个字节到描述符fd的当前文件位置。若描述符fd的当前位置为k，则会将当前位置增加到k+n。如果操作成功则返回写入的字节数（n），如果失败则返回-1。

```c
off_t lseek(int fd, off_t offset, int whence);
```

lseek函数，将描述符为fd的文件的当前位置按照whence参数移动offset。当打开文件时，当前位置通常是在文件开头，若以附加的方式打开文件，则会处于文件结尾。参数whence有以下几个宏定义：SEEK_SET，参数offset 即为新的当前位置；SEEK_CUR，以目前的当前位置往后增加offset 个位移量；SEEK_END，将当前位置指向文件尾后再增加offset 个位移量。SEEK_CUR和SEEK_END时允许offset为负值。如果操作成功，返回当前位置，失败返回-1。

#### 8.3 printf的实现分析

printf的函数体如下：

```c
int printf(const char *fmt, ...)
{
	va_list args;
	int i;
	va_start(args, fmt);
	write(1,printbuf,i=vsprintf(printbuf, fmt, args));
	va_end(args);
	return i;
}
```

参数fmt即为格式字符串，“...”就是格式参数（不定）。printf函数读入格式参数，调用vsprintf生成了完整的字符串，并保存在buf字符串中。vsprintf返回的i为字符串的字节数。接着进行系统调用write，传入printbuf字符串和字节数i，最后返回。write函数的第一个参数1是当前进程的标准输出，是/dev/stdout（标准输出）的文件描述符。由于write函数是一个系统IO函数，所以调用应当通过陷阱来实现，汇编中write函数将一些参数传递到寄存器中，其中包括将系统调用号传入eax寄存器，接着使用INT指令引发了一个中断（陷阱）。在操作系统初始化时，start_kernel函数使用了trap_init函数完成了系统调用的初始化（陷阱），在该函数中将system_call和0x80绑定。此时的int 0x80一旦执行，将默认使用调用system_call处理该中断。system_call取出eax中的系统调用号，按照类型查找系统调用表，执行系统调用，向文件描述符为1的文件中写入printbuf字符串，文件描述符为1的文件是stdout的设备文件。当向stdout设备（通常是控制台）文件写入字符时，stdout的设备驱动程序（字符显示驱动程序）将不断读入字符，按照字符的ASCII码查找对应的字符的点阵（像素）信息和RGB信息，并存储在VRAM（显存）中。当显示芯片按照刷新频率进行下一次刷新时，将逐行扫描VRAM，通过视频信号线向显示器传输每个像素的RGB信息，显示器将其显示在屏幕上。

#### 8.4 getchar的实现分析

getchar函数是单个字符的输入函数。当程序执行getchar函数时，C代码调用read，执行一个int 0x80指令，引发一个陷阱，进程请求系统调用，system_call根据系统调用号和系统调用表，执行系统IO函数read。read函数的目标文件描述符为0，即/dev/stdin（标准输入），此时将等待用户从键盘输入。当用户按下键盘时，键盘驱动程序将按键扫描码转化为ascii码，并从/dev/stdin输出，read一个一个字符读取到一个字符串变量（缓冲区）中，并同时write到/dev/stdout中（同步显示），直到读取到回车符，表示输入结束，read函数返回，getchar函数将缓冲区的第一个字符返回。

#### 8.5 本章小结

输入输出赋予了一个程序以实用价值，如果一个程序没有输入也没有输出，即使正确执行也是没有意义的。输入输出设备的管理贯彻了Linux的“一切皆文件”的理念，简化了设备的管理，于是Linux便可以直接使用文件的系统调用管理外部设备的数据。