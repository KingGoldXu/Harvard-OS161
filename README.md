# Harvard-OS161
<h4>The code in this repository are my solutions for ECE344 labs at the University of Toronto. Please do not use it for your own course work, thank you.</h4> 
<h4>Feel Free to email me if you have any questions about OS programming. We got full marks for 0-2 Assignment. Last assignment, we can pass 80 testcases.
<a href="mailto:chuanrui.li@mail.utoronto.ca?Subject=CSC343" target="_top">Send me a mail</a></h4> 
<h3>Assignment 0: An Introduction to OS161 </h3>
<p>easy</p>

<h3>Assignment 1: Synchronization</h3>
<p>Objectives:</p>
<ul>
<li>Understand how OS161 implements semaphores.</li>
<li>Be comfortable developing/implementing synchronization primitives.</li>
<li>Be able to select an appropriate synchronization primitive for a given problem.</li>
<li>Be able to properly synchronize different types of problems.</li>
</ul>
<p>Tips: 1, Implement the lock and CV first. 2, Think about the deadlock cases</p>
<p>Suggestions: Cat problem - using locks, Intersection problem - using semaphore</p>
<h3>Assignment 2: System Calls and Processes</h3>
<p>Objectives:</p>
<ul>
<li>Understand how OS161 implements semaphores.</li>
<li>Understand how to represent processes in an operating system.</li>
<li>Design and implement data structures to manage processes in an OS.</li>
<li>Understand how to implement system calls.</li>
<li>Understand how to use synchronization in OS code.</li>
</ul>
<p>Suggestions: starting point: syscall.c and Implement one-to-one process and thread data structure first</p>
<p>System call implementation sequence: write, getpid, waitpid, exit, fork, runprogram, read, execv</p>

<h3>Assignment 3:  Virtual Memory</h3>
<p>Objectives:</p>
<ul>
<li>Become familiar with OS handling of TLB and page faults.</li>
<li>Develop a solid intuition about virtual memory and what it means to support it.</li>
</ul>
<h2>Suggestions: Implementation Plan</h2>


<p>The virtual memory system is one of the most complicated components of any operating system.  Even though OS161 is a rather simple OS, this is still significantly more difficult than any of the labs you have done so far.  To help you with this we've divided this assignment into several steps for you to complete.   You should completely test each step before moving onto the next step.  In addition, after implementing a step, it is good practice to go back and re-run all your previous tests to make sure you haven't broken any of your old code!  Finally, once you start implementing this code, you will find that any memory errors you cause in the kernel will cause a TLB miss and caught by your own fault handling code, resulting in very confusing output.  Remember to refer to the 10 commandments of OS161 programming when you feel lost!</p>
<h5>1. Managing kernel memory with a Coremap</h5>

<p>For now, copy dumbvm.c as your addrspace.c as a starting point.  Study and understand the functions getppages, ram_stealmem, alloc_kpages and free_kpages.  In dumbvm, getppages calls ram_stealmem to allocate memory, but memory allocated by ram_stealmem cannot be marked free once allocated. You must implement code that will keep track of the memory used by the kernel.  One could do this with a bitvector, but in reality a coremap is much more useful.  A coremap keeps track of which pages are in use and also keeps track of which address spaces a page is mapped into.  Remember that a page can be mapped into more than one address space (why?).  As you will see later, this information is useful when we implement swap and copy-on-write pages.  Your task in this section is to implement tracking of which memory pages are allocated and which ones aren't, as well as track which process is using which pages using a coremap.  You should modify getppages and free_kpages to call your coremap functions instead of using ram_stealmem.  Note that you should put any initialization code for your coremap in vm_bootstrap.  One tricky problem is what if your initialization code needs to allocate memory for your coremap -- how should you allocate that memory if your coremap is not setup yet? 
Your paging system will also need to support page allocation requests generated by kmalloc(). You should review kmalloc() to understand how these requests are generated, so that your system will respond to them correctly. Currently, pages that are allocated to kmalloc are not returned to the system. You can use the core map to correctly implement allocated and freed pages.</p>

<h5>2. On-demand memory allocation</h5>
<p>Currently when a process created, all memory is allocated up front by as_prepare_load.  Your next task is to modify the memory management code to allocate memory pages the first time your program touches them.  First, read and understand the dumbvm implementation of as_prepare_load, as_define_region, as_activate, as_create and as_destroy.  You should also know what functions call these functions and why so you understand what these functions are expected to do.  Next, you will re-implement all these functions, as so that pages are only allocated in vm_fault, which handles all TLB faults.  Note, you will likely have to enhance the definition of struct addrspace.

You will need to guarantee that the TLB state is initialized properly on a context switch. One implementation alternative is to invalidate all the TLB entries on a context switch. The entries are then re-loaded by taking TLB faults as pages are referenced. We recommend that you separate implementation of the TLB entry replacement algorithm from the actual piece of code that handles the replacement. This will make it easy to experiment with different replacement algorithms if you wish to do so. Finally, currently execv loads all program pages when the program is started.  Replace this implementation so that pages are loaded on demand.

One thing to watch out for is to make sure that as_destroy actually frees all the memory allocated by a process.  To ensure this is to enhance your coremap implementation with a counter that counts the overall number of allocated pages.  Add debugging code that prints this counter out on each as_create and as_destroy.  If the number of pages keeps on going up everytime you run a program, then you have a memory leak, where you are allocating memory somewhere and forgetting to de-allocate it later.  If you do not fix this, your OS will graduatlly lose use of all its memory -- which is not a good thing!  

After you implement this component, you should be able to run the matmult, sort, stacktest and palin tests.  You may also try to run huge, but you will probably run out of memory depending on your configuration.  This is a good way to test whether your code correctly detects when you have run out of memory!  By the end of this section, you should be able to handle all four of these tests correctly.</p>

<h5>3. Fork and as_copy</h5>

<p>Any user program that calls fork will fail becuase we have not implemented as_copy at this point.  Now you will implement as_copy to make a copy of the current address space.  Note that you should only copy the data structures of the address space that exist in the kernel.  You should not copy any user pages!  Instead, you must implement copy-on-write and make the pages read-only while they are mapped into more than one address space.  Only when one a process wants to modify the page, should you make a copy.  Thus, you must check the reason for faults in vm_fault and handle the fault appropriately (Hint: use the capability of your coremap impelmentation to keep track of which address spaces a page is mapped into to figure out whether the page is copy-on-write).

After you implement this component, you should be able to run tests that call fork.  This includes tests such as triplemat and triplesort, as well as any other tests you used previously to test fork.  triplehuge may work, but you will likely run out of memory until swap is working.</p>
<h5>4. Implementing the sbrk system call.</h5>
<p>
Currently, user programs that use malloc or free will fail because you have not implemented heap management within the kernel. Heap management within the kernel is embodied in the sbrk system call.  Read the man page for the sbrk system call (in man/syscall/sbrk.html) as well as understand the code that calls sbrk in the implementation of malloc in libc (in lib/libc/malloc.c),

After this, you should be able to successfully run the tests in malloctest.</p>
<h5>5. Swap and Paging</h5>

<p>Currently, if you run out of physical ram, it's likely that you will just panic the kernel.  Instead, you want to swap to disk when there is not enough ram.  The default sys161.conf includes two disks; you can use one of those disks for swapping. Please do swap to a disk and not somewhere else (such as a file). Also, be sure not to use that disk for anything else! To help prevent errors or misunderstandings, please have your system print the location of the swap space when it boots. You will have to figure out how to use the swap device (hint: use vfs_open and VOP_STAT to initislize the swap device, and mk_kuio, VOP_READ and VOP_WRITE to read and write from the swap device).  You will also need to keep track of which portions of the swap device are in use and which are free.  Remember that any initialization code you write for the swap device can get called in vm_boostrap.  Finally, remember to gracefully handle the case when your application runs out of swap (i.e. kill the process and clean up properly). 

When a page is evicted, you must remove all page table mappings to it since the page is no longer in memory.    You will find your coremap very handy for this operation. As in previous sections, print out the count of allocated pages of swap.  If this number continually increases, you have a memory leak in swap!  After you complete this component, you shoud be able to run the huge and triplehuge tests.  Note for triplehuge, you may need to increase the kernel stack size.</p>
<h5>6. Instrumentation and Tuning</h5>

<p>In this section, we ask you to tune the performance of your virtual memory system. Your first step is to ensure that you have fine-grain locks.  You should only be using splhigh and splx to ensure that you do not get an interrupt while modifying the TLB as your TLB could be inconsistent when the interrupt arrives.  Other than that, all shared data structures should be protected by their own locks to maximize concurrency.  The TAs will be looking for inefficient locking in your virtual memory implementations.

Next, you want to tune your memory system to minimize the number of TLB misses and page faults.  As a start, we suggest that you implement the following counters:

The number of page faults.
The number of TLB faults.
The number of page faults that require a write of a page.
You should add the necessary infrastructure to maintain these statistics as well as any other statistics that you think you will find useful in tuning your system.  You can minimize the number of writes during page fault handling by implementing a thread in the kernel that detects when the amount of free memory is getting low and pre-emptively evicts memory pages to swap.  Your page replacement algorithm can also preferentially pick pages that are read-only to evict, since these do not need to be written to disk.

Finally, you should verify that your memory system is not leaking memory.  The count of allocated pages should return to essentially zero when the test programs complete.  Note that if you are using arrays, there may be a small number of pages left over that do not get deallocated, but in general the number of used pages should not constantly increase.
Once you have completed all the problems in this assignment and added instrumentation, it is time to tune your operating system. Use the file performance.txt as your "lab notebook" for this section.

At a minimum, use the matmult and sort programs provided to determine your baseline performance (the performance of your system using the default TLB and paging algorithms). Experiment with different TLB and paging algorithms and parameters in an attempt to improve your performance. As before, predict what will happen as you change algorithms and parameters. Compare the measured to the predicted results; obtaining a different result from what you expect might indicate a bug in your understanding, a bug in your implementation, or a bug in your testing. Figure out where the error is! We suggest that you tune the TLB and paging algorithms separately and then together. You may not rewrite the matmult program to improve its performance.

You should add other test programs to your collection as you tune performance, otherwise your system might perform well at matrix multiplication and sorting, but little else. Try to introduce programs with some variation in their uses of the memory system.</p>

