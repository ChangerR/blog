---
title: glog在协程中调用导致程序崩溃问题
tags:
  - coroutine glog
categories:
  - coroutine 
date: 2019-07-27 16:59:19
---

由于使用的协程是手动撸的，没有保存TIB中的信息，可以注释到其中OutputDebugStringA，或者保存TIB信息

更改win32的实现

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
		
		mov    edi, DWORD PTR [eax]
		; load NT_TIB into ECX
		mov    edx, fs:[018h]
		; load fiber local storage
		mov    ecx, [edx+010h]
		mov    [edi+32], ecx
		; load current deallocation stack
		mov    ecx, [edx+0e0ch]
		mov    [edi+32+4], ecx
		; load current stack limit
		mov    ecx, [edx+08h]
		mov    [edi+32+4*2], ecx
		; load current stack base
		mov    ecx, [edx+04h]
		mov    [edi+32+4*3], ecx
		; load current SEH exception list
		mov    ecx, [edx]
		mov    [edi+32+4*4], ecx
		
		mov    esp,DWORD PTR [eax + 4]

		; restore fiber local storage
		mov    ecx, [esp+32]
		mov    [edx+010h], ecx
		; restore current deallocation stack
		mov    ecx, [esp+32+4]
		mov    [edx+0e0ch], ecx
		; restore current stack limit
		mov    ecx, [esp+32+4*2]
		mov    [edx+08h], ecx
		; restore current stack base
		mov    ecx, [esp+32+4*3]
		mov    [edx+04h], ecx
		; restore current SEH exception list
		mov    ecx, [esp+32+4*4]
		mov    [edx], ecx
		
		
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

保存x64的实现
``` c

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
	
	; load NT_TIB
    mov  r10,  gs:[030h]
    ; save fiber local storage
    mov  rax, [r10+020h]
	mov  QWORD PTR [rcx + 112], rax
    ; save current deallocation stack
    mov  rax, [r10+01478h]
	mov  QWORD PTR [rcx + 112 + 8], rax
    ; save current stack limit
    mov  rax, [r10+010h]
	mov  QWORD PTR [rcx + 112 + 8 * 2], rax
    ; save current stack base
    mov  rax,  [r10+08h]
	mov  QWORD PTR [rcx + 112 + 8 * 3], rax
	
	; restore fiber local storage
    mov  rax, QWORD PTR [rdx + 112]
    mov  [r10+020h], rax
    ; restore current deallocation stack
    mov  rax, QWORD PTR [rdx + 112 + 8]
    mov  [r10+01478h], rax
    ; restore current stack limit
    mov  rax, QWORD PTR [rdx + 112 + 8 * 2]
    mov  [r10+010h], rax
    ; restore current stack base
    mov  rax, QWORD PTR [rdx + 112 + 8 * 3]
    mov  [r10+08h], rax
	
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

st_save_fpu:
	add    rcx,15
	and    rcx,0fffffffffffffff0h
	fxsave  [rcx]
	xor    eax,eax
	ret

st_restore_fpu:
	add    rcx,15
	and    rcx,0fffffffffffffff0h
	fxrstor [rcx]
	xor    eax,eax
	ret

public st_swap, st_save_fpu, st_restore_fpu

END


```