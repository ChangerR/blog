---
title: windwos下的协程设计
tags:
  - coroutine
categories:
  - coroutine 
date: 2019-07-27 16:59:19
---

最近工作使用到异步IO,在解某一段数据是无法数据是否已经收取完毕，所有想到使用协程来简化异步IO的业务处理逻辑。

在Linux下可以使用ucontext库来进行跳转，windows下面有纤程但感觉不是特别好用，就参照libco手撸了一个windows下可以用的协程库。

定义st_regs结构体用于保存当前的寄存器

``` c
#define kEIP 0
#define kESP 7

#define kRDI 7
#define kRSI 8
#define kRIP 9
#define kRSP 13
#define kRCX 11

struct st_regs
{
#ifdef _MSC_VER  
#if defined(_WIN32) && !defined(_WIN64)
	void *regs[8];
#else
	void *regs[14];
#endif
#endif	
#ifdef SAVE_FPU
	char fpu[528];
#endif
	size_t ss_size;
	char *ss_sp;
};

```

置换当前运行环境,win32实现

``` c
#if defined(_MSC_VER) && defined(_WIN32) && !defined(_WIN64)
__declspec( naked ) int st_swap(struct st_regs* current, struct st_regs* target)
{
	__asm
	{
		lea    eax,[esp + 4]
		mov    esp,DWORD PTR [esp + 4]
		lea    esp,[esp + 32]
		push   eax
		push   ebp
		push   esi
		push   edi
		push   edx
		push   ecx
		push   ebx
		push   DWORD PTR [eax - 4]
		pop    eax
		pop    ebx
		pop    ecx
		pop    edx
		pop    edi
		pop    esi
		pop    ebp
		pop    esp
		push   eax
		xor    eax,eax
		ret
	}
}
```

置换当前运行环境,由于win64不能直接把汇编代码嵌入到cpp文件中，所有建立st_swap64.asm文件

``` asm

.CODE

st_swap:
	lea    rax , [rsp + 8]
	lea    rsp , [rcx + 112]
	push   rax
	push   rbx
	push   rcx
	push   rdx
	push   QWORD PTR [rax - 8]
	push   rsi
	push   rdi
	push   rbp
	push   r8
	push   r9
	push   r12
	push   r13
	push   r14
	push   r15
	
	mov    rsp,rdx
	
	pop    r15
	pop    r14
	pop    r13
	pop    r12
	pop    r9
	pop    r8
	pop    rbp
	pop    rdi
	pop    rsi
	pop    rax
	pop    rdx
	pop    rcx
	pop    rbx
	pop    rsp
	push   rax
	xor    eax,eax
	ret


public st_swap

END


```

初始化当前st_reg环境

``` c
typdef void (*STCALL_ROUTINE_FUNC)(void*);

void make_st_regs(struct st_regs* reg, int stack_size, STCALL_ROUTINE_FUNC stcall_routine_func, void* ctx)
{
    char* sp;
	  struct st_param_t* param;
    reg->ss_size = stack_size;
	  reg->ss_sp = (char*)malloc(stack_size);
    
#if (defined(_MSC_VER) && defined(_WIN32) && !defined(_WIN64)) || (defined(__i386__) && defined(__unix__))
		sp = reg->ss_sp + reg->ss_size - sizeof(struct st_param_t);
		sp = (char*)((unsigned long)sp & -16l);
		
		param = (struct st_param_t*)sp ;
		param->s1 = ctx;

		memset(reg->regs, 0, sizeof(reg->regs));

		reg->regs[ kESP ] = (char*)(sp) - sizeof(void*);
		reg->regs[ kEIP ] = (char*)stcall_routine_func;
	
#elif defined(_WIN64) || defined(__MINGW64__)
		(void)param;
		sp = reg->ss_sp + reg->ss_size - sizeof(struct st_param_t);
		sp = (char*)((unsigned long long)sp & -16ll);
		
		memset(reg->regs, 0, sizeof(reg->regs));

		reg->regs[ kRSP ] = (char*)(sp) - sizeof(void*);
		reg->regs[ kRIP ] = (char*)stcall_routine_func;
		reg->regs[ kRCX ] = (char*)ctx;

#endif
}
```

