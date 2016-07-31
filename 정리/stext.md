# `stext`

> Google Docs를 기반으로 정리 진행 중입니다 :)

```
ENTRY(stext)
       bl      preserve_boot_args
       bl      el2_setup                       // Drop to EL1, w20=cpu_boot_mode
       bl      set_cpu_boot_mode_flag
      (......)
       bl      __create_page_tables            // x25=TTBR0, x26=TTBR1
       /*
        * The following calls CPU setup code, see arch/arm64/mm/proc.S for
        * details.
        * On return, the CPU will be ready for the MMU to be turned on and
        * the TCR will have been set.
        */
       ldr     x27, 0f                         // address to jump to after
                                               // MMU has been enabled
       adr_l   lr, __enable_mmu                // return (PIC) address
       b       __cpu_setup                     // initialise processor
ENDPROC(stext)
```

### `stext` > `preserve_boot_args`
#### 메모
Preserve the arguments passed by the bootloader in x0 .. x3  
드라이버들에게 전달해야 할 인자를 미리 정적으로 할당된 버퍼에 저장?


### `stext` > `el2_setup`
If we're fortunate enough to boot at EL2, ensure that the world is sane before dropping to EL1.
Returns either `BOOT_CPU_MODE_EL1` or `BOOT_CPU_MODE_EL2` in x20 if booted in EL1 or EL2 respectively.

#### 메모
EL2를 사용할 지 불명확함. EL2가 아니면 무시하는 내용?



### `stext` > `set_cpu_boot_mode_flag`
Sets the `__boot_cpu_mode` flag depending on the CPU boot mode passed in x20. See `arch/arm64/include/asm/virt.h` for more info.

#### 메모
`el2_setup` 에서 변경된 x20 값을 사용해 EL 값을 __boot_cpu_mode에 써놓는다?

#### 참조
[/arch/arm64/include/asm/virt.h](http://lxr.free-electrons.com/source/arch/arm64/include/asm/virt.h)  

### `stext` > `__create_page_tables`
Setup the initial page tables. We only setup the barest amount which is required to get the kernel running. The following sections are required:  
 - identity mapping to enable the MMU (low address, `TTBR0`)  
 - first few MB of the kernel linear mapping to jump to once the MMU has been enabled  

#### 노트
페이지 테이블 초기화.   
`TTBR0` - 커널이 사용할 `TTBR1` 이 아닌 유저를 위한 `TTBR0` 만 초기화 하는 이유가 무엇인지?


#### 참조

### `stext` > `__enable_mmu`
Enable the MMU.

`x0`  = SCTLR_EL1 value for turning on the MMU.  
`x27` = **virtual** address to jump to upon completion  

Other registers depend on the function called upon completion. Checks if the selected granule size is supported by the CPU. If it isn't, park the CPU
#### 노트
`x27`은 `stext`에서 ```ldr x27, 0f``` 명령으로 MMU 활성화 이후 점프할 주소를 기록한 상태
#### 참조
`head.S` 231라인 
```
    .section        ".idmap.text", "ax"
```


### `__cpu_setup`
`stext`에서 넘어옴.  
Initialise the processor for turning the MMU on.  Return in `x0` the value of the `SCTLR_EL1` register.


### 
The following fragment of code is executed with the MMU enabled.
```
    .set    initial_sp, init_thread_union + THREAD_START_SP
__mmap_switched:
bl      __pi_memset
```


### // 
KASAN : Kernel Address Sanitizer
#### 노트
`kasan_early_init`는 x86_64를 위한 코드. Slab 디버깅에 필요한 내용
#### 참조
 - https://www.kernel.org/doc/Documentation/kasan.txt
 - https://lwn.net/Articles/652057/
 
```
#ifdef CONFIG_KASAN
        bl      kasan_early_init
#endif

#ifdef CONFIG_RANDOMIZE_BASE
        cbnz    x23, 0f                         // already running randomized?
        mov     x0, x21                         // pass FDT address in x0
        bl      kaslr_early_init                // parse FDT for KASLR options
        cbz     x0, 0f                          // KASLR disabled? just proceed
        mov     x23, x0                         // record KASLR offset
        ret     x28                             // we must enable KASLR, return
                                                // to __enable_mmu()
0:
#endif

b       start_kernel
```

-----------------------
