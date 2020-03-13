# PLSQL

SQL语言只能查询结果，而不能用作过程化的开发，而PLSQL就是为了弥补这个问题。它可以和其他编程语言那样实现逻辑判断、条件循环等操作，但是它又包括了一些SQL的语言的特性。

PLSQL是一种块结构的语言，它把一些语句放在了一个块内，随后将这个块一次性发送到服务器端，然后经过比如Oracle服务器编译解析，随后在PLSQL引擎内执行，执行时它将语句块中的内容切分，SQL语句由专门的SQL语句执行器执行，其他部分交给过程执行引擎执行。

## 块

这是PLSQL中最基本的单位，可以分为声明部分、执行部分、异常处理部分。

```
[DECLARE]
// 声明部分，可选，声明执行部分需要的变量或者常量
BEGIN
// 执行部分
[EXCEPTION]
异常处理部分，可选，处理执行部分出现的错误
END；
```

## 数据类型

1. 数值类型：类似NUMBER、BINARY_INTEGER等类型，可以用来存放数值类型的数据。
2. 字符类型：CHAR存放固定长度的字符串、VARCHAR类型存储可变长度的字符串、LONG存储更长的可变长度的字符串。
3. 时间类型：有DATE和TIMESTAMP两种时间类型。
4. 布尔类型：TURE、FLASE、NULL
5. 引用类型：%TYPE类型引用了数据库中的表的某列的类型作为变量的数据类型；%ROWTYPE是引用了数据库表中一行作为数据类型，即RECORD类型作为数据记录，这个类似对象的实例，访问其中的属性可以使用'.'。

## 逻辑控制语句

### 顺序结构

常常使用GOTO作为顺序结构中的跳转语句，它能够跳转到一个标签处，但是不能跳转到类似IF语句的位置。并且为了防止代码凌乱，不到迫不得已不会使用。

```
....
begin
...
	goto flag1;
...
	<< flag1 >>
end;
....
```

### 分支结构

IF-THEN-END

```
if <condition> then
	<statements>
end if;
```

IF-THEN-ELSE-END

```
if <condition> then
	<statements>
else
	<statements>
end if;
```

IF-THEN-ELSIF-THEN-END

```
if <condition1> then
	<statements>
elsif <condition2> then
	<statements>
else
	<statements>
end if;
```

CASE-WHEN-THEN

```
CASE <choice>
	WHEN <expression1> then <statements>;
	WHEN <expression2> then <statements>;
	...
	ELSE <expression_n> then <statements>;
end CASE;
```

### 循环结构

FOR

```
for <var> in (SELECT statement) loop
	<statements>
end loop;

for <var> in <low-bound> ... <up-bound> loop
	<statement>
end loop;
```

WHILE

```
while <conditions> loop
	<statements>
end loop;
```

## DDL执行

理论上在PLSQL中可以执行对应的DML语句和事务控制等语句，但是不能直接执行DDL语句。如果非要执行相应的DDL语句，则需要使用动态SQL的方法：

```
EXECUTE IMMEDIATE <sql statement>
[into <var list>]
[using <par list>]
```

如果说这里的动态SQL语句执行的是SELECT语句，那么可以将结果保存到varlist中；如果在sql语句中存在着一些参数需要替换，则可以使用parlist给出参数。

```
sql_yj:='select * from stuinfo_201812 where stuid=:1';  //:1是参数
execute immediate sql_yj into ls_stuinfo using ls_stuid;
```

## 异常

当程序发生异常时会自动跳转到块中的异常处理部分，它首先需要经过异常匹配，接下去调用对应的处理方法；假设并没有找到对应的异常处理程序，程序会直接中断抛出异常。

```
...
exception
	when <exception1> then
		<statements>
	when <exception2> then
		<statements>
	when others then // 如果上述都没有匹配到
		<statements>
end;
```

其中<exception>表达式可以是数据库中已经提供的预定义的异常名称；如果要使用那些非预定义的异常，则只需要在声明部分声明一个异常，这个声明过程需要将异常的名称和错误的编号进行一个关联。

```
ex_only exception; // 声明一个异常名称
PRAGMA EXCEPTION_INIT(ex_only,-00001); // 主键唯一性错误
```

当然在这个语言中也支持自定义异常，可以实现自己指定的那些异常。这需要在声明部分声明一个异常名称，然后在执行部分使用RAISE关键字主动抛出异常。

## 创建函数

在oracle中船舰函数，是通过使用PLSQL自定义编写的，要编写一个函数，就要处理函数的输入和输出，还有就是函数的执行部分：

```
create [or replace] function <name>
([par1, par2, ... , par_n])
return <datatype> // 返回的数据类型
is 
<declarations>
begin
<statements>
end;
```

如果要对函数进行修改，就可以使用括号中的replace关键字，当然可以在PLSQL中删除一个不想使用的函数，使用关键字drop完成该功能：

```
drop function <name>;
```

这样用户创建的函数就可以和Oracle内建函数一样使用了。

## 创建存储过程

存储过程和上面看到的函数有些类似，不过函数的功能一般限制于一些对于数据的计算，存储过程是一段存储在数据库中执行某个业务逻辑功能的程序模块，个人理解它比函数的功能要大。它可以由一段或者多段的PLSQL代码段组成。

```
create [or replace] procedure <name>
(par 1 in|out <datatype> // in 代表入参，使用时必须传入参数；out代表出参，必须在使用时有变量接收。
...)
is
<declarations>
begin
<statements>
end;

// 比如使用时 sp_score_pm('SC201801001','R20180101',ls_pm /*出参*/);
```

## 触发器

当对Oracle数据库对象做特定的某些操作的时候，需要特别触发执行一段PLSQL代码，这个功能叫做触发器，触发器的作用有很多，它可以辅助完成一些日志的操作，或者对数据的合法和正确性进行核查，同时对数据库对象的操作进行一定的限制。

### DML触发器

这是Oracle开发中最常用到的，主要针对insert，delete，update操作的事件进行触发。

```
create [or replace] trigger <name>
before|after  // 确定是在数据改变前触发还是改变后触发
<processing(delete, insert or update)> // 确定触发的事件的类型
on <table_name> 
[for each row] // 如果有这一行说明是行级的触发器，否则是语句级别的触发器
[follows <name1>]  // 代表触发顺序，这说明这个触发器是跟在某个触发器后的
[when <condtions>] // 触发的具体条件
declare
...
begin
<statements>
end;
```

### DDL触发器

这一类触发器是专门对create，drop等操作制定的，为了限定或者记录对某个对象的DDL操作。

```
create [or replace] trigger <name>
before|after // 说明触发器是在操作前触发还是操作后触发
ddl_event|database event // 说明这个触发的类型
on SCHEMA(<obj>)|DATABASE(<database>) // 表明这个触发器作用的对象是单个对象还是数据库
[follows <name1>]
[when <conditions>]
declare
begin
<statements>
end;
```

## 游标

游标的作用是可以通过类似于数组一样，将使用select语句查询出来的数据集存放在内存当中，然后通过游标可以做到查询指定数据的作用。

通常对游标的使用分为五步：声明游标、打开游标、读取数据、关闭游标、删除游标。对应到PLSQL中：

```
declare cursor <name>
is <statement>

open <name>

fetch <name> into <var>

close <name>
```

