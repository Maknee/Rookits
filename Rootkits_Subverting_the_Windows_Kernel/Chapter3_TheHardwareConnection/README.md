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


