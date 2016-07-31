# `start_kernel` 함수
> Google Docs를 기반으로 정리 진행 중입니다 :)

#### 노트
__weak(함수 덮어쓰기).  

[Xref](http://lxr.free-electrons.com/source/init/main.c#L479)

`kernel/setup.c`에 있는 코드가 실행될 것으로 예상됨
arm64 에서 코드를 확인할 필요 있음


##   > `set_task_stack_end_magic(&init_task);` 
- `init_task` stack 마지막 영역에 magic code 삽입

##   > `smp_setup_processor_id`
##   > `debug_objects_early_init`
##   > `boot_init_stack_canary`


##   > `cgroup_init_early`
#### 노트
> Cgroup은 프로세스들을 그룹화하여 그룹에게 자원을 분배하거나 
제어하는 기능을 제공해준다.  - 모기향 118p  

분할 초기화 : `cgroup_init()`에서 한번 더 초기화 과정을 거친다.

#### 참조
https://www.kernel.org/doc/Documentation/cgroup-v1/cgroups.txt

#### 노트
프로세서의 인터럽트 중지


##   > `local_irq_disable`;
```c
early_boot_irqs_disabled = true;
```

##   > `boot_cpu_init`
Activate the first processor.
Interrupts are still disabled. Do necessary setups, then enable them 

첫번째 프로세서 활성화


##   > `page_address_init`
페이지들에 대한 hash table 초기화 (spin lock이 있는 linked bucket 구조)


##   > `setup_arch`
아키텍처 셋업. 내용이 많음
Boot argument를 parse를 비롯해 다수의 처리

#### 참조

http://jake.dothome.co.kr/boot_cpu_init/

##   > `mm_init_cpumask`
```c
mm_init_cpumask(&init_mm)
```

##   > `setup_command_line`
```c
    setup_command_line(command_line)  
```

We need to store the untouched command line for future reference. We also need to store the touched command line since the parameter parsing is performed in place, and we should allow a component to store reference of name/value for future reference.


##   > `setup_nr_cpu_ids`
CPU 최대 개수 저장 : NR

##   > `setup_per_cpu_areas`
CPU마다 각각 per-CPU data를 초기화. Area에 대해서는 명확하지 않음


##   > `boot_cpu_state_init`

```c
/*
* Must be called _AFTER_ setting up the per_cpu areas
*/
void __init boot_cpu_state_init(void)
{
   per_cpu_ptr(&cpuhp_state, smp_processor_id())->state 
        = CPUHP_ONLINE;
}
```

##   > `smp_prepare_boot_cpu`
arch-specific boot-cpu hooks

[/arch/arm64/kernel/smp.c](http://lxr.free-electrons.com/source/arch/arm64/kernel/smp.c#L438)

##   > `build_all_zonelists`
Called with zonelists_mutex held always unless 
`system_state == SYSTEM_BOOTING`. 

> CH17 메모리를 빌려줄 후원자 구성하기 - 모기향

[/mm/page_alloc.c](http://lxr.free-electrons.com/source/mm/page_alloc.c#L5029)


##   > `page_alloc_init`

```c
void __init page_alloc_init(void)
{
    // Hot plugin 할 때 notifier를 등록하는 코드?
    hotcpu_notifier(page_alloc_cpu_notify, 0);
}
```
[/mm/page_alloc.c](http://lxr.free-electrons.com/source/mm/page_alloc.c#L6726)

> 모기향  
> `hotcpu_notifier` : cpu hot-plugin 시 호출되는 callback 함수 등록
> `page_alloc_cpu_notify`
>   - cpu dead/dead-frozen 일 경우에 해당(문제가 발생한) CPU 가 관리하던 page 를 프리리스트(free-list 인 것으로 사료됨)로 옮
>   - 해당(문제가 발생한) CPU 의 page 관련 이벤트들을 현재(`page_alloc_cpu_notify`를 수행하는) CPU 에서 사용할 수 있도록 관련 내용을 옮겨준다.
>   - hot plug-in 된 CPU 의 zone 의 `per_cpu_pageset` 구조체 정보를 가져와 업데이트 해 준다.


##   > `parse_early_param`

Arch code calls this early on, or if not, just before other parsing.
kernel parameter 초기 파싱. 디버그/장치 관련 목적으로 사용

###   > `parse_early_param` > `parse_early_options`


###   > `parse_early_param` > `parse_early_options` > `parse_args`

> [문c 블로그](http://jake.dothome.co.kr/parse_args/)  
> cmdline 인수로 받은 파라메터를 param = value 형태로 다듬고 구형 커널 파라메터 블럭(`__setup_start` ~ `__setup_end`) 에서 매치된 파라메터 중 early가 아닌 경우 해당 파라메터에 연결된 설정 함수를 호출한다. 또한 모듈(modprobe) 관련 파라메터가 있는 경우 일단 무시한다. 그 외에 매치되지 않은 unknown 파라메터는 값이 없는 경우 `envp_init[]` 배열에 추가하고 값이 있는 경우 `argv_init[]` 배열에 추가한다

[`parse_early_param`](http://jake.dothome.co.kr/parse_early_param/)

[/kernel/params.c](http://lxr.free-electrons.com/source/kernel/params.c#L216)  
인자를 분석하고 Callback 함수를 호출

#### 참조
`parse_args()` : callback = `do_early_param()`  

`parse_args()` : callback = `unknown_bootoption()`  

`parse_args()` : callback = `set_init_arg()`  

`do_early_param()` : early_param 매크로에 의해 등록된 함수들 실행 http://lxr.free-electrons.com/ident?v=4.6&i=early_param  

`unknown_bootoption()` : jakhome


`set_init_arg()` : 나머지 인자는 init process 에 전달하도록 준비

```c
/* Anything after -- gets handed straight to init. */
static int __init set_init_arg(char *param, char *val,
                               const char *unused, void *arg)
```


##   > `jump_label_init`
[/kernel/jump_label.c](http://lxr.free-electrons.com/source/kernel/jump_label.c#L231)  
[문c 블로그](http://jake.dothome.co.kr/jump_label_init/)  
[Kernel Documentation : Static Key](https://www.kernel.org/doc/Documentation/static-keys.txt)  

##   > `setup_log_buf`
함수 내용 없음
##   > `pidhash_init`
PID 해시 테이블 만들기  
The pid hash table is scaled according to the amount of memory in the  machine. From a minimum of 16 slots up to 4096 slots at one gigabyte or more.

##   > `vfs_caches_init_early`
> [문c 블로그](http://jake.dothome.co.kr/vfs_caches_init_early/)
> PID 테이블을 해쉬로 구현하여 빠르게 검색하고 처리할 수 있도록 하는데 이 함수를 사용하여 초기화 한다.

```c
void __init vfs_caches_init_early(void)
{
        dcache_init_early();
        inode_init_early();
}
```

[`/fs/decache.c`](http://lxr.free-electrons.com/source/fs/dcache.c#L3700)



##   > `sort_main_extable`
Main Exception Table을 정렬한다. Exception Table은 Main과 Module 2가지로 나뉜다.

> 모기향  
> 실제 Exception Table Entry는 코드 여러곳에 분산되어 있을 수 있으므로, 이들을 모아서 정렬하고, 찾아서 사용한다. 

```c
/* Sort the kernel's built-in exception table */
void __init sort_main_extable(void)
{
    if (main_extable_sort_needed 
        && __stop___ex_table > __start___ex_table) 
    {
        pr_notice("Sorting __ex_table...\n");
        sort_extable(__start___ex_table, __stop___ex_table);
    }
}
```
[문c 블로그 : `sort_main_extable`](http://jake.dothome.co.kr/sort_main_extable/)  
[문c 블로그 : `exception_table`](http://jake.dothome.co.kr/exception-table/)  


##   > `trap_init`
[/arch/arm64/kernel/traps.c](http://lxr.free-electrons.com/source/arch/arm64/kernel/traps.c#L569)

```c
/* This registration must happen early, before debug_traps_init(). */
void __init trap_init(void)
{
    register_break_hook(&bug_break_hook);
}
```

`bug_break_hook`는 커널 패닉 발생시 화면에 report를 출력하고 커널을 멈추는 bug handler로 추정. 


##   > `mm_init`
[/init/main.c](http://lxr.free-electrons.com/source/init/main.c#L464)

```c
/*
* Set up kernel memory allocators
*
* `page_ext` requires contiguous pages,
* bigger than MAX_ORDER unless SPARSEMEM.
*/
static void __init mm_init(void)
{
    page_ext_init_flatmem();
    mem_init();
    kmem_cache_init();
    percpu_init_late();
    pgtable_init();
    vmalloc_init();
    ioremap_huge_init();
}
```

###   > `mm_init` > `page_ext_init_flatmem`
[/include/linux/page_ext.h](http://lxr.free-electrons.com/source/include/linux/page_ext.h#L85)

```c
// CONFIG_SPARSEMEM 이 선언되지 않은 경우에만 사용
// defconfig : CONFIG_SPARSEMEM=y

#ifdef CONFIG_SPARSEMEM
static inline void page_ext_init_flatmem(void)
{
    // 내용 없음
}
extern void page_ext_init(void);
#else
extern void page_ext_init_flatmem(void);
static inline void page_ext_init(void)
{
}
#endif
```

```c
 void __init page_ext_init_flatmem(void)
 {
         int nid, fail;
 
         if (!invoke_need_callbacks())
                 return;
 
         for_each_online_node(nid)  {
                 fail = alloc_node_page_ext(nid);
                 if (fail)
                         goto fail;
         }
         pr_info("allocated %ld bytes of page_ext\n", total_usage);
         invoke_init_callbacks();
         return;
 
 fail:
         pr_crit("allocation of page_ext failed.\n");
         panic("Out of memory");
 }
```

###   > `mm_init` > `mem_init`


`mem_init()` marks the free areas in the mem_map and tells us how much memory is free.  This is done after various parts of the system have claimed their memory after the kernel image.

```c
 void __init mem_init(void)
 {
         swiotlb_init(1);
 
         set_max_mapnr(pfn_to_page(max_pfn) - mem_map);
 
 #ifndef CONFIG_SPARSEMEM_VMEMMAP
         free_unused_memmap();
 #endif
         /* this will put all unused low memory onto the freelists */
         free_all_bootmem();
 
         mem_init_print_info(NULL);
 
 #define MLK(b, t) b, t, ((t) - (b)) >> 10
 #define MLM(b, t) b, t, ((t) - (b)) >> 20
 #define MLG(b, t) b, t, ((t) - (b)) >> 30
 #define MLK_ROUNDUP(b, t) b, t, DIV_ROUND_UP(((t) - (b)), SZ_1K)
        // ...
 #undef MLK
 #undef MLM
 #undef MLK_ROUNDUP
 
         /*
          * Check boundaries twice: Some fundamental inconsistencies can be
          * detected at build time already.
          */
 #ifdef CONFIG_COMPAT
         BUILD_BUG_ON(TASK_SIZE_32  > TASK_SIZE_64);
 #endif
 
         /*
          * Make sure we chose the upper bound of sizeof(struct page)
          * correctly.
          */
         BUILD_BUG_ON(sizeof(struct page) > (1 << STRUCT_PAGE_MAX_SHIFT));
 
         if (PAGE_SIZE >= 16384 && get_num_physpages() <= 128) {
                 extern int sysctl_overcommit_memory;
                 /*
                  * On a machine this small we won't get anywhere without
                  * overcommit, so turn it on by default.
                  */
                 sysctl_overcommit_memory = OVERCOMMIT_ALWAYS;
         }
}
```

lowmem 및 highmem에 대한 모든 free memblock 영역에 대해 Buddy memory allocator의 빈 페이지로 이관 등록한다. memblock을 더 이상 사용하지 않는 경우 reserved & memory memblock 관리배열까지 Buddy로 free 시킨다.




###   > `mm_init` > `kmem_cache_init`

> [문c 블로그]  
> 커널이 부트업 프로세스 과정에서 slub 메모리 할당자를 활성화 하기 위해 다음 그림과 같이 먼저 캐시 초기화 과정을 수행한다.

[Slab, Slob, Slub](http://events.linuxfoundation.org/sites/events/files/slides/slaballocators.pdf) 할당자  
[Xref](http://lxr.free-electrons.com/ident?i=kmem_cache_init)

`slab.c`, `slob.c`, `slub.c` 3개 파일에 모두 선언되어 있음. 기본 설정에서는 slub 사용.

```
// 정의부 주석
/*
 * struct kmem_cache related prototypes
 */
void __init kmem_cache_init(void);

// 구현부 주석 - slab.c
/*
 * Initialisation.  Called after the page allocator have been initialised and
 * before smp_init().
 */
void __init kmem_cache_init(void)
```


###   > `mm_init` > `percpu_init_late`
[/mm/percpu.c](http://lxr.free-electrons.com/source/mm/percpu.c#L2272)
```c
/*
 * First and reserved chunks are initialized with temporary allocation
 * map in initdata so that they can be used before slab is online.
 * This function is called after slab is brought up and replaces those
 * with properly allocated maps.
 */
void __init percpu_init_late(void)
```
###   > `mm_init` > `pgtable_init`


> 문c블로그  
> `page->ptl`용 slub 캐시  
> `DEBUG_SPINLOCK`과 `DEBUG_LOCK_ALLOC`이 활성화된 경우 x86_64 아키텍처에서 `spinlock_t`는 72 바이트가 사용되고 slab(slub) 캐시를 통해 할당되는 경우 근접한 사이즈인 `kmalloc-96`이 사용되고 24바이트의 사이즈가 낭비된다. 수 만개 이상의 페이지에 사용되는 `page->ptl`에 대해 너무 많은 메모리가 낭비된다 판단하여 `spinlock_t` 사이즈에 딱 맞는 캐시를 별도로 만들어 사용하게 되었다.
> [mm: create a separate slab for page->ptl allocation](https://github.com/torvalds/linux/commit/b35f1819acd9243a3ff7ad25b1fa8bd6bfe80fb2)


First and reserved chunks are initialized with temporary allocation map in initdata so that they can be used before slab is online. This function is called after slab is brought up and replaces those with properly allocated maps.

```c
static inline void pgtable_init(void)
{
    ptlock_cache_init();
    pgtable_cache_init();
}
```

###   > `mm_init` > `vmalloc_init`

> [문c 블로그]  
> 커널이 사용하는 연속된 가상 주소 메모리를 할당 받는 메커니즘을 위해 초기화를 수행한다.


[/mm/vmalloc.c](http://lxr.free-electrons.com/source/mm/vmalloc.c#L1222)

CPU 수만큼 루프를 돌면서 vmap_area 할당

###   > `mm_init` > `ioremap_huge_init`
`CONFIG_HAVE_ARCH_HUGE_VMAP` 활성화시에만 사용
주석 없음

[/lib/ioremap.c](http://lxr.free-electrons.com/source/lib/ioremap.c#L28)

```c
#ifdef CONFIG_HAVE_ARCH_HUGE_VMAP
    void __init ioremap_huge_init(void); // <<<
    int arch_ioremap_pud_supported(void);
    int arch_ioremap_pmd_supported(void);
#else
    static inline void ioremap_huge_init(void) { }
```

----------------
```
// 체크포인트!
```
----------------



##  > `sched_init`
[/kernel/sched/core.c](http://lxr.free-electrons.com/source/kernel/sched/core.c#L7338)  

Set up the scheduler prior starting any interrupts (such as the timer interrupt). Full topology setup happens at `smp_init()` time - but meanwhile we still have a functioning scheduler.

## > `preempt_disable`






