---
layout: page
title: About
permalink: /about/
---
```
.intel_syntax noprefix

.text
.global _start
_start:
    sub rsp, 0x20
    vpxor ymm0, ymm0, ymm0
    vmovdqu [rsp], ymm0
    mov eax, 1 
    mov edi, 1
    mov dword ptr [rsp], 0x6f6c2049 
    mov dword ptr [rsp + 0x4], 0x74206576 
    mov dword ptr [rsp + 0x8], 0x6e696e75
    mov dword ptr [rsp + 0xc], 0x65702067 
    mov dword ptr [rsp + 0x10], 0x0a6672
    mov rsi, rsp
    mov edx, 19
    syscall
    mov eax, 60 
    xor edi, edi
    syscall
```
