---
layout: post
title:  "RDPMC - x86 benchmarking instruction"
date:   2020-02-16 01:00:00 +0000
categories: perf
---
#### *Benchmarking application on modern CPU's is hard.*

They have advanced beyond simple sequential predictability a long time ago with the introduction of Out of Order execution, branch prediction, power licenses, etc.
Multiple cores and SMT technology complicates this even further.
That's why we need the most precise tools and metrics to get a job done.  
It's not common knowledge that in our processors there is the whole set of metrics dedicated to benchmarking. 
Because unlocking PMC's isn't straightforward, I want to show why we want to use them and why often they are the best tool in our optimization toolbox.  
Our starting point is a non-trivial question: 

*What can we do when we need to measure the performance of a single function? Or to compare her efficiency with other implementation?* 

First, that comes to our mind is of course: 

### 1. Use profiler.
This is the best option when we need to make measurements in the whole application.
Tools like [Visual Studio Profiler](https://visualstudio.microsoft.com/), 
[Perf](https://perf.wiki.kernel.org/index.php/Main_Page) or [Intel VTune](https://software.intel.com/en-us/vtune) offer a variety of metrics and are free to use. 
But neither of them offers an option to measure only one function.  

I don't deny that maybe there is a supported way in some profilers - I couldn't find one. 

Anyway, we can hack our way to use profiler:  

**a) _Make a mock function you want to test._**  
**b) _Run profiler - these results are your noise._**  
**c) _Implement valid implementation A._**  
**d) _Run profiler - subtract results from your noise result._**  
**e) _Implement valid implementation B._**  
**f) _Run profiler - subtract results from your noise result._**  
**g) _Compare results._**  
**h) _GOTO b) as many times as you need to be confident._**

There is an excellent article about [this](https://medium.com/@veedrac/learning-the-value-of-good-benchmarking-technique-with-c-magic-squares-b61b3386c97f) approach.
A big advantage of this method is that we gain the whole power of profiler with all metrics and conveniences. 
A major disadvantage is a necessity of dealing with a noise factor, which can be brutal.

Any piece of non-determinism (syscall, malloc) increase noise, which means that your improvements need to be more and more significant to be statistically provable. This, unfortunately, makes the profiler approach very hard to use without separating the required function from the application code and put it into a clean deterministic environment (e.g. new program). Sometimes it’s possible, sometimes it’s not.

### 2. RDTSC
RDTSC instruction is the most frequent answer on stack overflow and is, unfortunately, half correct.  
TSC means the Time Stamp Counter. Its measurements represent the main clock of the processor.  
We can read it from userspace by `rdtsc` instruction like this:
```c
#define rdtsc(eax, edx) asm volatile ("rdtsc\n" : "=a"(eax), "=d"(edx)) 

size_t start;
unsigned int low_1, high_1;
unsigned int low_2, high_2;
rdtsc(low_1, high_1);
//...code to measure...
rdtsc(low_2, high_2);
size_t start = (size_t) low_1 | (size_t) high_1 << 32;
size_t end = (size_t) low_2 | (size_t) high_2 << 32;
size_t duration = end - start;
printf("Elapsed: %lld cycles\n", duration);
```
or we can use intrinsic:
```c
#include <immintrin.h>

__int64 start = _rdtsc();
//...code to measure...
__int64 end = _rdtsc();
printf("Elapsed: %lld cycles\n", end - start);
```
If we want to find out processor time elapsed in seconds, we simply need to divide the duration variable by speed of our processor. 
```
duration in cycles / speed of processor (in Hertz) = elapsed time
```

So what's not right with this approach?

No retired instructions, no mispredicted branches.  
But isn't that sufficient to evaluate, which implementation is better?

Unfortunately no.  
There is a potential problem with this metric. 
Core on which process is running can slow down.
Results of `rdtsc` can be incoherent between runs because of this issue.

### 3. RDPMC
#### 3.1 Model Specific Registers.
Every modern CPU with x86 architecture (since Intel Pentium) have Model Specific Registers (MSR's).  

There is a distinct set of MSR's for every CPU core. 
This means that if you have multi-core processor you need to specify on which core you need to operate.   
MSR's functions vary in dependence of register. Some of them control particular features of a CPU core, some give useful metrics.  
Besides their name plenty of them are standardized - in Intel nomenclature, they are called "architectural".   

Reading or writing to MSR needs the usage of special assembler instruction `wrmsr` (to write) and `rdmsr` (to read).  
Read for example:
```c
#define rdmsr(msr, high, low) asm volatile("rdmsr\n" : "a="(low), "d="(high) : "c"(msr))

unsigned int low, high;
unsigned int msr = 0x38f;
rdmsr(msr, high, low);
size_t result = (size_t) low | (size_t) high << 32;
printf("Value of msr 0x%x = %p\n", msr, result);
```
This snippet works only in kernel mode because wrmsr and rdmsr are ring 0 only instructions.
Fortunately for us - Linux has a way to expose MSR's into userspace.
There is a Linux Kernel Module (LKM) called `msr` and is available by default.  
We only need to do:
```sh
sudo modprobe msr
```
and LKM should expose MSR nodes for every core in path fulfilling regex   
`/dev/cpu/[0-9]+/msr`.  
If we try to directly read any of exposed nodes, we get garbage. We need to install an additional package provided by Intel called `msr-tools`.
On popular Linux distribution, there is a big chance that you have this package in your repository.  

For example, for Debian Buster, we only need to install the package:
```sh
sudo apt-get install msr-tools
```
If not, we can easily compile the package from sources:
```sh
git clone https://github.com/intel/msr-tools
cd msr-tools
mkdir build
cd build
../autogen.sh
make -j
```
After a successful installation, we should have two required ELF binaries `rdmsr` and `wrmsr` - the same names as assembler instructions.
With them, we can successfully read and write particular MSR's.  
Read:
```sh
sudo rdmsr -0 -c -a 0x38f
```
and write:
```sh
sudo wrmsr -a 0x38d 0xbbb
```
More information about their syntax you can get after running help:
```sh
sudo rdmsr -h
sudo wrmsr -h
```

MSR's are accessed as bitmasks - which means that every bit in the register is a switch to some other option.
Full list of MSR's with addresses for your processor you can find in [Intel x86/64 Manual](https://software.intel.com/en-us/download/intel-64-and-ia-32-architectures-software-developers-manual-volume-4-model-specific-registers) - Volume 4.

#### 3.2 Control Registers.
Another hardware processor registers which need a little description are Control Registers. Current x86 architecture processors have at least four such registers for every core. 
They are used to control the behavior of the CPU core. The main practical difference to MSR's is that they don't need special assembler instruction to access. 
We can read or write them by simple `mov` but same as MSR's only from kernel mode.  
In this example, we want to read CR4 to variable st_cr4:
```c
size_t st_cr4;
asm volatile(
        ".intel_syntax noprefix\n"
        "mov %0, cr4\n"
        ".att_syntax\n"
        : "=r"(st_cr4)
);
```
Basically in this guide, only a control register which is interesting for us is CR4. We access him with our own LKM later in the guide.

#### 3.3 Performance Monitor Counters (PMC's).

Performance Monitor Counters (PMC's) are a subset of MSR's. 
Their main purpose is to enhance the benchmarking experience by providing sufficient hardware-based metrics. 
With PMC's we have a full variety of metrics e.g retired instructions, mispredicted branches or L1 cache hit rate.
PMC's are quite standardized - they behave consistently between architectures and have standard revisions called "Architectural Performance Monitoring". 
Till this month (Feb 2020) there are five revisions and the most processors support up to the fourth standard. 
In most popular, fourth revision we have two types of counters:
* performance events  - **IA_PMCx**
* fixed counters - **IA32_FIXED_CTRx**

With the fourth revision of the standard, on most processors, we can choose at least four events from the pool. 
There is a variety of metrics to choose like L1 cache misses or branch misprediction rate - and many more. 
In opposite to events, fixed counters have a constant purpose and always measure the same thing. Our only choice is to enable or disable them.
We can interact with PMC's in two supported ways.  
First, like other MSR's, we can use the msr-tools package or instruction in kernel mode.
The second way is an `rdpmc` instruction which is by default in the Linux enabled only in a kernel mode.
We will enable `rdpmc` instruction to operate in the userspace later in the guide. 
As usual, PMC's are much more extensively explained in [Intel x86/64 Manual](https://software.intel.com/en-us/download/intel-64-and-ia-32-architectures-sdm-volume-3b-system-programming-guide-part-2) - Volume 3B in Chapter 18.

#### 3.4 How to enable CPU_CLK_UNHALTED.CORE.

CPU_CLK_UNHALTED.CORE describes elapsed cycles on the core.
In opposite, to raw TSC measure, this metric is resistant to changes in core speed. 
This is clear advantage over reading base TSC and in my opinion, it's best metric to measure a feature implementation efficiency in cycles.
This metric is fixed counter with index on one - IA32_FIXED_CTR1.

![d](/assets/d.png)

**a)** First, we need to enable correct metric. We need to modify the MSR called IA32_FIXED_CTR_CTRL_MSR on the address 0x38d.

![c](/assets/c.png)

To enable the desired counter we need to enable a required bits 4, 5 and 7. This gives us value 0x0b0. 
There is a high probability that there is enabled one fixed counter on our operating system by default. 
To avoid the risk of messing something up I recommend to enable all fixed counters.
This is a little bit easier, more portable and doesn't have any performance impact.  
To enable all fixed counters we need to write value 0xbbb to 0x38d MSR:
```sh
sudo wrmsr -a 0x38d 0xbbb
```
In the above snippet, we enable all fixed counters in IA32_FIXED_CTR_CTRL_MSR on every core.

**b)** Next, we need to enable fixed counters on IA32_PERF_GLOBAL_CTR_MSR which is a global MSR switch for PMC's. We find it on 0x38f address.

![a](/assets/a.png)

We need to switch to one bits on 32, 33 and 34 position.
```sh
sudo wrmsr -a 0x38f 0x700000000
```
To enable all fixed counters.

**c)** We can now enumerate enabled fixed registers on addresses 0x309 - 0x30b:
```sh
sudo rdmsr -a 0x30a
```
Now you should see nonzero numbers as result. These are CPU_CLK_UNHALTED.CORE (IA32_FIXED_CTR1) for all cores.  
If you see 0 on every counter then GOTO FAQ below.

#### 3.5 How to enable L1D_PEND_MISS.PENDING.

Metrics which are performance events are a little bit more difficult to enable.
In case of performance events, we need to choose the correct one from the event pool which is defined in [Intel x86/64 Manual](https://software.intel.com/en-us/download/intel-64-and-ia-32-architectures-sdm-volume-3b-system-programming-guide-part-2) - in Chapter 19.
For this example we choose L1D_PEND_MISS.PENDING which describes L1 cache misses.

**a)** First we need to get our desired metric event number and umask value from the intel manual.
In my case (i have Kaby Lake Processor) my event number is 0x48 and umask is 0x1.

![e](/assets/e.png)

**b)** Secondly, we want to set our performance event up. 
We need to enable the first bit in IA32_PERF_GLOBAL_CTR_MSR but before that, we need to confirm that we won't break anything in our operating system.
To do that we want to enable only the first bit but won't change anything which is enabled right now.
First, we need to read IA32_PERF_GLOBAL_CTR_MSR:
```sh
sudo rdmsr -a 0x38f
```
When we get our value it should look like this:
```
0x700000000
```
We simply need to set the first bit to this form:
```
0x700000001
```
and then:
```sh
sudo wrmsr -a 0x38f 0x700000001
```
 
**c)** Modify IA32_PERFEVTSELx MSR where x is an index of our PMC. In our case, it's 0 so our MSR address is 0x186.
![b](/assets/b.png)

With values which we found in step 'a' we should get a number similar to this:
```
0x00430148 // Binary: 0b10000110000000101001000 
```
0x0148 - is our event number and umask.
0x0043 - means that we enable counter and is available in ring 0 and 3.

```sh
wrmsr -a 0x186 0x00430148
```

**d)** It's done. Now let's check that's all fine.
```sh
sudo rdmsr -0 -c -a 0x186
sudo rdmsr -0 -c -a 0xc1
```
We should see non zero numbers in both cases. If not GOTO FAQ below.

#### 3.6 How to enable RDPMC instruction in user space?

We can enable it by echoing two to `/sys/devices/cpu/rdpmc`:
```
sudo sh -c 'echo 2 > /sys/devices/cpu/rdpmc'
```

#### 3.7 How to use RDPMC

Same as with `rdtsc` we have two choices.
In assembly:
```c
#define rdpmc(pmc, eax, edx) asm volatile ("rdpmc\n" : "=a"(eax), "=d"(edx) : "c"(pmc)) 

size_t start;
unsigned int low_1, high_1;
unsigned int low_2, high_2;
unsigned int pmc = 0x40000000 + 2;
rdpmc(pmc, low_1, high_1);
//...code to measure...
rdpmc(pmc, low_2, high_2);
size_t start = (size_t) low_1 | (size_t) high_1 << 32;
size_t end = (size_t) low_2 | (size_t) high_2 << 32;
size_t duration = end - start;
printf("Elapsed: %lld cycles\n", duration);
```
or we can use intrinsic:
```c
#include <immintrin.h>

unsigned int pmc = 0x40000000 + 2;
__int64 start = _rdpmc(pmc);
//...code to measure...
__int64 end = _rdpmc(pmc);
printf("Elapsed: %lld cycles\n", end - start);
```
Values of variable pmc are fixed and rather won't change in the near future.  

| PMC             | value         | 
|:---------------:|:-------------:|
| IA32_FIXED_CTR0 |  0x40000000   | 
| IA32_FIXED_CTR1 |  0x40000001   |
| IA32_FIXED_CTR2 |  0x40000002   |
| IA32_PMC0       |  0x0          |
| IA32_PMC1       |  0x1          |
| IA32_PMC2       |  0x2          |
| IA32_PMC3       |  0x3          |
| IA32_PMC4       |  0x4          |

### FAQ
#### 1. Are PMC's works on AMD processors?
Yes, an AMD processors have their own PMC's. I haven't ever had AMD processor so I don't have experience with them.
#### 2. Do profilers use PMC's?
Yes, they are in use by every major profiler.
#### 3. Msr-tools don't work. What I can do?
Are you sure that you aren't inside VM? In VM everything depends on your hypervisor. Some MSR's can work but, by default, I assume that they don't.
If you aren't inside VM then check MSR address in manual. Maybe there is a mistake.
#### 4. RDPMC instruction rises the SEGFAULT signal.
Did you enabled it to work in userspace?
