# ARM裸机编程——start.s汇编分析

[TOC]



## 源代码内容
```asm
    .globl _start
    _start: 
    b	reset
    ldr	pc, _undefined_instruction
    ldr	pc, _software_interrupt
    ldr	pc, _prefetch_abort
    ldr	pc, _data_abort
    ldr	pc, _not_used
    ldr	pc, _irq
    ldr	pc, _fiq
    _undefined_instruction: .word _undefined_instruction
    _software_interrupt:	.word _software_interrupt
    _prefetch_abort:	.word _prefetch_abort
    _data_abort:		.word _data_abort
    _not_used:		.word _not_used
    _irq:			.word _irq
    _fiq:			.word _fiq
    
    reset:
    /* set the cpu to SVC32 mode*/
    mrs	r0, cpsr
    bic	r0, r0, #0x1f  
    orr	r0, r0, #0xd3  @11010011
    msr	cpsr,r0
    
    /* Set vector address in CP15 VBAR register */
    @CP15 协处理器 负责帮助CPU管理存储事宜
    ldr	r0, =0x41000000
    mcr	p15, 0, r0, c12, c0, 0	@Set VBAR
    
    /*stack init*/
    ldr sp, stacktop  @设置 svc的栈
    sub r1, sp, #128 
    
    /* set the cpu to user mode*/
    mrs	r0, cpsr
    bic	r0, r0, #0x1f  
    orr	r0, r0, #0xd0  @11010000
    msr	cpsr,r0
    
    mov sp, r1      @设置user的栈
    
    b	main
    
    
    stacktop:  .word stack+128*4
    stack: .space 128*4 
```




## 源代码部分解析

	通过汇编代码设置中断向量表的入口地址，想量表地址从0x00--0x1C 共 7*4 = 28Byte 
	想量表顺序通过查ARM芯片手册得到，地址偏移表如下:
|地址|中断名称|
|---|---|
|0x1C|FIQ_IRQ_Handler|
|0x18|IRQ_Handler|
|0x14|NotUsed_Handler|
|0x10|DataAbort_Handler|
|0x0C|Prefetch_Handler|
|0x08|SWI_Handler|
|0x04|Undefine_Handler|
|0x00|Reset_Handler|

```asm
    b reset
    ldr	pc, _undefined_instruction
    ldr	pc, _software_interrupt
    ldr	pc, _prefetch_abort
    ldr	pc, _data_abort
    ldr	pc, _not_used
    ldr	pc, _irq
    ldr	pc, _fiq
```
	每一个异常想量表都对应一个入口地址，对应着用户解决异常的方法的入口地址，数据宽度为1个字 32Bit，其中省略了Reset_Handler的异常处理地址，选择了直接跳转reset方法。
```asm
    _undefined_instruction: .word _undefined_instruction
    _software_interrupt:	.word _software_interrupt
    _prefetch_abort:	.word _prefetch_abort
    _data_abort:		.word _data_abort
    _not_used:		.word _not_used
    _irq:			.word _irq
    _fiq:			.word _fiq
```
	reset处理函数如下，主要是负责对各个模式的栈进行初始化，在内存上对各个模式开辟栈地址空间，开辟代码如下:
```asm
    stack_top:.word stack+4*128 @获得栈地址空间的顶部
    stack:.space 128*4	@开辟4*128字节
```
	reset函数程序代码如下:
```asm

    reset:
    /* set the cpu to SVC32 mode*/
    mrs	r0, cpsr
    bic	r0, r0, #0x1f  
    orr	r0, r0, #0xd3  @11010011  FIQ和IRQ被关闭
    msr	cpsr,r0

    /* Set vector address in CP15 VBAR register */
    ldr	r0, =0x41000000 
    mcr	p15, 0, r0, c12, c0, 0	@Set VBAR

    /*stack init*/
    ldr sp, stacktop  @设置 svc的栈
    sub r1, sp, #128 

    /* set the cpu to user mode*/
    mrs	r0, cpsr
    bic	r0, r0, #0x1f  
    orr	r0, r0, #0xd0  @11010000	FIQ和IRQ被关闭
    msr	cpsr,r0

    mov sp, r1      @设置user的栈

    b	main	@跳转主函数
```

	CPSR 程序状态寄存器
|M[4:0]|处理器·模式|Hex|
|-|-|-|
|0b10000|user|0x10|
|0b10001|FIQ|0x11|
|0b10010|IRQ|0x12|
|0b10011|SVC|0x13|
|0b10111|Abort|0x17|
|0b11011|Undefined|0x1B|
|0b11111|System|0x1F|
|0b10110|Secure monitor|0x16|








