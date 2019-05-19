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
