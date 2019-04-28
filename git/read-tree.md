# git 源码分析

###### read-tree 

```
#include "cache.h"

static int unpack(unsigned char *sha1)
{
	void *buffer;
	unsigned long size;
	char type[20];

​```
buffer = read_sha1_file(sha1, type, &size);  // 2
if (!buffer)
	usage("unable to read sha1 file");
if (strcmp(type, "tree"))
	usage("expected a 'tree' node");
while (size) {     // 3
	int len = strlen(buffer)+1;
	unsigned char *sha1 = buffer + len;
	char *path = strchr(buffer, ' ')+1;
	unsigned int mode;
	if (size < len + 20 || sscanf(buffer, "%o", &mode) != 1)
		usage("corrupt 'tree' file");
	buffer = sha1 + 20;
	size -= len + 20;
	printf("%o %s (%s)\n", mode, path, sha1_to_hex(sha1));
}
return 0;
​```

}

int main(int argc, char **argv)
{
	int fd;
	unsigned char sha1[20];

​```
if (argc != 2)
	usage("read-tree <key>");
if (get_sha1_hex(argv[1], sha1) < 0)
	usage("read-tree <key>");
sha1_file_directory = getenv(DB_ENVIRONMENT);
if (!sha1_file_directory)
	sha1_file_directory = DEFAULT_DB_ENVIRONMENT;
if (unpack(sha1) < 0)  // 1
	usage("unpack failed");
return 0;
​```

}
```

###### 1、其实这个文件只有这个unpack函数是需要进行分析和研究的，那么具体这个函数是干什么的呢？

######  2、根据这边的sha1参数读取文件内容，这里的读取函数定义在read-cache.h中

###### 3、这个进入这个函数的主循环阶段，目前所能看懂的部分是，这部分是在循环读取文件信息，具体是怎么读取的需要进行dbg后再进行分析

