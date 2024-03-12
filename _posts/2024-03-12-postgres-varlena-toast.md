---
layout: post
title: postgres varlena和toast机制
date: 2024-03-12
Author: levy5307
tags: [postgres]
comments: true
toc: true
---

## varlena

postgres使用varlena来表示变长的数据类型，通过语句`SELECT typname FROM pg_type WHERE typlen = -1`就可以看到所有采用varlena格式的数据类型：

```
                typname                 
----------------------------------------
 bytea
 int2vector
 text
 oidvector
 pg_type
 pg_attribute
 pg_proc
 pg_class
 json
 xml
:
```

### 通用结构

varlena有一个通用的格式：

```c
struct varlena
{
	char		vl_len_[4];		/* Do not touch this field directly! */
	char		vl_dat[FLEXIBLE_ARRAY_MEMBER];	/* Data content is here */
};
```

可以将其理解为一个接口，实际上varlena细分为很多格式：`varattrib_1b`/`varattrib_4b`/`varattrib_1b_e`：

- 当第一个字节的最高位为1，并且第一个字节不为`1000 0000`时，那么就是`varattrib_1b`。

- 当第一个字节的最高位为1，并且第一个字节为`1000 0000`时，那么就是`varattrib_1b_e`。

- 当第一个字节的最高位为0时，那么就是`varattrib_4b`。

这里的`xb`中的x，其实是指`va_header`部分的长度。

### varattrib_1b

### varattrib_4b

### varattrib_1b_e

## toast

## 写入

### 切片策略

- PLAIN(p)

- EXTENDED(x)

- EXTERNAL(e)

- MAIN(m)
