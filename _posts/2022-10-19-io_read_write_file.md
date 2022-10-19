---
layout: post
title: 多进程/线程读写文件问题
date: 2022-10-19 16:02:33 +0800
categories: IO技术
tags: io 文件读写
author: Rxsi
---

* content
{:toc}

# 进程/线程打开文件
首先需要关注的是 linux 系统的文件系统结构，当我们使用`open()`函数打开一个文件时，底层会创建如下的文件结构：

![open_one_file.png](/images/io_read_write_file/open_one_file.png)

在每个进程的`task_struct`结构中，包含有文件描述符表，int 类型的`fd`就是这个表的下标位置。一个文件每被打开一次，系统就会为其创建一个`file`结构，用以标识连接信息，主要包含文件状态（可读、可写、可读写），当前文件的偏移量，引用计数，指向 inode 的指针等。使用`fork()`可以复制父进程的文件描述符，父子进程会共用`file`结构，`file`结构会增加引用计数。如下图所示：
<!--more-->

![fork_open_one_file.png](/images/io_read_write_file/fork_open_one_file.png)

如果是其他进程也使用`open()`打开了相同的文件，那么会创建一个新的`file`结构，而`inode`结构的引用计数会增加。如下图所示：

![fork_opne_two_file.png](/images/io_read_write_file/fork_opne_two_file.png)

在同进程下的多个线程是共享一个文件描述符的，因此这个结构是和单进程环境下是一致的。
# 文件操作API
我们常用的文件操作 API 分为两类，一类是 linux 平台的系统调用，另一类则是 C 语言库函数。这两类 API 的区别在于 C 语言库函数是带有缓冲区的，围绕`流（stream）`以**文件指针**操作，具有更高的执行效率，且会依据系统平台的不同而有不同的底层实现，因此具有可移植性。linux 系统调用顾名思义只能用于 linux 系统，不具有缓存区，借助的是**文件描述符**（fd）来进行文件访问。当在 linux 系统平台使用 C 语言库函数，则对应的底层实现就是 linux 系统调用。

文件指针本质上只是 C 语言平台的一层数据封装，其定义如下：
```cpp
  struct _iobuf {
    char *_ptr;
    int _cnt;
    char *_base;
    int _flag;
    int _file; // 在 linux 平台此处的值就等价于文件描述符，通过fileno(FILE *stream)获取
    int _charbuf;
    int _bufsiz;
    char *_tmpfname;
  };
```

## linux 系统调用
### creat

- 用途：该函数用以创建文件，当文件已经存在时则不会创建新文件
- 头文件：sys/types.h、sys/stat.h、fcntl.h
- 函数原型：int creat(const char *filename, mode_t mode);
- 参数：
   - const char *filename：文件名，需要包含完整的文件路径
   - mode_t mode：文件模式，见 open() 函数

一般比较少使用这个函数，因为`open()`函数已经可以完成相同的功能

### open

- 用途：打开文件
- 头文件：sys/types.h、sys/stat.h、fcntl.h
- 函数原型：int open( const char * pathname, int oflags, ... /*mode_t mode*/);
- 参数：
   - const char *pathname：文件路径+文件名
   - int oflags：文件模式，有以下几种选项：
      - O_RDONLY：只读模式
      - O_WRONLY：只写模式
      - O_RDWR：读写模式
      - O_EXEC：执行模式
      - O_SEARCH：搜索模式，这个模式一般用于校验搜索权限
            这五个模式只能选一种

      - O_APPEND：每次写时都添加到文件末尾，在多进程同时写时要设置这个模式
      - O_CLOEXEC：此时执行 exec 函数族则自动关闭该文件描述符
      - O_CREAT：此文件不存在则创建
      - O_EXCL：如果同时指定了`O_CREAT`属性且文件已经存在则创建失败，这形成原子性操作
      - O_NONBLOCK：设置为非阻塞模式
      - O_TRUNC：如果文件本身存在，则以只写或者读写方式打开则将长度截断为 0
   - mode_t mode：文件属性，当flags指定了`O_CREAT`标识，那么此处参数必填，常见有：
      - S_IRUSR：读权限，文件属主
      - S_IWUSR：写权限，文件属主
      - S_IXUSR：执行权限，文件属主
      - S_IRGRP：读权限，文件所属组
      - S_IWGRP：写权限，文件所属组
      - S_IXGRP：执行权限，文件所属组
      - S_IROTH：读权限，其它用户
      - S_IWOTH：写权限，其它用户
      - S_IXOTH：执行权限，其它用户

### close

- 用途：关闭文件
- 头文件：unistd.h
- 函数原型：int close(int filedes);
- 参数：
   - int filedes：文件句柄

当进程关闭之后，系统会自动关闭由这个进程打开的所有文件，不过严格的开发流程是需要主动调用`close()`函数的。此外一个进程可以打开的文件描述符是有上限的，可通过`ulimit -n`指令查看，默认值是**1024**。

### write

- 用途：写文件
- 头文件：unistd.h
- 函数原型：ssize_t write(int filedes, const void *buf, size_t nbytes);
- 参数：
   - int filedes：文件句柄
   - const void *buf：字节序列，一般使用 char buf[] = "xxx"
   - size_t nbytes：字节数，一般使用 sizeof(buf)

这个函数和`fwrite()`函数都具有原子性，但是多个进程同时对一个文件进行读写时是无法保证写入顺序的，而只有在同个进程内才能保证写入的顺序，一般应用于多进程时需要添加`O_APPEND`属性。只有在`pipe`、`fifo`管道下操作文件大小小于`PIPE_BUF`的数据才能保证在多进程写入的原子性。

### read

- 用途：读文件
- 头文件：unistd.h
- 函数原型：ssize_t read(int filedes, void *buf, size_t nbytes);
- 参数：
   - int filedes：文件句柄
   - void *buf：用户态的缓冲区，一般使用 char buf[N]
   - size_t nbytes：要读取的字节数，这个值一般等于小于N

这个函数和`fread()`函数都具有原子性，但是如果同时有多进程在进行写和读，那么有可能会出现进程正在读取的内容被其他进程修改的情况，因此一般需要加锁处理。

### lseek

- 用途：修改文件偏移量，这是唯一可以修改文件偏移量的函数
- 头文件：unistd.h
- 函数原型：off_t lseek(int filedes, off_t offset, int whence);
- 参数：
   - int filedes：文件句柄
   - off_t offset：偏移量
   - int whence：决定了offset参数的含义，常见值有：
      - SEEK_SET：将文件偏移量修改为offset值
      - SEEK_CUR：将当前文件偏移量加上offset值，offset可正可负
      - SEEK_END：将文件偏移量设置为文件长度加上offset值，offset可正可负

这个函数也可以用来获取当前的的偏移量，比如：off_t cur = lseek(fd, 0, SEEK_CUR);

### fcntl

- 用途：用来操作文件描述符的特性
- 头文件：fcntl.h
- 函数原型：int fcntl(int filedes, int cmd, .../* arg*/);
- 参数：
   - int filedes：文件句柄
   - int cmd：功能类型，常见值有：
      - F_DUPFD：复制一个现有的文件描述符
      - F_GETFD/F_SETFD：获取/设置文件描述符标记
      - F_GETFL/F_SETFL：获取/设置文件状态标记
      - F_GETLK/F_SETLK/F_SETLKW：获取/设置记录锁，F_SETLKW是阻塞模式
   - arg：有两种形式，当是设置文件属性时，此属性是 int 类型，如果是设置记录锁，则此处要使用 flock 结构体指针。

flock 结构体结构：
```cpp
struct flock {
  short int l_type;
  short int l_whence;
  off_t l_start;
  off_t l_len;
  pid_t l_pid;
};

/*
l_type: 此参数表示所得类型，其可能的取值包括以下三种:
	F_RDLCK : 读锁
	F_WRLCK : 写锁
	F_UNLCK : 解锁
l_start: 此参数锁区域的开始位置的偏移量
l_whence: 此参数决定锁开始的位置。其可选参数为:
	SEEK_SET: 当前位置为文件的开头
	SEEK_CUR: 当前位置为文件指针的位置
	SEEK_END: 当前位置为文件末尾
l_len: 锁定文件的长度，0代表锁定从l_start位置开始到最后所有的字节

若要锁定整个文件，通常的方法为将l_start设为0，l_whence设为SEEK_SET,l_len设为0．
*/
```
这个函数的通常用法是先通过`F_GETFL`获取当前的属性，然后再通过位运算添加新的属性值，最后再`F_SETFL`的方式设置新属性。如果设置成功会返回0，否则返回 INVALID_FD。
```cpp
int oldSocketFlag = fcntl(listenfd, F_GETFL, 0); // 旧属性
int newSocketFlag = oldSocketFlag | O_NONBLOCK; // 新属性
if (fcntl(listenfd, F_SETFL, newSocketFlag) == INVALID_FD)
{
	// 设置失败
}
//....
```

## C 语言库函数
### fopen

- 用途：打开文件
- 头文件：stdio.h
- 函数原型：FILE * fopen(const char * path, const char * mode);
- 参数：
   - const char *path：文件路径
   - const char *mode：操作文件方式，可选有：
      - "r"或"rb"：以只读方式打开文件，该文件必须存在。
      - "w"或"wb"：以只写方式打开文件，如果文件不存在会新建文件，并把文件长度截短为零，清空原有的内容
      - "a"或"ab"：以只写方式打开文件，如果文件不存在会新建文件，不会清空原内容，新内容会追加在文件尾。（a 是 append 的意思）
      - "r+"或"rb+"或"r+b"：以读+写的方式打开文件，该文件必须存在
      - "w+"或"wb+"或"w+b"：以读+写的方式打开文件，如果文件不存在会新建文件，并把文件长度截短为零，清空原有的内容
      - "a+"或"ab+"或"a+b"：以读+写的方式打开文件，如果文件不存在会新建文件，不会清空原内容，新内容会追加在文件尾。（a 是  append 的意思）

字母 b 表示文件时一个二进制文件而不是文本文件。（linux 下不区分二进制文件和文本文件，windows 下文本文件是以 \r\n 结尾而二进制文件是以 \n 结尾）
成功则返回 FILE 指针，否则返回 NULL 且会设置 errno 标识错误

### fclose

- 用途：关闭文件
- 头文件：stdio.h
- 函数原型：int fclose(FILE *stream);
- 参数：
   - FILE *stream：FILE 结构体指针，即文件指针

如果关闭成功则返回 0，否则返回 EOF

### fgetc

- 用途：获取下一个无符号字符，并把偏移量向前移动
- 头文件：stdio.h
- 函数原型：int fgetc(FILE *stream);
- 参数：
   - FILE *stream：FILE 结构体指针，即文件指针

该函数以无符号 char 强制转换为 int 的形式返回读取的字符，即每次读取一个字节，如果到达文件末尾或发生读错误，则返回 EOF。

### fputc

- 用途：向字节流写入一个无符号字符，偏移量向前移动
- 头文件：stdio.h
- 函数原型：int fputc(int char, FILE *stream);
- 参数：
   - int char：要被写入的字符，这里填入的是实际字符对应的 ASCII 码，比如写入 65，那么对应的是字符 A
   - FILE *stream：FILE 结构体指针，即文件指针

如果写入成功，则返回被写入的字符，而错误返回 EOF 并设置错误标识符

### fgets

- 用途：从字节流中读取一行数据（读取到换行符），最大不超过n字节，因此有可能会返回一个不完整的行（不推荐使用）
- 头文件：stdio.h
- 函数原型：char *fgets(char *str, int n, FILE *stream);
- 参数：
   - char *str：用户态的缓冲区
   - int n：要读取的字符数
   - FILE *stream：FILE 结构体指针，即文件指针

有三种返回情况：

1. 还没有读取到文件末尾，那么会返回和 str 相同的指针
2. 如果到达了文件末尾或者没有读取到任何字符，那么 str 包含了读取到的内容，但是返回空指针
3. 如果发生了错误，那么返回空指针

### fwrite

- 用途：将指定数据写入字节流
- 头文件：stdio.h
- 函数原型：size_t fwrite(const void *ptr, size_t size, size_t nmemb, FILE *stream);
- 参数：
   - const void *ptr：要被写入的元素数组的指针，貌似只能写入 char 数组
   - size_t size：要被写入的每个元素的大小，以字节位单位
   - size_t nmemb：写入元素的个数
   - FILE *stream：FILE 结构体指针，即文件指针

该函数具有原子性
```c
char a[] = "abcde";
fwrite(a, 1, sizeof(a), fd); // 一个字节，总共6个元素，包含一个空字符
```

### fread

- 用途：从字节流读取指定大小的数据
- 头文件：stdio.h
- 函数原型：size_t fread(void *ptr, size_t size, size_t nmemb, FILE *stream);
- 参数：
   - void *ptr：用户态的缓冲区
   - size_t size：每个元素的大小，单位字节
   - size_t nmemb：元素个数
   - FILE *stream：FILE 结构体指针，即文件指针

该函数具有原子性，如果成功则返回读取到的元素总数，如果该值与 nmemb 不相同，那么可能是发生了错误，或者到达了文件末尾。

### fseek

- 用途：设置偏移量
- 头文件：stdio.h
- 函数原型：int fseek(FILE *stream, long int offset, int whence);
- 参数：
   - FILE *stream：FILE 结构体指针，即文件指针
   - long int offset：偏移量
   - int whence：设置模式，可选值有：
      - SEEK_SET：将文件偏移量修改为 offset 值
      - SEEK_CUR：将当前文件偏移量加上 offset 值，offset 可正可负
      - SEEK_END：将文件偏移量设置为文件长度加上  offset 值，offset  可正可负

### rewind

- 用途：将偏移量设置到文件头
- 头文件：stdio.h
- 函数原型：void rewind(FILE *stream);
- 参数：
   - FILE *stream：FILE 结构体指针，即文件指针

### ftell

- 用途：返回当前的偏移量
- 头文件：stdio.h
- 函数原型：long int ftell(FILE *stream);
- 参数：
   - FILE *stream：FILE 结构体指针，即文件指针

### fileno

- 用途：将 FILE 结构体转换为对应的 fd 文件描述符
- 头文件：stdio.h
- 函数原型：int fileno(FILE *stream);
- 参数：
   - FILE *stream：FILE 结构体指针，即文件指针

当我们使用 C 语言库函数族时，如果要设置文件的属性，那么需要借助这个函数 

### flock

- 用途：对文件描述符添加文件建议锁
- 头文件：sys/file.h
- 函数原型：int flock(int fd, int operation);
- 参数：
   - int fd：文件描述符
   - int operation：锁类型，参数有：
      - LOCK_SH：设置一个共享锁
      - LOCK_EX：设置一个互斥锁
      - LOCK_UN：移除本进程添加的共享/互斥锁
      - LOCK_NB：设置为非阻塞模式，函数默认是阻塞式加锁的，使用该配置可以修改为非阻塞

返回 0 则代表成功，-1 则加锁失败，因此可以使用 while 循环判断加锁是否成功。需要注意的是如果传入的`fd`当前已经持有锁，则再次调用会成功，表现结果类似于可重入锁，但是只需要一次解锁

#### flock & fcntl
linux 系统实际支持两种文件锁的形式，一种是建议锁，另一种则是强制锁。所谓建议锁就是指每次操作前都需要主动去检测当前进程是否持有锁，如果当前进程加了锁，但是其他进程使用前没有进行加锁检测，那么仍是可以进行修改的，这个规则和我们平时编码上对于锁的使用方式是一致的。而强制锁则是直接由内核进行管理，即使其他进程未进行加锁检测，调用`read`、`write`、`open`等系统调用也会失败，但是这种锁不建议使用。

flock 添加的锁是一个建议锁，fcntl 则支持添加建议锁或强制锁，他们的区别在于作用的粒度不相同。flock 添加的建议锁是直接锁定整个文件，这使得不支持数据库系统的运行，因为某些系统不支持对部分文件加锁。fcntl 添加的建议锁称为记录锁，允许对一个文件的任意字节数添加区域性锁定。

这两个函数的另外一个区别在于，fcntl 添加的锁关联的是进程，通过`fork()`函数生成的子进程不会继承该锁，而`exec()`函数创建的新进程则可以继承这个锁，进程如果同时存在多个`fd`指向这个被加锁的文件，则只要任一个`fd`关闭则文件锁就会被释放。由于这个特性，同一个进程内的多个线程将会共享对同一个文件的锁，因此多线程环境下无法确保对同一区域的访问控制，下文代码有示例。

flock 添加的锁关联的是`fd`，因此使用`fork()`函数生成的子进程会继承该锁，而`exec()`函数创建的新进程也会继承该锁。由于这个特性，多进程环境下使用该锁需要注意避免对`fd`的拷贝（fork、dup等），而多线程则可以通过在不同线程创建指向同一个文件的`fd`来实现对文件的加锁访问控制。

# 多线程
## 多线程操作同一个文件指针/文件描述符
当一个进程中的多个线程对同个文件指针/文件描述符进行读、写、读+写时都能保证有序的进行，这是由于`fread()`、`fwrite()`、`read()`、`write()`等函数具有原子性。示例：
```cpp
void readFunc(FILE *stream)
{
    int i = 200;
    char buf[10];
    while (i--)
    {
        std::cout << "threadID: " << std::this_thread::get_id() << ", ";
        ssize_t len = fread(buf, 1, sizeof(buf), stream); // 这里每次都可以完整的读取一整行"aaaaaaaaa"，说明write和read是交替完成的
        std::cout << "fread len: " << len << std::endl;
        if (len != 0) std::cout << "data: " << buf << std::endl;
    }
}

void writeFunc(FILE *stream, char (*buf)[10]) // char buf[]、char *buf、char buf[11]都会被转换为指针丢失了数组特性，因此如果要保留数组特性那么需要使用数组指针 char (*buf)[]
{
    int i = 200;
    while (i--)
    {
        ssize_t len = fwrite(*buf, 1, sizeof(*buf), stream);
    }
}


int main()
{
    FILE *stream = fopen(FILEPATH, "r+");
    char buf1[] = "aaaaaaaaa";
    std::thread t1(writeFunc, stream, &buf1);
    std::thread t2(readFunc, stream);
    t1.join();
    t2.join();
    fclose(stream);
}
```

## 多线程操作指向同一个文件的不同文件指针/文件描述符
由于不同的文件指针有有不同的文件偏移量，因此这种情况下是无法保证结果正确性的，比如`write`操作会互相覆盖。这种情况下，就需要进行必要的同步操作，有两种方式：

1. 使用写追加属性
2. 使用文件建议锁

### 使用文件追加属性
文件追加属性实质上是将偏移量修改与文件写入组合成一个函数，使这两个操作具有原子性。在使用这种属性实现文件同步时，linux 平台系统调用和 C 语言库函数都可以达到文件同步目的，**这也是优先推荐使用的方式**。
```cpp
void writeFunc(FILE *stream, char (*buf)[10]) // char buf[]、char *buf、char buf[11]都会被转换为指针丢失了数组特性，因此如果要保留数组特性那么需要使用数组指针 char (*buf)[]
{
    int i = 200;
    while (i--)
    {
        ssize_t len = fwrite(*buf, 1, sizeof(*buf), stream);
    }
}


int main()
{
    FILE *stream1 = fopen(FILEPATH, "a+"); // 使用append模式，这样就不会互相覆盖了
    char buf1[] = "aaaaaaaaa";
    std::thread t1(writeFunc, stream1, &buf1);
    FILE *stream2 = fopen(FILEPATH, "a+");
    char buf2[] = "bbbbbbbbb";
    std::thread t2(writeFunc, stream2, &buf2);
    t1.join();
    t2.join();
    fclose(stream1);
    fclose(stream2);
}
```

### 使用文件建议锁
在使用锁的情况下，发现使用带缓存版本的`fwrite`、`fopen`、`fread`等 API，即使正确的加锁也无法保证文件写入的正确性，当线程发生切换时，另一个线程的写入偏移量大概率会和上一个线程写入偏移量重合，因此最终造成数据覆盖。**所以这种情况下只能使用不带缓存的 linux 平台的文件操作 API。**

这里使用的场景是多线程，由于`fcntl`添加的记录锁是关联到进程，因此这种情况下使用`fcntl`加锁是无法保证文件的访问控制性的，如以下代码：
```cpp
void writeFunc(int fd, char (*buf)[10]) // char buf[]、char *buf、char buf[11]都会被转换为指针丢失了数组特性，因此如果要保留数组特性那么需要使用数组指针 char (*buf)[]
{
    int i = 200;
    struct flock lock;
    memset(&lock, 0, sizeof(lock));
    lock.l_whence = SEEK_SET;
    lock.l_start = 0;
    lock.l_len = 0;
    while (i--)
    {
        lock.l_type = F_WRLCK;
        int ret = fcntl(fd, F_SETLKW, &lock); // 使用阻塞模式，如果本进程已经拥有锁那么是可以再次调用的，会替换掉已有的锁。
        if (ret != 0) std::cout << "lock fail" << std::endl;
        lseek(fd, 0, SEEK_END);// 移动到文件尾
        std::cout << "thredID: " << std::this_thread::get_id() << ", offset: " << lseek(fd, 0, SEEK_CUR) << std::endl;
        ssize_t len = write(fd, buf, sizeof(*buf)); // 由于记录锁是进程共享的，因此这里的写会互相覆盖
		// lock.l_type = F_UNLCK;
    	// fcntl(fd, F_SETLK, &lock); 这里只加锁不解锁，但是两个线程都还是可以进行写入，因为前面加的锁是整个进程共享的。
    }
}

int main()
{
    int fd1 = open(FILEPATH, O_WRONLY|O_TRUNC);
    char buf1[] = "aaaaaaaaa";
    std::thread t1(writeFunc, fd1, &buf1);
    int fd2 = open(FILEPATH, O_WRONLY|O_TRUNC);
    char buf2[] = "bbbbbbbbb";
    std::thread t2(writeFunc, fd2, &buf2);
    t1.join();
    t2.join();
    close(fd1);
    close(fd2);
}
```
为了避免写操作的互相覆盖，那么就需要使用`flock`进行加锁
```cpp
void writeFunc(int fd, char (*buf)[10]) // char buf[]、char *buf、char buf[11]都会被转换为指针丢失了数组特性，因此如果要保留数组特性那么需要使用数组指针 char (*buf)[]
{
    int i = 200;
    while (i--)
    {
        int ret = flock(fd, LOCK_EX); // 阻塞加锁
        if (ret != 0) std::cout << "lock fail" << std::endl;
        lseek(fd, 0, SEEK_END);// 移动到文件尾
        std::cout << "thredID: " << std::this_thread::get_id() << ", offset: " << lseek(fd, 0, SEEK_CUR) << std::endl;
        ssize_t len = write(fd, buf, sizeof(*buf)); // 按顺序交替写入成功
        flock(fd, LOCK_UN);
    }
}

int main()
{
    int fd1 = open(FILEPATH, O_WRONLY|O_TRUNC);
    char buf1[] = "aaaaaaaaa";
    std::thread t1(writeFunc, fd1, &buf1);
    int fd2 = open(FILEPATH, O_WRONLY|O_TRUNC);
    char buf2[] = "bbbbbbbbb";
    std::thread t2(writeFunc, fd2, &buf2);
    t1.join();
    t2.join();
    close(fd1);
    close(fd2);
}
```
验证写入成功：
```shell
rxsi@VM-20-9-debian:~/learncpp$ grep -c "aaaaaaaaa" ../file_test.txt 
200
rxsi@VM-20-9-debian:~/learncpp$ grep -c "bbbbbbbbb" ../file_test.txt 
200
```

# 多进程
## 多进程操作同一个文件指针/文件描述符
要使多进程能够操作同一个文件指针/文件描述符，可以使用`fork()`或者`dup()`对已有文件描述符进行复制，由于操作的是同一个文件指针/文件描述符，因此与多线程下操作同一个文件指针/文件描述符的情况一致。
```cpp
void readFunc(int fd)
{
    int i = 100;
    while (i--)
    {
        std::cout << "processID: " << getpid() << ", ";
        char buf[10];
        std::cout << "before ftell: " << lseek(fd, 0, SEEK_CUR) << ", ";
        size_t len = read(fd, buf, sizeof(buf)); // 交替顺序输出
        std::cout << "len: " << len << ", ";
        if (len == 0)
        {
            std::cout << "empty data" << ", ";
        }
        else
        {
            std::cout << "data: " << buf << ", "; 
        }
        std::cout << "after ftell: " << lseek(fd, 0, SEEK_CUR) << std::endl;
    }
}

int main()
{
    int fd = open(FILEPATH, O_RDONLY);
    pid_t pid = fork();
    if (pid == 0)
    {
        readFunc(fd);
    }
    else
    {
        readFunc(fd);
        int status;
        wait(&status);
    }
}
```
在测试的过程中发现如果使用文件指针以上面那种代码形式会出现读取异常，其中一个进程的读取偏移量会直接定位到文件尾部，不清楚具体原因：
```shell
flag: parent, processID: 9737, before ftell: 980, len: 10, data: aaaaaaaaa, after ftell: 990, fd: 3
flag: child, processID: 9738, before ftell: 4000, len: 0, empty data, after ftell: 4000, fd: 3
flag: parent, processID: 9737, before ftell: 990, len: 10, data: aaaaaaaaa, after ftell: 1000, fd: 3
flag: child, processID: 9738, before ftell: 4000, len: 0, empty data, after ftell: 4000, fd: 3
```
经测试发现需要把父进程的读操作提前到`fork()`函数之前才可以正常读取：
```cpp
int main()
{
    int fd = open(FILEPATH, O_RDONLY);
	readFunc(fd);
    pid_t pid = fork();
    if (pid == 0)
    {
        readFunc(fd);
    }
    else
    {
        int status;
        wait(&status);
    }
}
```

## 多进程操作指向同一个文件的不同文件指针/文件描述符
如果多进程同时打开了相同的文件，那么系统会为每个进程创建一个`file`结构体，因此不同进程的读取操作不会互相干扰，可以保证读取结果的正确性。

如果应用的场景是多进程读+写，那么由于每个`file`结构具有独立的文件偏移量，则可能出现读取过程中文件数据被其他进程修改而导致读取到错误数据的问题，如以下代码示例：
```cpp
void writeFunc(int fd, char (*buf)[6])
{
    ssize_t len = write(fd, *buf, sizeof(*buf));
}

// 假设当前读取缓存区不足以一次性读取所有的数据，因此分了三次进行读取
void readFunc(int fd)
{
    int i = 3;
    char buf[6];
    int step = 0;
    char temp[2];
    while (i--)
    {
        std::this_thread::sleep_for(std::chrono::seconds(1));
        size_t len = read(fd, temp, sizeof(temp));
        strcpy(buf+step, temp);
        step += 2;
    }
    std::cout << buf << std::endl; // 输出aabbb
}

int main()
{
    pid_t pid = fork();
    if (pid == 0) 
    {
        int fd = open(FILEPATH, O_WRONLY|O_TRUNC);
        char buf1[] = "aaaaa";
        writeFunc(fd, &buf1); // 先写入了aaaaaaaaa
        std::this_thread::sleep_for(std::chrono::seconds(2));
        lseek(fd, 0, SEEK_SET);
        char buf2[] = "bbbbb";
        writeFunc(fd, &buf2); // 再从头写入bbbbbbbbb
    }
    else
    {
        int fd = open("/home/rxsi/hello_world.txt", O_RDONLY);
        readFunc(fd);
        int status = 0;
        wait(&status);
    }
}
```
上面程序的输出结果印证了我们的结论，因此要实现安全的读取，就需要添加文件锁使读操作和写操作形成互斥。这里应用的场景是不同进程，因此`fcntl()`和`flock()`都是可以使用的。示例代码：
```cpp
void writeFunc(int fd, char (*buf)[6])
{
    struct flock lock;
    memset(&lock, 0, sizeof(lock));
    lock.l_whence = SEEK_SET;
    lock.l_start = 0;
    lock.l_len = 0;
    lock.l_type = F_WRLCK; // 这里加写锁
    int ret = fcntl(fd, F_SETLKW, &lock);
    ssize_t len = write(fd, *buf, sizeof(*buf));
    lock.l_type = F_UNLCK;
    fcntl(fd, F_SETLK, &lock); // 解锁
}

void readFunc(int fd)
{
    int i = 3;
    char buf[6];
    int step = 0;
    char temp[2];
    struct flock lock;
    memset(&lock, 0, sizeof(lock));
    lock.l_whence = SEEK_SET;
    lock.l_start = 0;
    lock.l_len = 0;
    while (i--)
    {
        std::this_thread::sleep_for(std::chrono::seconds(1));
        lock.l_type = F_RDLCK; // 这里加读锁就好了，如果有多个进程同时读是可以同时进行的
        int ret = fcntl(fd, F_SETLKW, &lock);
        size_t len = read(fd, temp, sizeof(temp));
        strcpy(buf+step, temp);
        step += 2;
    }
    lock.l_type = F_UNLCK;
    fcntl(fd, F_SETLK, &lock); // 读取完毕，解锁
    std::cout << buf << std::endl; // 输出：aaaaa
}
```
同样的道理，如果是多进程同时写，一种方式是使用`O_APPEND`属性，另外一种就是使用`建议锁`，总体上上面演示的代码结构相似，这里就不做额外的代码展示了。