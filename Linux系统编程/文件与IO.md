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

Linux系统编程中错误通常通过函数返回值表示, 通过特殊变量errno来描述, 这个全局变量在 `<errno.h>` 头文件中, 声明是 `extern int errno;` , 对应的错误处理函数是 `perror` 和 `strerror`

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
    return EXIT_SUCCESS;
}
```

运行结果如下

```bash
$ ./a.out
close error: Bad file descriptor
close error with msg: Bad file descriptor
```

## FD文件描述符

* 对文件/设备的操作通过文件描述符
* 文件描述符由系统返回, 是打开/新建文件时返回的非负整数
* 进程启动默认打开3个文件描述符

  * STDIN_FILENO 0  // 标准输入
  * STDOUT_FILENO 1 // 标准输出
  * STDERR_FILENO 2 // 标准错误

ANSI C定义的是文件指针, 系统调用的文件描述符是非负整数

| SYSTEM CALL   | ANSI C   |
| :------------ | :------- |
| STDIN_FILENO  | stdin    |
| STDOUT_FILENO | stdout   |
| STDERR_FILENO | stderror |

### 文件描述符和文件指针的相互转换

利用 `fileno` 将文件指针转换成文件描述符; 利用 `fdopen` 将文件描述符转换成文件指针

```cpp
#include <stdio.h>
int main(void)
{
    int outfd = fileno(stdout);
    printf("fileno(stdout) = %d\n", outfd);
    FILE *fp = fdopen(outfd, "w+");
    fprintf(fp, "hello world\n");
    return EXIT_SUCCESS;
}
```

运行结果如下

```bash
$ ./a.out
fileno(stdout) = 1
hello world
```

## 文件I/O函数

* open
* close
* create
* read
* write

### open

#### open不带权限参数

函数原型 `int open(const char *path, int flags)`

* path 文件名称, 可以包括绝对/相对路径
* flags 文件打开模式
* 执行成功返回文件描述符, 失败返回-1

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
    return EXIT_SUCCESS;
}
```

执行结果如下

```bash
$ ./a.out
open error: No such file or directory
```

open第二参数定义在 `fcntl.h` 中

| DEFINE    | VALUE  | OPERATOR            |
| :-------- | :----- | :------------------ |
| O_RDONLY  | 0x0000 | read only           |
| O_WRONLY  | 0x0001 | write only          |
| O_RDWR    | 0x0002 | read and write      |
| O_ACCMODE | 0x0003 | access mode         |
| O_APPEND  | 0x0008 | append mode         |
| O_CREAT   | 0x0200 | creat if not exists |
| O_EXCL    | 0x0800 | error if exists     |
| O_TRUNC   | 0x0400 | truncate            |

#### open带权限参数

函数原型 `int open(const char *path, int flags, mode_t mode)`

* path 文件名称, 可以包括绝对/相对路径
* flags 文件打开模式
* mode 访问权限
* 执行成功返回文件描述符, 失败返回-1

相较于第一种 `open` , 多了权限参数

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
    return EXIT_SUCCESS;
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

得到的文件权限为 `-rw-r--r--` , 对应0644, 跟给定的0666不一致

##### umask

命令格式 `umask [选项][掩码]`

最开始的权限由文件创建掩码决定, 每次注册进入系统umask命令都被执行并自动设置掩码改变默认值, 新的权限会把旧的覆盖

| OPEN MODE | VALUE   | DESCRIPTION                |
| :-------- | :------ | :------------------------- |
| S_IRWXU   | 0000700 | \[XSI\] RWX mask for owner |
| S_IRUSR   | 0000400 | \[XSI\] R for owner        |
| S_IWUSR   | 0000200 | \[XSI\] W for owner        |
| S_IXUSR   | 0000100 | \[XSI\] X for owner        |
| S_IRWXG   | 0000070 | \[XSI\] RWX mask for group |
| S_IRGRP   | 0000040 | \[XSI\] R for group        |
| S_IWGRP   | 0000020 | \[XSI\] W for group        |
| S_IXGRP   | 0000010 | \[XSI\] X for group        |
| S_IRWXO   | 0000007 | \[XSI\] RWX mask for other |
| S_IROTH   | 0000004 | \[XSI\] R for other        |
| S_IWOTH   | 0000002 | \[XSI\] W for other        |
| S_IXOTH   | 0000001 | \[XSI\] X for other        |

### close

利用 `close` 释放打开的文件描述符, 函数原型 `int close(int fd);`

* fd 要关闭的文件描述符
* 执行成功返回0, 失败返回-1

### create

函数原型 `int create(const char *path, mode_t mode)`

* path 文件名称,可以包括绝对/相对路径
* mode 访问权限
* 执行成功返回文件描述符, 失败返回-1

与早期UNIX系统的兼容,等同于`open(const char *path, O_WRONLY | O_CREAT | O_TRUNC);`

### read

通过O_RDONLY或O_RDWR打开的文件描述符可通过 `read` 读取字节, 函数原型 `ssize_t read(int fd, void *buf, size_t count);`

* fd 文件描述符
* buf 存放读出数据的指针
* count 复制到buf中的字节数
* 错误返回-1读文件结束返回0未结束返回从文件到缓冲区中的字节数

### write

通过O_WRONLY或O_RDWR打开的文件描述符可通过 `write` 写入字节, 函数原型 `ssize_t write(int fd, const void *buf, size_t count);`

* fd 文件描述符
* buf 存放取出数据的指针
* count 需要写入文件的字节数
* 执行成功返回写入到文件的字节数, 失败返回-1

#### `size_t` 和 `ssize_t`

* `size_t` 是标准c库中定义类型, 32位系统定义为 `unsigned int` 64位系统定义为 `long unsigned int`
* `ssize_t` 执行读写操作的数据块大小, `typedef signed int size_t`

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
        fprintf(stderr, "Usage %s src dest\n", argv[0]);
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
    return EXIT_SUCCESS;
}
```

执行结果如下

```bash
$ ls -l
total 32
-rwxr-xr-x  1 zengke  staff  8880 Aug 10 19:53 a.out
-rw-r--r--@ 1 zengke  staff   991 Aug 10 19:53 cp.c
$ ./a.out cp.c test
$ ls -l
total 40
-rwxr-xr-x  1 zengke  staff  8880 Aug 10 19:53 a.out
-rw-r--r--@ 1 zengke  staff   991 Aug 10 19:53 cp.c
-r-x------  1 zengke  staff   991 Aug 10 19:53 test
$
```

看到拷贝的文件和源文件一致

#### `read` 和 `write` 差别

* `read` 读过程中可能被某些信号中断
* `read` 读指定字节数返回大于0时表示已经从文件读到缓冲区中
* `write` 写指定字节数返回大于0时表示数据缓冲区已经拷贝到内核缓冲区, 不代表同步到磁盘
* 利用 `fsync` 可以即时将数据同步到磁盘
* 利用给 `open` 设定一个参数 `flags` 值 `0_SYNC` 也可以即时将数据同步到磁盘, 此时写文件会阻塞直到数据缓冲区写到物理磁盘

## 文件定位

### lseek

通过指定相对于开始位置, 当前位置或末尾位置的字节数来重定位curp, 由 `lseek` 函数中指定的位置决定, 函数原型 `off_t lseek(int fd, off_t offset, int base);`

* fd 文件描述符
* offset 偏移量
* base 搜索起始位置
* 返回新的文件偏移值

base表示搜索的起始位置

| BASE     | POS                                      |
| :------- | :--------------------------------------- |
| SEEK_SET | 0 set file offset to offset              |
| SEEK_CUR | 1 set file offset to current plus offset |
| SEEK_END | 2 set file offset to EOF plus offset     |

 `lseek` 对应于c语言的 `fseek` 返回当前偏移量

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
    return EXIT_SUCCESS;
}
```

执行结果如下

```bash
$ ./a.out
buf=hello
current offset=5
```

利用 `lseek` 产生空洞文件, UNIX文件操作中文件位移量可以大于文件长度, 下一次写入将根据文件位移量继续写入

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
    return EXIT_SUCCESS;
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

文件包含32字节空洞字符, 没有被实际写入文件的所有字节由重复的 `\0` 表示, 空洞是否占磁盘空间由文件系统\(file system\)决定

## 目录操作

* opendir
* closedir
* mkdir
* rmdir

### 打开目录

函数原型 `DIR *opendir(char *pathname);`

* pathname 文件路径名
* 执行成功返回一个目录指针, 失败返回0

### 访问指定目录中下一个链接的细节

函数原型 `struct dirent *readdir(DIR *dirptr);`

* dirptr 目录指针
* 执行成功返回一个指向dirent结构的指针, 包含指定目录中下一个链接的细节, 没有更多链接时返回0

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

函数原型 `int closedir(DIR *dirptr);`

* 执行成功返回0, 失败返回-1

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
    return EXIT_SUCCESS;
}
```

### mkdir

函数声明 `int mkdir(char *pathname, mode_t mode);`

* pathname 创建的文件路径名
* mode 创建访问权限
* 执行成功返回0, 失败返回-1

### rmdir

函数声明 `int rmdir(char *pathname);`

* pathname 删除的文件路径名
* 执行成功返回0, 失败返回-1

## 权限控制

* chmod
* fchmod
* chown
* fchown

### chmod

函数声明 `int chmod(char *pathname, mode_t mode);`

* pathname 文件路径名
* mode 访问权限
* 执行成功返回0, 失败返回-1

### fchmod

函数声明 `int fchmod(int fd, mode_t mode);`

* fd 文件描述符
* mode 访问权限
* 执行成功返回0, 失败返回-1

### chown

函数声明 `int chown(int fd, uid_t owner, gid_t group);`

* pathname 文件路径名
* owner 所有者识别号
* group 用户组识别号
* 执行成功返回0, 失败返回-1

### fchown

函数声明 `int fchown(int fd, uid_t owner, gid_t group);`

* fd 文件描述符
* owner 所有者识别号
* group 用户组识别号
* 执行成功返回0, 失败返回-1

## 文件属性

### stat

函数声明 `int stat(const char *path, struct stat *buf);`

关于stat结构如下

```cpp
struct stat {
    dev_t     st_dev;     /* ID of device containing file文件保存在磁盘上的设备号,包含主设备号和次设备号,它是16位的整数,高八位为主设备号,低八位为次设备号 */
    ino_t     st_ino;     /* inode number */
    mode_t    st_mode;    /* protection文件的权限信息 */
    nlink_t   st_nlink;   /* number of hard links 文件的硬连接数*/
    uid_t     st_uid;     /* user ID of owner */
    gid_t     st_gid;     /* group ID of owner */
    dev_t     st_rdev;    /* device ID (if special file) 如果文件是设备文件,所对应的设备ID*/
    off_t     st_size;    /* total size, in bytes */
    blksize_t st_blksize; /* blocksize for file system I/O 系统当中每个块的大小 */
    blkcnt_t  st_blocks;  /* number of 512B blocks allocated 块数目 */
    time_t    st_atime;   /* time of last access 文件总共访问的次数*/
    time_t    st_mtime;   /* time of last modification 最后的修改时间*/
    time_t    st_ctime;   /* time of last status change 最后状态改变的时间,比如说改变了文件权限,并未改变文件的内容*/
};
```

#### `struct stat`结构体中的`st_dev`记录主设备号和次设备号、`st_ino`记录文件节点

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <unistd.h>

#define ERR_EXIT(m)         \
    do                      \
    {                       \
        perror(m);          \
        exit(EXIT_FAILURE); \
    } while (0)

#define MAJOR(m) (int)((unsigned short)m >> 8)
#define MINOR(m) (int)((unsigned short)m && 0xFF)

int main(int argc, char *argv[])
{
    if (argc != 2)
    {
        fprintf(stderr, "usage: %s filename pathname\n", argv[0]);
        exit(EXIT_FAILURE);
    }

    struct stat buf;

    if (stat(argv[1], &buf) == -1)
    {
        return EXIT_FAILURE;
    }

    printf("MAJOR DEV : %d\n", MAJOR(buf.st_dev));
    printf("MINOR DEV : %d\n", MINOR(buf.st_dev));

    printf("INODE : %d\n", (int)buf.st_ino); //需转为int

    return EXIT_SUCCESS;
}
```

执行结果如下

```bash
$ df test_st_dev.c
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1       7.9G  3.7G  3.8G  49% /
$ ls -l /dev/sda1
brw-rw---- 1 root disk 8, 1  8月 18 18:44 /dev/sda1
$ ls -li test_st_dev.c
331789 -rw-r--r-- 1 zeng zeng 806  8月 18 19:18 test_st_dev.c
$ ./a.out test_st_dev.c
MAJOR DEV : 8
MINOR DEV : 1
INODE : 331789
```

`ls -l /dev/sda1`命令结果中`brw-rw---- 1 root disk 8, 1  8月 18 18:44 /dev/sda1`的`8, 1`代表主设备号为8,次设备号为1

`ls -li test_st_dev.c` 命令结果中`331789 -rw-r--r-- 1 zeng zeng 806  8月 18 19:18 test_st_dev.c`的`331789`代表文件indoe节点数为331789

主设备号包含同一类型的设备(决定了用何种驱动程序来访问设备),次设备号用来区分同一设备中的不同的分区

#### `struct stat`结构体中的`st_mode`记录文件属性

`st_mode`一般是无符号整型,其中低16位定义如下

![st_mode说明](/Linux系统编程/st_mode说明.png)

`st_mode`包含了`ls -l`中需要的信息

```cpp
#include <dirent.h>
#include <grp.h>
#include <pwd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <time.h>
#include <unistd.h>

#define ERR_EXIT(m)         \
    do                      \
    {                       \
        perror(m);          \
        exit(EXIT_FAILURE); \
    } while (0)

#define MAJOR(m) (int)((unsigned short)m >> 8)
#define MINOR(m) (int)((unsigned short)m && 0xFF)

void file_prem(struct stat *, char *);
void file_prem(struct stat *buf, char *perm)
{
    // 确定文件权限
    if (buf->st_mode & S_IRUSR) // read permission, owner
        perm[1] = 'r';
    if (buf->st_mode & S_IWUSR) // write permission, owner
        perm[2] = 'w';
    if (buf->st_mode & S_IXUSR) // execute/search permission, owner
        perm[3] = 'x';
    if (buf->st_mode & S_IRGRP) // read permission, group
        perm[4] = 'r';
    if (buf->st_mode & S_IWGRP) // write permission, group
        perm[5] = 'w';
    if (buf->st_mode & S_IXGRP) // execute/search permission, group
        perm[6] = 'x';
    if (buf->st_mode & S_IROTH) // read permission, other
        perm[7] = 'r';
    if (buf->st_mode & S_IWOTH) // write permission, other
        perm[8] = 'w';
    if (buf->st_mode & S_IXOTH) // execute/search permission, other
        perm[9] = 'x';
    return;
}

void file_type(struct stat *, char *);
void file_type(struct stat *buf, char *type)
{
    // 确定文件类型
    switch (buf->st_mode & S_IFMT) //type of file
    {
    case S_IFIFO:
        type[0] = 'p';
        break;
    case S_IFCHR:
        type[0] = 'c';
        break;
    case S_IFDIR:
        type[0] = 'd';
        break;
    case S_IFBLK:
        type[0] = 'b';
        break;
    case S_IFREG:
        type[0] = '-';
        break;
    case S_IFLNK:
        type[0] = 'l';
        break;
    case S_IFSOCK:
        type[0] = 's';
        break;
    case S_IFWHT:
        type[0] = '?';
        break;
    default:
        break;
    }
    return;
}

void file_list(const struct dirent *);
void file_list(const struct dirent *entry)
{
    struct stat buf;

    if (lstat(entry->d_name, &buf) == -1)
    {
        ERR_EXIT("stat error");
    }

    // 类型权限组合字段
    char s_buf[12] = {0};
    strcpy(s_buf, "---------- ");
    // 文件类型
    file_type(&buf, s_buf);
    // 文件权限
    file_prem(&buf, s_buf);
    // 是否是该用户创建的
    if (getuid() == buf.st_uid)
    {
        s_buf[10] = ' ';
    }
    // 获取用户名
    struct passwd *passwdp = getpwuid(buf.st_uid);
    // 获取组名
    struct group *groupp = getgrgid(buf.st_gid);
    // 获取时间
    struct tm t;
    char date_time[64];
    strftime(date_time, sizeof(date_time), "%h %d %H:%M", localtime_r(&buf.st_mtimespec.tv_sec, &t));
    // 获取文件名
    char file_name[256] = {0};
    if (DT_LNK == entry->d_type)
    {
        char link_buf[256] = {0};
        strcpy(link_buf, entry->d_name);
        if (readlink(entry->d_name, link_buf, 256) == -1)
        {
            ERR_EXIT("readlink error");
        }
        sprintf(file_name, "%s -> %s", entry->d_name, link_buf);
    }
    else
    {
        sprintf(file_name, "%s", entry->d_name);
    }

    printf("%s %d %s %s "
           "%6d %s %s"
           "\n",
           s_buf, buf.st_nlink, passwdp->pw_name, groupp->gr_name,
           (int)buf.st_size, date_time, file_name);
}

int select_dirent(const struct dirent *);
int select_dirent(const struct dirent *entry)
{
    if (entry->d_name[0] == '.')
    {
        return 0;
    }
    return 1;
}

int main(int argc, char *argv[])
{
    if (argc < 2)
    {
        fprintf(stderr, "usage: %s filename pathname args\n", argv[0]);
        exit(EXIT_FAILURE);
    }

    DIR *dirp = opendir(argv[1]);
    if (NULL == dirp)
    {
        ERR_EXIT("opendir error");
    }

    struct dirent **entry_list;
    int entry_nums = scandir(argv[1], &entry_list, select_dirent, alphasort);

    for (int i = 0; i < entry_nums; i++)
    {
        file_list(entry_list[i]);
    }

    free(entry_list);

    return EXIT_SUCCESS;
}
```

执行结果如下

```bash
$ ln -s ls\ -l.c link
$ ls -l
total 40
-rwxr-xr-x  1 zengke  staff  9652 Aug 18 20:38 a.out
lrwxr-xr-x  1 zengke  staff     7 Aug 18 20:39 link -> ls -l.c
-rw-r--r--@ 1 zengke  staff  3991 Aug 18 20:38 ls -l.c
-r-x------  1 zengke  staff   991 Aug 10 19:53 test
$ ./a.out .
-rwxr-xr-x  1 zengke staff   9652 Aug 18 20:38 a.out
lrwxr-xr-x  1 zengke staff      7 Aug 18 20:39 link -> ls -l.c
-rw-r--r--  1 zengke staff   3991 Aug 18 20:38 ls -l.c
-r-x------  1 zengke staff    991 Aug 10 19:53 test
```

### 复制文件描述符

* dup
* dup2
* fcntl

#### 通过复制文件描述符完成输出重定向

```cpp
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <unistd.h>

#define ERR_EXIT(m)         \
    do                      \
    {                       \
        perror(m);          \
        exit(EXIT_FAILURE); \
    } while (0)

int main(int args, char *argv[])
{
    if (args != 2)
    {
        fprintf(stderr, "usage: %s filename\n", argv[0]);
        exit(EXIT_FAILURE);
    }

    int fd_redirect = open(argv[1], O_WRONLY);
    if (fd_redirect == -1)
    {
        ERR_EXIT("file open error");
    }
    // 关闭标准输出
    close(1);
    //dup查找空闲的文件描述符,如果定位到标准屏幕输出可以重定向
    if (dup(fd_redirect) != 1)
    {
        ERR_EXIT("dup open error");
    }
    // dup2 等同于close+dup
    if (dup2(fd_redirect, 1) != 1)
    {
        ERR_EXIT("dup2 open error");
    }
    //printf输出到复制的文件描述符1
    printf("helloworld\n");
    return EXIT_SUCCESS;
}
```

运行结果如下

```bash
$ touch test
$ > test
$ cat test
$ ./a.out test
$ cat test
helloworld
```

## 系统内核表示打开文件描述符

* 文件描述符表
* 文件表
* v节点表

待补充

### 文件描述符表

进程启动默认打开3个文件描述符, 用户打开的文件描述符从3开始

### 文件表

文件状态标志:读,写,追加,同步,非阻塞...  
当前文件偏移量:lseek随机读写利用该文件偏移量  
文件被文件描述符指向数量:复制文件描述符  
v指针指向v节点表:v节点表(一个文件仅有一个v节点表)

### v节点表

v节点表:说明  
v节点信息:stat读取文件元数据  
i节点信息:inode no和device id(文件打开时复制到v节点信息中)

### 一个进程两次打开一个文件内核数据结构

![文件描述符文件指针说明](/Linux系统编程/文件描述符文件指针说明.png)

一个进程两次打开同一个文件, 有各自独立的文件表项, v节点表是共享的

```cpp
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <unistd.h>

#define ERR_EXIT(m)         \
    do                      \
    {                       \
        perror(m);          \
        exit(EXIT_FAILURE); \
    } while (0)

int main(int args, char *argv[])
{
    if (args != 2)
    {
        fprintf(stderr, "usage: %s filename\n", argv[0]);
        exit(EXIT_FAILURE);
    }

    int fd_RDONLY = open(argv[1], O_RDONLY);
    // 以只读方式打开文件
    if (fd_RDONLY == -1)
    {
        ERR_EXIT("file open error");
    }

    int fd_RDWR = open(argv[1], O_RDWR);
    // 以读写方式打开
    if (fd_RDWR == -1)
    {
        ERR_EXIT("file open error");
    }

    // 两次读取,文件表偏移量是否相互影响,以判断文件描述符对应文件表是否独立
    char buff[1024];
    memset(buff, 0x00, sizeof(buff));
    read(fd_RDONLY, buff, 5);
    printf("%s\n", buff);
    memset(buff, 0x00, sizeof(buff));
    read(fd_RDWR, buff, 5);
    printf("%s\n", buff);

    // 继续写入,文件内柔是否相互影响,以判断文件是否独立
    write(fd_RDWR, "WRITE", 5);
    memset(buff, 0x00, sizeof(buff));
    read(fd_RDONLY, buff, 5);
    printf("%s\n", buff);

    close(fd_RDONLY);
    close(fd_RDWR);

    return EXIT_SUCCESS;
}
```

执行结果如下

```bash
$ touch test
$ echo helloworld >> test
$ cat test
helloworld
$ ./a.out test
hello
hello
WRITE
$ cat test
helloWRITE
```
