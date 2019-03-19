---
title: Python Ctypes Callback 踩坑记录  
tags:
  - ctypes
  - python
categories:
  - python
date: 2018-06-22 16:59:19
---

最近写的C++程序库需要对暴露出的接口进行测试，但是工期比较紧张，就想把函数接口封装到Python中，然后让QA直接使用Python对函数接口进行测试。在封装过程中遇到一些坑记录一下。

### 坑1:
``` c
typedef void* (*CALLBACK)();

extern DLL_API void* foo(CALLBACK callback) 
{
    printf("foo calling callback\n");
    callback();
    printf("callback returned in foo\n");
}

// gcc -shared -fPIC -O2 -o libfoo.so foo.c
```

在Python程序中调用foo

``` python
from ctypes import *
p = c_int(1)
def callback(*args):
    
    # 注意在此处不能直接返回byref(p),这样会报错，要将其转为c_void_p，并且去取value,不然会报错
    # 报错信息如下：
    #   foo calling callback
    #   Traceback (most recent call last):
    #      File "source/callbacks.c", line 216, in 'converting
    r = cast(byref(p), c_void_p)
    return r.value

lib = cdll['libfoo.so']
lib.foo.argtypes = [CFUNCTYPE(c_void_p)]
lib.foo(lib.foo.argtypes[0](callback))
```

### 坑2:

对cytpes的指针的指针赋值
``` python
FOO_TYPE = CFUNCTYPE(None, POINTER(c_void_p))
t = c_int(1)
def c_callback_foo(p):
    # 对ctypes中指针的指针赋值应该使用其下标索引，其他赋值无法传入上层
    p[0] = cast(byref(t), c_void_p)

func = FOO_TYPE(c_callback_foo)
```