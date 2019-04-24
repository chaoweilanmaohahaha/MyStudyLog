# git 源码分析 ---   初版git代码

###### init-db --- 初始化文件夹，在文件夹中新建隐藏文件 .dircache

```
#include "cache.h"

int main(int argc, char **argv)
{
	char *sha1_dir = getenv(DB_ENVIRONMENT), *path; // 1

​```
if (mkdir(".dircache", 0700) < 0) {  //2
	perror("unable to create .dircache");
	exit(1);
}

/*
 * If you want to, you can share the DB area with any number of branches.
 * That has advantages: you can save space by sharing all the SHA1 objects.
 * On the other hand, it might just make lookup slower and messier. You
 * be the judge.
 */
sha1_dir = getenv(DB_ENVIRONMENT);
if (sha1_dir) {
	struct stat st;
	if (!stat(sha1_dir, &st) < 0 && S_ISDIR(st.st_mode))
		return;
	fprintf(stderr, "DB_ENVIRONMENT set to bad directory %s: ", sha1_dir);
}

/*
 * The default case is to have a DB per managed directory. 
 */
sha1_dir = DEFAULT_DB_ENVIRONMENT;  // 3
fprintf(stderr, "defaulting to private storage area\n");
len = strlen(sha1_dir);
if (mkdir(sha1_dir, 0700) < 0) {
	if (errno != EEXIST) {
		perror(sha1_dir);
		exit(1);
	}
}
path = malloc(len + 40); // 4
memcpy(path, sha1_dir, len); // 5
for (i = 0; i < 256; i++) {
	sprintf(path+len, "/%02x", i);  // 6
	if (mkdir(path, 0700) < 0) {
		if (errno != EEXIST) {
			perror(path);
			exit(1);
		}
	}
}
return 0;
​```

}
```



###### 1、getenv 函数的目的是获取系统环境变量；存在于stdlib.h头文件下，使用原型是 char* getenv(char* envvar), envvar 指环境变量中的key，函数返回其value.

###### 2、mkdir 函数是在linux下的函数，目的是创建某个文件夹，存在于\#include <sys/stat.h> 和 #include <sys/types.h> ，函数原型int mkdir(const char *pathname, mode_t mode)， pathname是指建立文件夹的路径，mode指的是文件夹的读写权限设置，返回为0则成功，失败返回-1

###### 3、#define DEFAULT_DB_ENVIRONMENT ".dircache/objects"  定义于cache.h

###### 4、这里不是很清楚为何多开40B的空间，可能为了后续的书写

###### 5、这里就是将路径赋值给path变量，该思考的问题是为何这里不直接用strcpy函数！！

###### 6、这个函数的功能其实就是拼接路径，这样可以生成子文件夹的路径名

###### 代码的本意是初始化这个仓库，在该文件夹下会生成一个.dircache的隐藏文件夹，然后再在这个文件夹下创建一个object文件夹，其中包含256个名字从00-ff的文件夹作为初始化