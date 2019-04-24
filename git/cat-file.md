# git 源码分析 ---   初版git代码

###### cat-file命令

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
if (write(fd, buf, size) != size)
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



###### *******************************************

###### 2、