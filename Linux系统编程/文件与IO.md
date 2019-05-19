# 文件I/O

输入输出是主存储器和外部设备之间拷贝数据的过程

* 输入操作是设备到内存
* 输出操作是内存到设备

### 高级I/O

* ANSI提供的标准I/O库
* 带缓冲区

### 低级I/O

* 系统调用I/O
* 不带缓冲区

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

文件指针和文件描述符可以互相转换

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

几种获得访问文件的文件描述符的方法

### open

#### 函数原型`int open(const char *path, int flags)`

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

#### 函数原型 `int open(const char *path, int flags, mode_t mode)`

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

| S_IRWXU         |0000700         |[XSI] RWX mask for owner |
| S_IRUSR         |0000400         |[XSI] R for owner */
| S_IWUSR         |0000200         |[XSI] W for owner |
| S_IXUSR         |0000100         |[XSI] X for owner |
| S_IRWXG         |0000070         |[XSI] RWX mask for group |
| S_IRGRP         |0000040         |[XSI] R for group */
| S_IWGRP         |0000020         |[XSI] W for group |
| S_IXGRP         |0000010         |[XSI] X for group |
| S_IRWXO         |0000007         |[XSI] RWX mask for other |
| S_IROTH         |0000004         |[XSI] R for other */
| S_IWOTH         |0000002         |[XSI] W for other |
| S_IXOTH         |0000001         |[XSI] X for other |
| S_ISUID         |0004000         |[XSI] set user id on execution |
| S_ISGID         |0002000         |[XSI] set group id on execution |
| S_ISVTX         |0001000         |[XSI] directory restrcted delete |
| S_ISTXT         |S_ISVTX         |sticky bit: not supported |
| S_IREAD         |S_IRUSR         |backward compatability |
| S_IWRITE        |S_IWUSR         |backward compatability |
| S_IEXEC         |S_IXUSR         |backward compatability |

## 错误处理

系统变成中错误通常通过函数返回值表示,通过特殊变量errno来描述,这个全局变量在`<errno.h>`头文件中,声明是`extern int errno;`,对应的错误处理函数是`perror`和`strerror`

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



