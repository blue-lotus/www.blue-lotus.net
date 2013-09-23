---
title: 'UFO CTF 2013 Reverse &#8211; KeyGenMe Writeup'
author: fish
layout: post
permalink: /ufo-ctf-2013-reverse-keygenme-writeup/
categories:
  - CTF
  - writeup
---
I would come back with more details of this challenge.

Here is the approach that I used to generate a keygen: make a DLL and inject it into the target process. The source code of my library is as follows.

    #include "stdafx.h"
    #include <iostream>
    
    typedef unsigned char byte;
    
    extern HMODULE m_hModule;
    HANDLE m_hConsole;
    
    #pragma pack(1)
    struct CPU_
    {
        byte al_;
        unsigned int k_init[4];
        unsigned int data[4];
        byte ip_;
        byte sp_;
        byte stack[8];
    };
    #pragma pack()
    
    unsigned int step_func[256] = {0x4011D0, 
        0x401310, 
        0x401490, 
        0x401660, 
        0x4017F0, 
        0x401940, 
        0x401AC0, 
        0x401C90, 
        0x401DF0, 
        0x401FA0, 
        0x402130, 
        0x4022E0, 
        0x402460, 
        0x402660, 
        0x4027D0, 
        0x402960, 
        0x402B50, 
        0x402CB0, 
        0x402E40, 
        0x403010, 
        0x4031C0, 
        0x403330, 
        0x4034D0, 
        0x4036A0, 
        0x403840, 
        0x4039A0, 
        0x403B00, 
        0x403C50, 
        0x403DE0, 
        0x403F30, 
        0x4040A0, 
        0x404260, 
        0x404430, 
        0x4045F0, 
        0x404760, 
        0x404900, 
        0x404A70, 
        0x404C30, 
        0x404DA0, 
        0x404F40, 
        0x4050C0, 
        0x4051E0, 
        0x405380, 
        0x4054D0, 
        0x4056A0, 
        0x405850, 
        0x4059D0, 
        0x405B50, 
        0x405D10, 
        0x405E90, 
        0x406030, 
        0x406140, 
        0x4062D0, 
        0x4064A0, 
        0x406610, 
        0x4067A0, 
        0x406940, 
        0x406A80, 
        0x406C20, 
        0x406DF0, 
        0x406F90, 
        0x407110, 
        0x4072A0, 
        0x407420, 
        0x4075A0, 
        0x407740, 
        0x407880, 
        0x407A50, 
        0x407C00, 
        0x407DB0, 
        0x407F80, 
        0x408120, 
        0x408290, 
        0x408410, 
        0x408570, 
        0x4086D0, 
        0x408890, 
        0x4089D0, 
        0x408B70, 
        0x408D20, 
        0x408EA0, 
        0x408FE0, 
        0x409170, 
        0x4092B0, 
        0x409490, 
        0x409640, 
        0x4097D0, 
        0x4099A0, 
        0x409B70, 
        0x409CC0, 
        0x409E90, 
        0x409FE0, 
        0x40A150, 
        0x40A290, 
        0x40A410, 
        0x40A5D0, 
        0x40A770, 
        0x40A8D0, 
        0x40AA30, 
        0x40ABA0, 
        0x40AD20, 
        0x40AEB0, 
        0x40B050, 
        0x40B1B0, 
        0x40B3B0, 
        0x40B530, 
        0x40B6D0, 
        0x40B860, 
        0x40BA40, 
        0x40BBD0, 
        0x40BD50, 
        0x40BEB0, 
        0x40C060, 
        0x40C1A0, 
        0x40C370, 
        0x40C510, 
        0x40C670, 
        0x40C7F0, 
        0x40C990, 
        0x40CB00, 
        0x40CCC0, 
        0x40CE50, 
        0x40CFF0, 
        0x40D1C0, 
        0x40D300, 
        0x40D460, 
        0x40D5D0, 
        0x40D710, 
        0x40D870, 
        0x40DA00, 
        0x40DBD0, 
        0x40DD60, 
        0x40DEE0, 
        0x40E080, 
        0x40E230, 
        0x40E3C0, 
        0x40E550, 
        0x40E6F0, 
        0x40E890, 
        0x40EA10, 
        0x40EB50, 
        0x40ED50, 
        0x40EE60, 
        0x40EF80, 
        0x40F0F0, 
        0x40F2A0, 
        0x40F400, 
        0x40F560, 
        0x40F710, 
        0x40F890, 
        0x40FA30, 
        0x40FBC0, 
        0x40FD50, 
        0x40FE90, 
        0x410020, 
        0x410180, 
        0x410340, 
        0x4104B0, 
        0x410620, 
        0x410780, 
        0x410930, 
        0x410B20, 
        0x410CB0, 
        0x410E00, 
        0x410F50, 
        0x411130, 
        0x411310, 
        0x4114B0, 
        0x4115F0, 
        0x411730, 
        0x4118E0, 
        0x411A40, 
        0x411BF0, 
        0x411DA0, 
        0x411F30, 
        0x4120F0, 
        0x412250, 
        0x4123B0, 
        0x412560, 
        0x4126D0, 
        0x412860, 
        0x4129F0, 
        0x412B90, 
        0x412D30, 
        0x412F10, 
        0x413070, 
        0x413200, 
        0x4133A0, 
        0x413510, 
        0x413660, 
        0x4137A0, 
        0x4138D0, 
        0x413A50, 
        0x413BC0, 
        0x413D30, 
        0x413F00, 
        0x4140C0, 
        0x414220, 
        0x4143A0, 
        0x4144F0, 
        0x414630, 
        0x4147C0, 
        0x414930, 
        0x414AC0, 
        0x414C70, 
        0x414DD0, 
        0x414F80, 
        0x4150E0, 
        0x4152A0, 
        0x415430, 
        0x4155E0, 
        0x415750, 
        0x4158C0, 
        0x415A40, 
        0x415BD0, 
        0x415D30, 
        0x415ED0, 
        0x416060, 
        0x416210, 
        0x416360, 
        0x416520, 
        0x416690, 
        0x416850, 
        0x4169F0, 
        0x416B20, 
        0x416CE0, 
        0x416E90, 
        0x417010, 
        0x417170, 
        0x417300, 
        0x417490, 
        0x417690, 
        0x417860, 
        0x417A20, 
        0x417B70, 
        0x417D50, 
        0x417EC0, 
        0x417FD0, 
        0x418140, 
        0x418290, 
        0x418420, 
        0x418610, 
        0x4187C0, 
        0x418930, 
        0x418AD0, 
        0x418C50, 
        0x418E30, 
        0x418FD0, 
        0x419140, 
        0x419280, 
        0x419410, 
        0x4195B0, 
        0x419710, 
        0x4198B0, 
        0x419A00, 
        0x419B60};
    
    typedef void (__thiscall *__prepare_teamname)(struct CPU_* pCpu, char* szTeamName);
    typedef void (*__tean)(int uDecryptFlag, byte* pSrc, byte* pDst, unsigned int *k, unsigned int length);
    typedef void (__thiscall *__step)(struct CPU_* pCpu, unsigned int operation);
    
    __prepare_teamname prepare_teamname;
    __tean tean;
    __step step[256];
    
    VOID Initialize()
    {
        prepare_teamname = (__prepare_teamname)0x419d00;
        tean = (__tean)0x41ed50;
    
        for(int i = 0; i <= 0xff; ++i)
        {
            step[i] = (__step)step_func[i];
        }
    }
    
    BOOL Search(byte* cmd, int ip, struct CPU_* cpu_, byte* dst_stack, byte* operation, int* final_ip, byte* init_stack)
    {
        if(ip >= 16)
        {
            // Reach the end!
            if(cpu_->ip_ >= 32 && cpu_->sp_ >= 8)
            {
                *final_ip = cpu_->ip_;
                return TRUE;
            }
            return FALSE;
        }
        else if(ip >= 7)
        {
            // Do not pop anything onto our stack!
            // We only try to manipulate the ip
            for(int i = 1; i <= 0xff; ++i)
            {
                struct CPU_ new_cpu;
                memcpy(&new_cpu, cpu_, sizeof(struct CPU_));
    
                int old_sp = new_cpu.sp_;
                // Step
                step[cmd[ip]](&new_cpu, i);
                if(old_sp != new_cpu.sp_)
                {
                    // A new value has been written on stack
                    // return FALSE;
                }
                else
                {
                    if(Search(cmd, ip + 1, &new_cpu, dst_stack, operation, final_ip, init_stack))
                    {
                        operation[ip] = i;
                        return TRUE;
                    }
                }
            }
        }
        else /* if(ip >= 0 && ip < 7) */
        {
            for(int init_stack_byte = 0; init_stack_byte <= 0xff; ++init_stack_byte)
            {
                cpu_->stack[cpu_->sp_ - 1] = (byte)init_stack_byte;
    
                for(int i = 1; i <= 0xff; ++i)
                {
                    struct CPU_ new_cpu;
                    memcpy(&new_cpu, cpu_, sizeof(struct CPU_));
    
                    int old_sp = new_cpu.sp_;
                    // Step
                    step[cmd[ip]](&new_cpu, i);
                    if(old_sp != new_cpu.sp_)
                    {
                        // A new value has been written on stack
                        if(new_cpu.stack[new_cpu.sp_ - 2] == dst_stack[new_cpu.sp_ - 2])
                        {
                            // Search the next step
                            char buf[40];
                            sprintf_s(buf, "sp = %d, init_stack = %x\n", 
                                new_cpu.sp_ - 2,
                                init_stack_byte);
                            WriteConsoleA(m_hConsole, buf, strlen(buf), NULL, NULL);
                            if(Search(cmd, ip + 1, &new_cpu, dst_stack, operation, final_ip, init_stack))
                            {
                                init_stack[new_cpu.sp_ - 2] = init_stack_byte;
                                operation[ip] = i;
                                return TRUE;
                            }
                        }
                    }
                    else
                    {
                        // No value is popped onto the stack
                        /*if(Search(cmd, ip + 1, &new_cpu, dst_stack, operation, final_ip))
                        {
                            operation[ip] = i;
                            return TRUE;
                        }*/
                        // return FALSE;
                    }
                }
            }
        }
    
    
        return FALSE;
    }
    
    VOID Process()
    {
        struct CPU_ cpu_;
        memset(&cpu_, 0, sizeof(cpu_));
    
        AllocConsole();
        m_hConsole = GetStdHandle(STD_OUTPUT_HANDLE);
    
        Initialize();
        WriteConsoleA(m_hConsole, "Initialization finished.\n", strlen("Initialization finished.\n"), NULL, NULL);
    
        prepare_teamname(&cpu_, "R3V3rZ3 I5 C00L");
        //prepare_teamname(&cpu_, "blue-lotus");
    
        char buf[2048];
        sprintf_s(buf, "CPU k_init = %08x %08x %08x %08x\n", 
            cpu_.k_init[0], 
            cpu_.k_init[1],
            cpu_.k_init[2],
            cpu_.k_init[3]);
        WriteConsoleA(m_hConsole, buf, strlen(buf), NULL, NULL);
    
        char src[] = "CTF_COOL";
        byte dst[8];
        tean(1, (byte*)src, dst, cpu_.k_init, 8);
        sprintf_s(buf, "Standard stack = %08x %08x\n", 
            *(unsigned int*)dst,
            *(unsigned int*)(dst + 4));
        WriteConsoleA(m_hConsole, buf, strlen(buf), NULL, NULL);
    
        byte k_flipped[16];
        byte data_flipped[16] = {0};
        for(int i = 0; i < 8; ++i)
        {
            k_flipped[i * 2] = ((byte*)cpu_.k_init)[i * 2 + 1];
            k_flipped[i * 2 + 1] = ((byte*)cpu_.k_init)[i * 2];
        }
    
        struct CPU_ new_cpu;
        int uFinalIp;
        byte init_stack[8];
        init_stack[7] = dst[7];
        memset(&new_cpu, 0, sizeof(struct CPU_));
        new_cpu.sp_ = 1;
        BOOL result = Search(k_flipped, 0, &new_cpu, dst, data_flipped, &uFinalIp, init_stack);
    
        sprintf_s(buf, "Result = %x, final_ip = %x\n",
            result,
            uFinalIp);
        WriteConsoleA(m_hConsole, buf, strlen(buf), NULL, NULL);
    
        byte data[25] = {0};
        // First 16 bytes
        for(int i = 0; i < 8; ++i)
        {
            data[i * 2] = data_flipped[i * 2 + 1];
            data[i * 2 + 1] = data_flipped[i * 2];
        }
        data[16] = (byte)uFinalIp;
        // 17 ~ 25 bytes
        for(int i = 17; i < 25; ++i)
        {
            data[i] = init_stack[i - 17];
        }
    
        // Convert it to keys
        char szAllowedChars[] = "ABCDEFGHIJKLMNOPQRSTUVWXYZ234567";
        char final_key[41] = {0};
        int pos = 0;
        int key_pos = 0;
        int bit_left = 0;
        int s = 0;
        while(pos < 25)
        {
            s = (s << 8) | data[pos];
            ++pos;
            bit_left += 8;
            while(bit_left >= 5)
            {
                int x = (s >> (bit_left - 5)) & 0x1f;
                final_key[key_pos ++] = szAllowedChars[x];
                bit_left -= 5;
            }
            s = s & 0x1f;
        }
        WriteConsoleA(m_hConsole, final_key, strlen(final_key), NULL, NULL);
    }
    
    BOOL WINAPI Inject(DWORD dwProcessID)
    {
        TCHAR strModulePath[2000] = {0};
        GetModuleFileName(m_hModule, strModulePath, 2000);
        HANDLE hProcess = OpenProcess(PROCESS_ALL_ACCESS, false, dwProcessID);
        FARPROC farLoadLibrary = GetProcAddress(GetModuleHandle(L"Kernel32.dll"), "LoadLibraryW");
        LPVOID lpDllAddr = VirtualAllocEx(hProcess, NULL, wcslen(strModulePath) * sizeof(TCHAR), MEM_COMMIT, PAGE_READWRITE); 
        if(lpDllAddr != NULL)
        {
            if(WriteProcessMemory(hProcess, lpDllAddr, strModulePath, wcslen(strModulePath) * sizeof(TCHAR), NULL))
            {
                HANDLE hT = CreateRemoteThread(hProcess, NULL, 0, (LPTHREAD_START_ROUTINE)farLoadLibrary, lpDllAddr, 0, NULL);   
                CloseHandle(hT);
                CloseHandle(hProcess);
                return TRUE;
            }
        }
        return FALSE;
    }