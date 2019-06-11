# 文件I/O

输入输出是主存储器和外部设备之间拷贝数据的过程

* 输入操作是设备到内存
* 输出操作是内存到设备

## 高级I/O

* ANSI提供的标准I/O库
* 带缓冲区

## 低级I/O

* 系统调用I/O
* 不带缓冲区

## 错误处理

系统编程中错误通常通过函数返回值表示,通过特殊变量errno来描述,这个全局变量在`<errno.h>`头文件中,声明是`extern int errno;`,对应的错误处理函数是`perror`和`strerror`

```cpp
#include <errno.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>
int main(void)
{
    int ret;
    ret = close(10);
    if (ret == -1)
    {
        // perror格式固定
        perror("close error");
        // fprintf可以自定义输出格式
        fprintf(stderr, "close error with msg:%s\n",
        // 头文件string.h中函数strerror将errno转成错误文本
                strerror(errno));
    }
    return 0;
}
```

运行结果如下

```bash
$ ./a.out
close error: Bad file descriptor
close error with msg:Bad file descriptor
```

## FD文件描述符

* 对文件/设备的操作通过文件描述符
* 文件描述符由系统返回,是打开/新建文件时返回的非负整数
* 进程启动默认打开3个文件描述符
  * STDIN\_FILENO 0 // 标准输入
  * STDOUT\_FILENO 1 // 标准输出
  * STDERR\_FILENO 2 // 标准错误

ANSI C定义的是文件指针,系统调用的文件描述符是非负整数

| 系统调用 | ANSI C |
| :--- | :--- |
| STDIN\_FILENO | stdin |
| STDOUT\_FILENO | stdout |
| STDERR\_FILENO | stderror |

### 文件描述符和文件指针的相互转换

利用`fileno`将文件指针转换成文件描述符;利用`fdopen`将文件描述符转换成文件指针

```cpp
#include <stdio.h>
int main(void)
{
    int outfd = fileno(stdout);
    printf("fileno(stdout) = %d\n", outfd);
    FILE *fp = fdopen(outfd, "w+");
    fprintf(fp, "hello world\n");
    return 0;
}
```

程序运行结果如下

```bash
$ ./a.out
fileno(stdout) = 1
hello world
```

## 文件系统调用

* open获得访问文件的文件描述符
* close释放文件描述符
* create
* read
* write

### open

#### open不带权限参数

函数原型`int open(const char *path, int flags)`

* path 文件名称,可以包括绝对/相对路径
* flags 文件打开模式
* 执行成功返回文件描述符,失败返回-1

```cpp
#include <errno.h> // errno
#include <fcntl.h> // open
#include <stdio.h>
#include <stdlib.h>    // exit
#include <string.h>    // strerror
#include <sys/stat.h>  // open
#include <sys/types.h> // open
#include <unistd.h>    // I/O原语 read write close
#define ERR_EXIT(m)         \
    do                      \
    {                       \
        perror(m);          \
        exit(EXIT_FAILURE); \
    } while (0)
int main(void)
{
    int fd = open("test", O_RDONLY);
    if (fd == -1)
    {
        ERR_EXIT("open error");
    }
    printf("open succ\n");
    return 0;
}
```

执行结果如下

```bash
$ ./a.out
open error: No such file or directory
```

open第二参数定义在`fcntl.h`中

| 定义 | 值 | 操作 |
| :--- | :--- | :--- |
| O\_RDONLY | 0x0000 | 仅读 |
| O\_WRONLY | 0x0001 | 仅写 |
| O\_RDWR | 0x0002 | 可读写 |
| O\_ACCMODE | 0x0003 | 访问模式 |
| O\_APPEND | 0x0008 | append模式 |
| O\_CREAT | 0x0200 | 不存在则创建 |
| O\_EXCL | 0x0800 | 已存在则报错 |
| O\_TRUNC | 0x0400 | 清空文件 |

#### open带权限参数

函数原型 `int open(const char *path, int flags, mode_t mode)`

* path 文件名称,可以包括绝对/相对路径
* flags 文件打开模式
* mode 访问权限
* 执行成功返回文件描述符,失败返回-1

相较于第一种`open`,多了权限参数

```cpp
#include <errno.h> // errno
#include <fcntl.h> // open
#include <stdio.h>
#include <stdlib.h>    // exit
#include <string.h>    // strerror
#include <sys/stat.h>  // open
#include <sys/types.h> // open
#include <unistd.h>    // I/O原语 read write close
#define ERR_EXIT(m)         \
    do                      \
    {                       \
        perror(m);          \
        exit(EXIT_FAILURE); \
    } while (0)
int main(void)
{
    int fd = open("test", O_RDONLY | O_CREAT, 0666);
    if (fd == -1)
    {
        ERR_EXIT("open error");
    }
    printf("open succ\n");
    return 0;
}
```

执行结果如下

```bash
$ ./a.out
open succ
$ ls -l
total 48
-rwxr-xr-x  1 user  staff  8568 May 19 15:48 a.out
-rw-r--r--@ 1 user  staff   595 May 19 15:48 main.c
-rw-r--r--  1 user  staff     0 May 19 15:48 test
```

得到的文件权限为`-rw-r--r--`,对应0644,跟给定的0666不一致

##### umask

命令格式`umask [选项][掩码]`

最开始的权限由文件创建掩码决定,每次注册进入系统umask命令都被执行并自动设置掩码改变默认值,新的权限会把旧的覆盖

| 打开方式 | 值 | 描述 |
| :--- | :--- | :--- |
| S\_IRWXU | 0000700 | \[XSI\] RWX mask for owner |
| S\_IRUSR | 0000400 | \[XSI\] R for owner |
| S\_IWUSR | 0000200 | \[XSI\] W for owner |
| S\_IXUSR | 0000100 | \[XSI\] X for owner |
| S\_IRWXG | 0000070 | \[XSI\] RWX mask for group |
| S\_IRGRP | 0000040 | \[XSI\] R for group |
| S\_IWGRP | 0000020 | \[XSI\] W for group |
| S\_IXGRP | 0000010 | \[XSI\] X for group |
| S\_IRWXO | 0000007 | \[XSI\] RWX mask for other |
| S\_IROTH | 0000004 | \[XSI\] R for other |
| S\_IWOTH | 0000002 | \[XSI\] W for other |
| S\_IXOTH | 0000001 | \[XSI\] X for other |

### close

利用`close`释放打开的文件描述符,函数原型`int close(int fd);`

* fd要关闭的文件描述符
* 错误返回-1成功返回0

### create

与早期UNIX系统的兼容

### read

通过O\_RDONLY或O\_RDWR打开的文件描述符可通过`read`读取字节,函数原型`ssize_t read(int fd, void *buf, size_t count);`

* fd文件描述符
* buf存放读出数据的指针
* count复制到buf中的字节数
* 错误返回-1读文件结束返回0未结束返回从文件到缓冲区中的字节数

### write

通过O\_WRONLY或O\_RDWR打开的文件描述符可通过`write`写入字节,函数原型`ssize_t write(int fd, const void *buf, size_t count);`

* fd文件描述符
* buf存放取出数据的指针
* count需要写入文件的字节数
* 错误返回-1成功返回写入到文件的字节数

#### `size_t`和`ssize_t`

* `size_t`是标准c库中定义类型,32位系统定义为`unsigned int`64位系统定义为`long unsigned int`
* `ssize_t`执行读写操作的数据块大小,`typedef signed int size_t`

### 简单的cp命令

简单的拷贝

```cpp
#include <errno.h> // errno
#include <fcntl.h> // open
#include <stdio.h>
#include <stdlib.h>    // exit
#include <string.h>    // strerror
#include <sys/stat.h>  // open
#include <sys/types.h> // open
#include <unistd.h>    // I/O原语 read write close
#define ERR_EXIT(m)         \
    do                      \
    {                       \
        perror(m);          \
        exit(EXIT_FAILURE); \
    } while (0)
int main(int args, char *argv[])
{
    if (args != 3)
    {
        fprintf(stderr, "Usage %s src dest\n", argv[0], S_IRWXU);
        exit(EXIT_FAILURE);
    }
    int outfd = open(argv[2], O_WRONLY | O_CREAT | O_TRUNC);
    if (outfd == -1)
    {
        ERR_EXIT("open dest error");
    }
    int infd = open(argv[1], O_RDONLY);
    if (infd == -1)
    {
        ERR_EXIT("open src error");
    }
    char buf[1024];
    int nread;
    while ((nread = read(infd, buf, 1024)) > 0)
    {
        write(outfd, buf, nread);
    }
    close(outfd);
    close(infd);
    return 0;
}
```

执行结果如下

```bash
$ ./a.out main.c test
$ ls -l
total 56
-rw-r--r--@ 1 cengke  staff  4520 May  5 15:33 README.md
-rwxr-xr-x  1 cengke  staff  8880 May 19 20:19 a.out
-rw-r--r--@ 1 cengke  staff   999 May 19 20:10 main.c
-rw-------  1 cengke  staff   999 May 19 20:19 test
```

看到拷贝的文件和源文件一致

#### `read`和`write`差别

* `read`读过程中可能被某些信号中断
* `read`读指定字节数返回大于0时表示已经从文件读到缓冲区中
* `write`写指定字节数返回大于0时表示数据缓冲区已经拷贝到内核缓冲区,不代表同步到磁盘
* 利用`fsync`可以即时将数据同步到磁盘
* 利用给`open`设定一个参数`flags`值`0_SYNC`也可以即时将数据同步到磁盘,此时写文件会阻塞直到数据缓冲区写到物理磁盘

## 文件随机读写

Linux系统中有一个文件偏移的机制,将当前文件偏移值改变到有关位置,迫使`read`或`write`发生在这一位置

### lseek

通过指定相对于开始位置,当前位置或末尾位置的字节数来重定位curp,由`lseek`函数中指定的位置决定,声明是`off_t lseek(int fd, off_t offset, int base);`

* fd文件描述符
* offset偏移量
* base搜索起始位置
* 返回新的文件偏移值

base表示搜索的起始位置

| base | 文件位置 |
| :--- | :--- |
| SEEK\_SET | 0 set file offset to offset |
| SEEK\_CUR | 1 set file offset to current plus offset |
| SEEK\_END | 2 set file offset to EOF plus offset |

`lseek`对应于c语言的`fseek`返回当前偏移量

```cpp
#include <errno.h> // errno
#include <fcntl.h> // open
#include <stdio.h>
#include <stdlib.h>    // exit
#include <string.h>    // strerror
#include <sys/stat.h>  // open
#include <sys/types.h> // open
#include <unistd.h>    // I/O原语 read write close
#define ERR_EXIT(m)         \
    do                      \
    {                       \
        perror(m);          \
        exit(EXIT_FAILURE); \
    } while (0)
int main(int args, char *argv[])
{
    int fd;
    fd = open("test", O_RDONLY);
    if (fd == -1)
    {
        ERR_EXIT("open error");
    }
    char buf[1024] = {0};
    // 读取5个字节
    int ret = read(fd, buf, 5);
    if (ret == -1)
    {
        ERR_EXIT("read error");
    }
    printf("buf=%s\n", buf);
    // 由于读取5个字节偏移量应为5
    ret = lseek(fd, 0, SEEK_CUR);
    if (ret == -1)
    {
        ERR_EXIT("lseek error");
    }
    printf("current offset=%d\n", ret);
    return 0;
}
```

执行结果如下

```bash
$ ./a.out
buf=hello
current offset=5
```

利用`lseek`产生空洞文件,UNIX文件操作中文件位移量可以大于文件长度,下一次写入将根据文件位移量继续写入

```cpp
#include <errno.h> // errno
#include <fcntl.h> // open
#include <stdio.h>
#include <stdlib.h>    // exit
#include <string.h>    // strerror
#include <sys/stat.h>  // open
#include <sys/types.h> // open
#include <unistd.h>    // I/O原语 read write close
#define ERR_EXIT(m)         \
    do                      \
    {                       \
        perror(m);          \
        exit(EXIT_FAILURE); \
    } while (0)
int main(int args, char *argv[])
{
    int fd;
    fd = open("test", O_WRONLY | O_CREAT | O_TRUNC);
    if (fd == -1)
    {
        ERR_EXIT("open error");
    }
    int ret = write(fd, "hello", 5);
    if (ret == -1)
    {
        ERR_EXIT("read error");
    }
    ret = lseek(fd, 32, SEEK_CUR);
    if (ret == -1)
    {
        ERR_EXIT("lseek error");
    }
    write(fd, "world", 5);
    close(fd);
    return 0;
}
```

执行结果如下

```bash
$ ./a.out
$ ls -l
total 56
-rwxr-xr-x  1 cengke  staff  8648 May 19 21:13 a.out
-rw-r--r--@ 1 cengke  staff   839 May 19 21:12 main.c
-rw-r--r--  1 cengke  staff    42 May 19 21:13 test
$ od -c test
0000000    h   e   l   l   o  \0  \0  \0  \0  \0  \0  \0  \0  \0  \0  \0
0000020   \0  \0  \0  \0  \0  \0  \0  \0  \0  \0  \0  \0  \0  \0  \0  \0
0000040   \0  \0  \0  \0  \0   w   o   r   l   d
0000052
```

文件包含32字节空洞字符,没有被实际写入文件的所有字节由重复的`\0`表示,空洞是否占磁盘空间由文件系统\(file system\)决定

## 目录访问

### 打开目录

函数原型`DIR *opendir(char *pathname);`

* pathname 文件路径名
* 执行成功返回一个目录指针,失败返回0

### 访问指定目录中下一个链接的细节

函数原型`struct dirent *readdir(DIR *dirptr);`

* dirptr 目录指针
* 执行成功返回一个指向dirent结构的指针,包含指定目录中下一个链接的细节,没有更多链接时返回0

```cpp
struct dirent
{
    long d_ino;                  // inode number
    __off_t d_off;               // offset to this dirent
    unsigned short int d_reclen; // length of this d_name
    char d_name[256];            // filename (null-terminated)
};
```

dirent核心结构中d_name代表文件名

### 关闭一个已经打开的目录

函数原型`int closedir(DIR *dirptr);`

* 执行成功返回0,失败返回-1

### ls命令

```cpp
#include <dirent.h>
#include <errno.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#define ERR_EXIT(m)         \
    do                      \
    {                       \
        perror(m);          \
        exit(EXIT_FAILURE); \
    } while (0)
int main(int args, char *argv[])
{
    char *pathname = ".";
    if (args == 2)
    {
        pathname = argv[1];
    }
    DIR *dirptr = opendir(pathname);
    struct dirent *entry = readdir(dirptr);
    while ((entry = readdir(dirptr)) != NULL)
    {
        if ((strncmp(entry->d_name, ".", 1)) == 0)
        {
            continue;
        }
        printf("%s ", entry->d_name);
    }
    printf("\n");
    return 0;
}
```

### 创建新目录
