# git 源码分析 ---   初版git代码

###### cat-file命令 --- 查看某个文件

```
#include "cache.h"

int main(int argc, char **argv)
{
	unsigned char sha1[20];
	char type[20];
	void *buf;
	unsigned long size;
	char template[] = "temp_git_file_XXXXXX";
	int fd;

​```
if (argc != 2 || get_sha1_hex(argv[1], sha1))
	usage("cat-file: cat-file <sha1>");       // 1
buf = read_sha1_file(sha1, type, &size); // 2
if (!buf)
	exit(1);
fd = mkstemp(template);  // 3
if (fd < 0)
	usage("unable to create tempfile");
if (write(fd, buf, size) != size) //4
	strcpy(type, "bad");
printf("%s: %s\n", template, type);
​```
}

```

###### 1、这里有两个调用的函数在cache.h中定义了但是在read-cache.c中实现。

```
void usage(const char *err)
{
	fprintf(stderr, "read-tree: %s\n", err);
	exit(1);
}
```

###### 这个代码就是打印错误信息

```
int get_sha1_hex(char *hex, unsigned char *sha1)
{
	int i;
	for (i = 0; i < 20; i++) {
		unsigned int val = (hexval(hex[0]) << 4) | hexval(hex[1]);
		if (val & ~0xff)
			return -1;
		*sha1++ = val;
		hex += 2;
	}
	return 0;
}
```

###### 这里的理解是这个函数是用来判断是否输入的这个文件标记是合法的，每个文件的标记应该是一个sha1摘要产生出来的值。可以看出当判断中出现错误会返回-1，否则返回0

###### *******************************************

###### 2、这里用到了 read-cache.c中的函数read_sha1_file，作用暂时未知，会在看read-cache.c时说明。

###### 3、这一句代码用到了mkstemp函数，这个函数包含在stdlib.h头文件中，函数原型为int mkstemp(char *template），这个template是一个如代码中一样末尾为XXXXXX结尾的非空字符串，函数会随机替换这些X，函数返回一个文件描述符，如果失败则返回-1

###### 4、write函数包含在头文件unistd.h.h中这个文件，函数原型为 ssize_t write(int fd, const void *buf, size_t nbyte)，fd为文件描述符，buf为一个缓存区，nbyte指的是要写入的长度，如果写入成功会返回写入文件的长度，否则返回-1



###### 总的来说这个命令的作用是查看某个文件的内容，过程是验证这个文件存在与否，然后生成一个临时文件，将该文件写入这个临时文件，最后显示在控制台。这里为什么需要建立这个临时文件，个人的看法是因为这个临时文件只能由本机用户进行操作，因此提升了系统的安全性，也理所当然