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

## 
