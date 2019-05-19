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
    int fd = open("test.txt", O_RDONLY);
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

open第二参数定义在<fcntl.h>中

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



