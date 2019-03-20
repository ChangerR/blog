---
title: Nginx源码分析-读写锁实现  
tags:
  - nginx
categories:
  - nginx
date: 2018-07-12 16:59:19
---

最近有空看了一下nginx的源码，看到了其读写锁的实现比较有意思，所以记录一下。

首先看下nginx 对cas操作的实现
``` c
static ngx_inline ngx_atomic_uint_t
ngx_atomic_cmp_set(ngx_atomic_t *lock, ngx_atomic_uint_t old,
    ngx_atomic_uint_t set)
{
    u_char  res;
    // nginx 直接调用汇编指令cmpxchgq，如果*lock == old,则将set值赋到*lock中 返回1，否则返回0，此条操作是原子操作。
    __asm__ volatile (

         NGX_SMP_LOCK
    "    cmpxchgq  %3, %1;   "
    "    sete      %0;       "

    : "=a" (res) : "m" (*lock), "a" (old), "r" (set) : "cc", "memory");

    return res;
}

```

接下来是nginx对写锁的实现，nginx对锁的等待主要使用自旋等待

``` c
#define NGX_RWLOCK_SPIN   2048
#define NGX_RWLOCK_WLOCK  ((ngx_atomic_uint_t) -1)


void
ngx_rwlock_wlock(ngx_atomic_t *lock)
{
    ngx_uint_t  i, n;
    // 无限循环直到得到锁的使用权
    for ( ;; ) {
        // 首先判断当前锁是否可用，如果可用则使用cas获得锁（将锁置为-1），成功则直接返回函数，失败继续下面的操作
        if (*lock == 0 && ngx_atomic_cmp_set(lock, 0, NGX_RWLOCK_WLOCK)) {
            return;
        }

        // 因为nginx的锁是使用自旋实现，只有在cpu数大于1时，自旋锁才有意义，否则会造成进程死锁
        if (ngx_ncpu > 1) {
            // 自旋等待，重试的时间从小到大
            for (n = 1; n < NGX_RWLOCK_SPIN; n <<= 1) {

                for (i = 0; i < n; i++) {
                    // x86平台调用 pause指令，降低CPU消耗
                    ngx_cpu_pause();
                }

                // 尝试获得锁
                if (*lock == 0
                    && ngx_atomic_cmp_set(lock, 0, NGX_RWLOCK_WLOCK))
                {
                    return;
                }
            }
        }
        //一轮尝试没有获得锁，让出CPU使用权
        ngx_sched_yield();
    }
}

```
下面是nginx对读锁的实现

``` c
void
ngx_rwlock_rlock(ngx_atomic_t *lock)
{
    ngx_uint_t         i, n;
    ngx_atomic_uint_t  readers;

    // 无限循环直到得到读锁
    for ( ;; ) {
        // 获取当前锁的数值
        readers = *lock;
        // 如果当前锁不处于写入状态，则对锁写入 readers + 1数值，如果写入成功就直接返回
        if (readers != NGX_RWLOCK_WLOCK
            && ngx_atomic_cmp_set(lock, readers, readers + 1))
        {
            return;
        }

        // 因为nginx的锁是使用自旋实现，只有在cpu数大于1时，自旋锁才有意义，否则会造成进程死锁
        if (ngx_ncpu > 1) {

            // 自旋等待，重试的时间从小到大
            for (n = 1; n < NGX_RWLOCK_SPIN; n <<= 1) {

                for (i = 0; i < n; i++) {
                    ngx_cpu_pause();
                }

                // 尝试获取锁
                readers = *lock;

                if (readers != NGX_RWLOCK_WLOCK
                    && ngx_atomic_cmp_set(lock, readers, readers + 1))
                {
                    return;
                }
            }
        }

        //一轮尝试没有获得锁，让出CPU使用权
        ngx_sched_yield();
    }
}

```

以下是取消锁定的代码实现
``` c
void
ngx_rwlock_unlock(ngx_atomic_t *lock)
{
    ngx_atomic_uint_t  readers;

    readers = *lock;

    // 如果当前是写入锁，则直接对其设置0，因为-1时不会再有其他的设置操作了
    if (readers == NGX_RWLOCK_WLOCK) {
        (void) ngx_atomic_cmp_set(lock, NGX_RWLOCK_WLOCK, 0);
        return;
    }

    // 循环直到执行CAS操作对readers - 1成功，读取锁减一
    for ( ;; ) {

        if (ngx_atomic_cmp_set(lock, readers, readers - 1)) {
            return;
        }

        readers = *lock;
    }
}
```

### 总结
nginx读写锁主要是使用CAS操作和自旋来实现的，通过锁的数值改变来进行加锁和解锁。