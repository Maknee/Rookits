Ring0 instructions to manipulate hardware

cli

sti

in

out

# CPU Tables

Global Descriptor Table (GDT), used to map addresses

Local Descriptor Table (LDT), used to map addresses

Page Directory, used to map addresses

Interrupt Descriptor Table (IDT), used to find interrupt handlers

# OS Tables

System Service Dispatch Table (SSDT), used by the Windows OS for handling system calls

# Memory pages

Lookup for pages occurs in steps: Descriptor check, Page Directory check, Page check, access read/write

Descriptor (or segment) check: 

Descriptor privilege level (DPL). The DPL contains the ring number (zero to three) required of the calling process. 

If the DPL requirement is lower than the ring level (sometimes called the current privilege level [CPL]) for the calling process, access is denied, and the memory check stops here.

Page directory check:
A user/supervisor bit is checked for an entire page table—that is, an entire range of memory pages. 

If the user/supervisor bit is set to zero, then only “supervisor” programs (Rings Zero, One, and Two) can access the range of memory pages; if the calling process is not a “supervisor,” the memory check stops here. 

If the user/supervisor bit is set to 1, then any program can access the range of memory pages.

Page check:
same as page directory check

# GDT does not provide any security

The first four entries (08, 10, 1B, and 23) encompass the entire range of memory for data and code, and both Ring Zero and Ring Three programs. The result is that the GDT does not provide any security for the system.

Thus, security is enforced using paging

# Paging

Memory pages can be marked as paged out (that is, stored on disk rather than in RAM). 

The processor will interrupt when any of these memory pages is sought. 

The interrupt handler reads the page back into memory, making it paged in.

Whenever a program reads memory, it must specify an address. For each process, this address must be translated into an actual physical memory address. 

The following steps are taken by the operating system and the CPU in order to translate a requested virtual address into a physical memory address:

The CPU consults CR3 to find the base of the page-table directory.

The requested memory address is split into three parts, as shown in Figure 3-5.

The top 10 bits are used to find the location in the page-table directory (see Figure 3-4).

Once the page-directory entry is located, the corresponding page table is found in memory.

The middle 10 bits of the address are used to find the index in the page table (see Figure 3-4).

The corresponding physical memory address (sometimes called the physical page frame) is found for the page.

The bottom 12 bits of the requested address are used to locate an offset in the physical page-frame memory (up to 4096 bytes). The resulting actual physical address contains the requested data.

# Note

On Windows XP and greater, the memory pages containing the SSDT and IDT are set to read-only in the page table. If an attacker wishes to alter the contents of these memory pages, she must first change the pages to read/write. The best way for a rootkit to do this is called the CR0 trick, described later in this chapter

# Processes and Threads

Rootkit developers should understand that the primary mechanism for managing running code is the thread, not the process. The Windows kernel schedules processes based on the number of threads, not processes.

That is, if there are two processes, one single-threaded and the other with nine threads, the system will give each thread 10% of the processing time. The single-threaded process would get 10% of the CPU time, while the process with nine threads would get 90%. 

Just what is a process? Under Windows, a process is simply a way for a group of threads to share the following data:

virtual address space (that is, the value used for CR3)

access token, including SID[5]

handle table for win32 kernel objects

working set (physical memory “owned” by the process)

When a context switch to a new thread occurs, the old thread state is saved. Each thread has its own kernel stack, so the thread state is pushed onto the top of the thread kernel stack. If the new thread belongs to a different process, the new page directory address for the new process is loaded into CR3. The page directory address can be found in the KPROCESS structure for the process. Once the new thread kernel stack is found, the new thread context is popped from the top of the new thread kernel stack, and the new thread begins execution. If a rootkit modifies the page tables of a process, the modifications will be applied to all threads in that process, because the threads all share the same CR3 value.

The Global Descriptor Table
A number of interesting tricks may be implemented via the GDT. The GDT can be used to map different address ranges. It can also be used to cause task switches. The base address of the GDT can be found using the SGDT instruction. You can alter the location of the GDT using the LGDT instruction.

The Local Descriptor Table
The LDT allows each task to have a set of unique descriptors. A bit known as the table-indicator bit can select between the GDT and the LDT when a segment is specified. The LDT can contain the same types of descriptors as the GDT.

Code Segments
When accessing code memory, the CPU uses the segment specified in the code segment (CS) register. A code segment can be specified in the descriptor table. Any program, including a rootkit, can change the CS register by issuing a far call, far jump, or far return, where CS is popped from the top of the stack.[6] It is interesting to note that you can make your code execute only by setting the R bit to zero in the descriptor.

Call Gates
A special kind of descriptor, called a call gate, can be placed in the LDT or the GDT. A program can make a far call with the descriptor set to the call gate. When the call occurs, a new ring level can be specified. A call gate could be used to allow a user-mode program to make a function call into kernel mode. This would be an interesting back door for a rootkit program. The same mechanism can be used with a far jump, but only when the call gate is of the same privilege level or lower than process performing the jump.[7]

When a call gate is used, the address is ignored—only the descriptor number matters. The call gate data structure tells the CPU where the code for the called function lives. Optionally, arguments can be read from the stack. For example, a call gate could be created such that the caller puts secret command arguments onto the stack.


The Interrupt Descriptor Table
The interrupt descriptor table register (IDTR) stores the base (the start address) of the interrupt descriptor table (IDT) in memory. The IDT, used to find the software function employed to handle an interrupt, is very important.[8] Interrupts are used for a variety of low-level functions in a computer. For example, an interrupt is signaled whenever a keystroke is typed on the keyboard.

The IDT is an array that contains 256 entries—one for each interrupt. That means there can be up to 256 interrupts for each processor. Also, each processor has its own IDTR, and therefore has its own interrupt table.

When an interrupt occurs, the interrupt number is obtained from the interrupt instruction, or from the programmable interrupt controller (PIC). In either case, the interrupt table is used to find the appropriate software function to call. This function is sometimes called a vector or interrupt service routine (ISR).

One trick employed by rootkits is to create a new interrupt table. This can be used to hide modifications made to the original interrupt table. 

```C
/* sidt returns idt in this format */
typedef struct
{
   unsigned short IDTLimit;
   unsigned short LowIDTbase;
   unsigned short HiIDTbase;
} IDTINFO;
```

Using the data provided by the SIDT instruction, an attacker can then find the base of the IDT and dump its contents.

```C
// entry in the IDT: this is sometimes called
// an "interrupt gate"

#pragma pack(1)
typedef struct
{
      unsigned short LowOffset;
      unsigned short selector;
      unsigned char unused_lo;
      unsigned char segment_type:4; //0x0E is interrupt gate
      unsigned char system_segment_flag:1;
      unsigned char DPL:2;     // descriptor privilege level
      unsigned char P:1;       // present
      unsigned short HiOffset;
} IDTENTRY;
#pragma pack()
```

To access the IDT, use the following code example as a guide:
```C
#define MAKELONG(a, b)
((unsigned long) (((unsigned short) (a)) | ((unsigned long) ((unsigned short) (b))) << 16))
```
