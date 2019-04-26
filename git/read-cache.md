# git 源码分析

###### read-cache --- 辅助性质的一个c文件，实现多个可以调用的接口函数

```
void * read_sha1_file(unsigned char *sha1, char *type, unsigned long *size)
{
	z_stream stream;    // 1
	char buffer[8192];
	struct stat st;   // 2
	int i, fd, ret, bytes;
	void *map, *buf;
	char *filename = sha1_file_name(sha1); //3

​```
fd = open(filename, O_RDONLY);  //O_RDONLY 只读
if (fd < 0) {
	perror(filename);
	return NULL;
}
if (fstat(fd, &st) < 0) { // 4
	close(fd);
	return NULL;
}
map = mmap(NULL, st.st_size, PROT_READ, MAP_PRIVATE, fd, 0); // 5  建立映射
close(fd);
if (-1 == (int)(long)map)
	return NULL;

/* Get the data stream */
memset(&stream, 0, sizeof(stream));
stream.next_in = map;   // 入口
stream.avail_in = st.st_size; // 入口总长
stream.next_out = buffer; //出口
stream.avail_out = sizeof(buffer); //出口长度最长8192

inflateInit(&stream);
ret = inflate(&stream, 0);  //  6 解压缩
if (sscanf(buffer, "%10s %lu", type, size) != 2)  // 7
	return NULL;
bytes = strlen(buffer) + 1;
buf = malloc(*size);
if (!buf)
	return NULL;

memcpy(buf, buffer + bytes, stream.total_out - bytes);
bytes = stream.total_out - bytes;
if (bytes < *size && ret == Z_OK) {
	stream.next_out = buf + bytes;
	stream.avail_out = *size - bytes;
	while (inflate(&stream, Z_FINISH) == Z_OK)
		/* nothing */;
}
inflateEnd(&stream);
return buf;
​```

}
```

###### 先来分析一下这个函数，因为这个函数在之前看到的cat-file中也出现过。

###### 1、z_stream 第一次见到这个类型，这个类型定义在zlib.h中，是在进行压缩前需要先初始化的一个数据类型，具体不做展开。

###### 2、stat这个类型是出现在<sys/stat.h>中，他是用来描述linux文件系统中一个文件的属性的结构体。

###### 3、sha1_file_name 函数 --- 目前暂时理解为根据sha1来获取文件名

```
char *sha1_file_name(unsigned char *sha1)
{
	int i;
	static char *name, *base;

​```
if (!base) {
	char *sha1_file_directory = getenv(DB_ENVIRONMENT) ? : DEFAULT_DB_ENVIRONMENT;
	int len = strlen(sha1_file_directory);
	base = malloc(len + 60);
	memcpy(base, sha1_file_directory, len);
	memset(base+len, 0, 60);
	base[len] = '/';
	base[len+3] = '/';
	name = base + len + 1;
}
for (i = 0; i < 20; i++) {
	static char hex[] = "0123456789abcdef";
	unsigned int val = sha1[i];
	char *pos = name + i*2 + (i > 0);
	*pos++ = hex[val >> 4];
	*pos = hex[val & 0xf];
}
return base;
​```

}
```

###### 4、 fstat 目的是根据文件描述符去判断目前文件的状态，执行成功返回0， 失败则返回-1。

###### 5、需要看一下mmap这个函数，这个函数的目的是将一个文件或者对象映射进内存中。函数原型：void* mmap(void* start,size_t length,int prot,int flags,int fd,off_t offset) ， start：映射区的开始地址； length：映射区的长度； prot：内存中的保护标治；flags：指定映射的类型； fd：文件描述符；offset: 被映射对象内容的起点。

###### 6、inflate 函数的目的是将参数指向的文件解压缩，辅助函数由inflateInit和inflateEnd

###### 7、sscanf 格式化输入，函数原型`int`  sscanf ( const char *buffer,   const char *format, [ argument ] ...   ); ，按照format指定格式将参数输入到buffer中。

###### 接下去暂时就看不懂了，但是总的意思应该是解压后读入一个缓存区，然后将这个缓存区中的内容，也就是文件的内容返回出去，有空使用gdb进行调试