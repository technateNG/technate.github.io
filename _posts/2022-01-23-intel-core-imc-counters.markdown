---
layout: post
title:  "iMC counters on Intel Core - measuring memory bandwidth"
date:   2022-01-23 01:00:00 +0000
categories: perf
---

### *Why read iMC memory counters?*
Because there isn't better way to aquire correct measurements for iMC between two points of time.
Enabling these counters isn't straight forward but often worth an enffort when we are dealing with memory bound workload.

### *Measuring memory bandwidth on Intel Core processor (from Skylake to at least Alder Lake)*
Consumer class chips have only one memory controler and less options to configure Performance Monitoring Unit. 
But in exchange configuring and reading available counters is much easier than in Xeon series.  
Our source of information is [Uncore Performance Monitoring Reference Manual for 6th Generation Intel Core processors](https://www.intel.com/content/www/us/en/develop/download/6th-generation-intel-core-processor-family-uncore-performance-monitoring-reference-manual.html) manual. It's relevant for any Intel Core architecture since Skylake.  
For bandwith measurements we need to read two metrics: _DRAM_DATA_READ_ and _DRAM_DATA_WRITE_.  
Each measures adequate amount of CAS operations requested for DIMM's for read and write.
To obtain results in bytes we need to multiply every CAS operation by 64 (that's the amount of bytes which travels by memory bus per one request).
So for example, if memory controller issued 10 CAS requests that means that through memory bus traveled 640 bytes.  

To enable iMC counters we need to:  
a) Obtain BAR (Base Address Register) from PCI confguration space for device under 0:0.0 address.  
We need to do it only once for iMC. We can get it pogramatically in our program or simply by lspci command.  
b) Mmap system memory space - /dev/mem into our program with correct offset matching our BAR.  
c) Assign pointers to correct addresses in memory.

Here is working example of algorithm steps:
```
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/mman.h>

#define BAR_MASK 0x0007fffff8000
#define ADDITIONAL_OFFSET 0x5000

int main() {
    char config_buff[80] = { 0 };
    int config_fd = open("/sys/bus/pci/devices/0000:00:00.0/config", O_RDONLY);
    if (config_fd == -1) {
        return 1;  
    }
    ssize_t err = read(config_fd, config_buff, 80);
    if (err == -1) {
        return 1;
    }
    uint64_t *base_value = (uint64_t *)(config_buff + 72);
    uintptr_t imc_bar = (uintptr_t)(*base_value & BAR_MASK);
    printf("iMC BAR: %p\n", imc_bar);
    int system_memory_fd = open("/dev/mem", O_RDONLY);
    char *imc_space = (char *) mmap(NULL, 1024, PROT_READ, MAP_SHARED, system_memory_fd, imc_bar + ADDITIONAL_OFFSET);
    uint32_t *dram_data_reads = (uint32_t *)(imc_space + 0x50);
    uint32_t *dram_data_writes = (uint32_t *)(imc_space + 0x54);
}
```
To use iMC counters just read them in the same manner like typical clock values:
```
    uint32_t reads_start = *dram_data_reads;
    uint32_t writes_start = *dram_data_writes;
    ...HERE YOU WANT TO PLACE TARGET CODE..
    uint32_t reads_end = *dram_data_reads;
    uint32_t writes_end = *dram_data_writes;
    printf("RDCAS=%u WRCAS=%u\n", reads_end - reads_start, writes_end - writes_start);
```

Here is full working example with 1GiB allocation:
```
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/mman.h>

#define BAR_MASK 0x0007fffff8000
#define ADDITIONAL_OFFSET 0x5000
#define ONE_GIB (1ULL << 30ULL)

int main() {
    char config_buff[80] = { 0 };
    int config_fd = open("/sys/bus/pci/devices/0000:00:00.0/config", O_RDONLY);
    if (config_fd == -1) {
        return 1;  
    }
    ssize_t err = read(config_fd, config_buff, 80);
    if (err == -1) {
        return 1;
    }
    uint64_t *base_value = (uint64_t *)(config_buff + 72);
    uintptr_t imc_bar = (uintptr_t)(*base_value & BAR_MASK);
    printf("iMC BAR: %p\n", imc_bar);
    int system_memory_fd = open("/dev/mem", O_RDONLY);
    char *imc_space = (char *) mmap(NULL, 1024, PROT_READ, MAP_SHARED, system_memory_fd, imc_bar + ADDITIONAL_OFFSET);
    uint32_t *dram_data_reads = (uint32_t *)(imc_space + 0x50);
    uint32_t *dram_data_writes = (uint32_t *)(imc_space + 0x54);
    char *ptr = (char*) aligned_alloc(32, ONE_GIB);
    char *end_ptr = ptr + ONE_GIB;
    for (size_t i = 0; i < ONE_GIB; ++i) {
        ptr[i] = i;
    }
    uint32_t reads_start = *dram_data_reads;
    uint32_t writes_start = *dram_data_writes;
    while (ptr < end_ptr) {
        asm volatile (
                "vmovdqu ymm0, [%[ptr]];"
                "vmovdqu [%[ptr]], ymm0;"
                :
                : [ptr] "r"(ptr)
                : "ymm0"
                );
        ptr += 32;
    }
    uint32_t reads_end = *dram_data_reads;
    uint32_t writes_end = *dram_data_writes;
    printf("RDCAS=%u WRCAS=%u\n", reads_end - reads_start, writes_end - writes_start);
}
```
Compile with:
```
gcc -masm=intel -g -o imc_count_example imc_count_example.c
```
If everything was done correct than result should be similar to this:
```
DCAS=17003012 WRCAS=16741931
```
To get bytes we need to multiply these values by 64. After conversion to GiB we should see results close to 1.
This value is correct result - we loaded 1 GiB and write 1 GiB to RAM.
