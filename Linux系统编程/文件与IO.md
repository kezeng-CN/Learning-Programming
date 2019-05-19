# 文件I/O

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
        perror("close error");
        fprintf(stderr, "close error with msg:%s\n",
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

关于代码中的两种报错方式，`perror`格式固定`fprintf`可以自定义输出格式,头文件<string.h>中函数`strerror`将`errno`转成错误文本

## I/O

输入/输出是主存和外部设备之间拷贝数据的过程
* 设备到内存是输入操作
* 内存到设备是输出操作

对于Linux的两种I/O类型

### 高级I/O

ANSI C提供的标准I/O库(带缓冲区)

### 低级I/O

系统调用I/O(不带缓冲区)

## 文件描述符

* 设备或文件的操作通过文件描述符进行
* 内核记录文件信息返回文件描述符(非负整数)

进程启动默认的3个文件描述符常量定义在`<unistd.h>`头文件中

1. STDIN_FILENO
2. STDOUT_FILENO
3. STDERR_FILENO
