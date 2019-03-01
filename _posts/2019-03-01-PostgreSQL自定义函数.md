---
layout: post
title: "PostgreSQL自定义C函数"
date: 2019-03-01 09:42
categories: [postgres]
tags: [postgres]
---

通常在数据库中我们都是通过SQL语言编写自定义函数，好处是语法简单，只需要关注要实现的业务
逻辑即可，但限制也很明显：只能使用数据库支持的SQL语法和函数。在PG数据库中为开发人员提供
非常灵活的接口，利用这些接口可以实现自定义C函数，利用这些C函数进一步扩展系统支持的函数集
合。

# 规范
---
要利用PG的接口实现自定义函数，必须得遵循其接口规范，主要有以下几点：
1. 这些C函数以动态库的方式提供；
2. 为了避免版本兼容问题，在C函数源文件中需要包含`fmgr.h`头文件并包含`PG_MODULE_MAGIC`；
3. 被调用的函数遵循调用规范，使用宏`PG_FUNCTION_INFO_V1`包裹被调用函数；


# 准备工作
---
在真正编写可在SQL语句中调用的函数之前，还有一些细节要弄清楚：怎样给函数传递参数，函数如何
返回值。在C语言中内置类型非常简单，也支持struct结构体，但数据库往往有自己的类型的，例如
PG中有int、text、bytea等类型，PG内核定义了C类型和数据库类型之间的对应关系，具体对应关系
如下：

| SQL Type                  | C Type        | Defined In                           |
| ------------------------- | ------------- | ------------------------------------ |
| abstime                   | AbsoluteTime  | utils/nabstime.h                     |
| bigint (int8)             | int64         | postgres.h                           |
| boolean                   | bool          | postgres.h (maybe compiler built-in) |
| box                       | BOX*          | utils/geo_decls.h                    |
| bytea                     | bytea*        | postgres.h                           |
| "char"                    | char          | (compiler built-in)                  |
| character                 | BpChar*       | postgres.h                           |
| cid                       | CommandId     | postgres.h                           |
| date                      | DateADT       | utils/date.h                         |
| smallint (int2)           | int16         | postgres.h                           |
| int2vector                | int2vector*   | postgres.h                           |
| integer (int4)            | int32         | postgres.h                           |
| real (float4)             | float4*       | postgres.h                           |
| double precision (float8) | float8*       | postgres.h                           |
| interval                  | Interval*     | datatype/timestamp.h                 |
| lseg                      | LSEG*         | utils/geo_decls.h                    |
| name                      | Name          | postgres.h                           |
| oid                       | Oid           | postgres.h                           |
| oidvector                 | oidvector*    | postgres.h                           |
| path                      | PATH*         | utils/geo_decls.h                    |
| point                     | POINT*        | utils/geo_decls.h                    |
| regproc                   | regproc       | postgres.h                           |
| reltime                   | RelativeTime  | utils/nabstime.h                     |
| text                      | text*         | postgres.h                           |
| tid                       | ItemPointer   | storage/itemptr.h                    |
| time                      | TimeADT       | utils/date.h                         |
| time with time zone       | TimeTzADT     | utils/date.h                         |
| timestamp                 | Timestamp*    | datatype/timestamp.h                 |
| tinterval                 | TimeInterval  | utils/nabstime.h                     |
| varchar                   | VarChar*      | postgres.h                           |
| xid                       | TransactionId | postgres.h                           |

知道了参数类型对应关系还不够，还需要知道如何处理对应参数。例如某个C函数实现根据表名查询
pg_class系统表中的信息，输入参数用SQL text类型来表示表名，返回值是pg_class中的一条记录。
由上述对应关系可知，C函数有个输入参数为`text *`，但返回值如何表示呢？

# Datum
---
有时候C函数的返回值可能是一个简单的int类型，有时候可能是某张表的一条记录，更复杂的可能是
多条记录。为了统一接口，在内核中定义了一种数据类型`Datum`：
```c
/*
 * A Datum contains either a value of a pass-by-value type or a pointer to a
 * value of a pass-by-reference type.  Therefore, we require:
 *
 * sizeof(Datum) == sizeof(void *) == 4 or 8
 *
 * The macros below and the analogous macros for other types should be used to
 * convert between a Datum and the appropriate C type.
 */

typedef uintptr_t Datum;

#define SIZEOF_DATUM SIZEOF_VOID_P
```
从源码可知，Datum实际上是类似`void *`的指针，在64位系统上占用8字节内存。如果返回值能够
存放在8字节的内存中，则采用`传值(pass-by-value)`方式，否则其中存放的是一个指针。

对于上一节的例子来说，输入值是text类型，在PG内核中的定义如下：
```c
/* ----------------
 *		Variable-length datatypes all share the 'struct varlena' header.
 *
 * NOTE: for TOASTable types, this is an oversimplification, since the value
 * may be compressed or moved out-of-line.  However datatype-specific routines
 * are mostly content to deal with de-TOASTed values only, and of course
 * client-side routines should never see a TOASTed value.  But even in a
 * de-TOASTed value, beware of touching vl_len_ directly, as its
 * representation is no longer convenient.  It's recommended that code always
 * use macros VARDATA_ANY, VARSIZE_ANY, VARSIZE_ANY_EXHDR, VARDATA, VARSIZE,
 * and SET_VARSIZE instead of relying on direct mentions of the struct fields.
 * See postgres.h for details of the TOASTed form.
 * ----------------
 */
struct varlena
{
	char		vl_len_[4];		/* Do not touch this field directly! */
	char		vl_dat[FLEXIBLE_ARRAY_MEMBER];	/* Data content is here */
};

#define VARHDRSZ		((int32) sizeof(int32))

typedef struct varlena text;
```
这是一个`变长`类型，只能以`传引用(pass-by-reference)`方式传递，返回值是某张表的一条记录，
自然也是变长的，故Datum中存放的是一个指针。

# 示例
---
在编写真正有用的函数之前先热热身，看一个简单的例子：
```c
#include <string.h>
#include "postgres.h"
#include "fmgr.h"

PG_MODULE_MAGIC;

/*
 * 函数调用规范隐藏了函数输入参数和返回值处理的细节。
 */
PG_FUNCTION_INFO_V1(copytext);

/*
 * copytext函数接受一个输入参数SQL text，返回其拷贝。
 * PG_FUNCTION_ARGS表示函数的输入参数，需要使用命名模式为PG_GETARG_xxx()的函数
 * 来获取。例如输入参数是SQL text，应该使用PG_GETARG_TEXT_PP()函数获取输入值，
 * 该函数接受一个整型输入参数，表示参数的顺序，从0开始计数。
 */
Datum
copytext(PG_FUNCTION_ARGS)
{
    text *input = PG_GETARG_TEXT_PP(0);

    /*
     * 返回值所需的内存要自己分配，注意text为变长类型，除了分配用于保存数据的内存
     * 外，还需要分配一定空间保存整个text结构的大小。PG内核中定义了相关宏来简化这
     * 一操作：
     * VARHDRSZ：变长结构的头大小，用来保存整个结构体长度；
     * VARSIZE_ANY_EXHDR：除头大小外剩余长度，也就是实际保存数据的大小。
     */
    text *output = (text *)palloc(VARHDRSZ + VARSIZE_ANY_EXHDR(input));

    /* 设置output总长度。 */
    SET_VARSIZE(output, VARHDRSZ + VARSIZE_ANY_EXHDR(input));

    /* 拷贝数据。 */
    memcpy((void *) VARDATA(output), (void *) VARDATA(input),
            VARSIZE_ANY_EXHDR(input));

    /* 返回数据，传引用方式。 */
    PG_RETURN_TEXT_P(output);
}
```

# 安装
---
假设上述文件保存为`copytext.c`，并编译为动态库`libcopytext.so`，修改`postgresql.conf`
配置文件的`shared_preload_libraries`参数：
```bash
# 这里没有指定动态库的全路径，假设该库存放在PG的lib目录中
shared_preload_libraries = 'libcopytext'
```

定义SQL语句中可调用的函数：
```sql
CREATE FUNCTION copytext(text) RETURNS text
     AS '$libdir/libcopytext', 'copytext'
     LANGUAGE C STRICT;
```
这里添加了`STRICT`关键字，表示如果输入参数为`NULL`，那么返回值也是`NULL`。至此就可以在
SQL语句中使用`copytext`函数了，示例如下：
```sql
postgres@postgres=# select copytext('hello world');
  copytext
-------------
 hello world
(1 row)
```

# 更复杂的例子
---
热身完毕，接下来我们实现一个稍微复杂点的例子：定义一个函数get_table_info，输入参数是表名，
字符串text类型，返回pg_class中该表的描述信息。