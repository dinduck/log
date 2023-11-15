我们已经让内核使用上第一条代码 `li x1, 100`

但是注意我们寄存器信息
`sp` 和 `fp` 的值均为 `0`
我们应该分配一个栈空间

编辑 `entry.asm`
```Assembly
  .section  .text.entry
  .globl  _start
_start:
  la sp, boot_stack_top
  call rust_main

  .section .bss.stack
  .globl boot_stack_lower_bound
boot_stack_lower_bound:
  .space 4096 * 16
  .globl boot_stack_top
boot_stack_top:
```
汇编仅仅是助记符， 相比机器语言， 汇编更为让人看懂
这里解释这段程序, 其实对于 `_start` `boot_stack_top` 有多种翻译， 助记符， 符号， 标签。`.section` 和 `.seguement`  就是设计上的， 因为机器只能读取二进制代码， 他们眼里没有 `rust_main` 这种大小， 这种符号的作用表示一层抽象， 在转换机器语言时 ， `rust_main` 就代表 `la sp, boot_stack_top` 这条指令位置,  程序里面会有很多助记符标记代码信息， `_start` 表示入口， 其实这里存在 `_start` -> `0x80200000` 的映射， 调试器知道 `_start` 在 `0x80200000`, 如果没有这种原信息, 调试器怎么知道 `0x80200000` 表示什么。 `ELF` 文件里面存在众多 `Meta Data` 这方便调试。
![image.png](https://s2.loli.net/2023/10/22/2V9FvLaobrceA6q.png)
我们可以得出他们间的映射关系，  显然这种关系是双向的， 我们需要这种抽象

最后， 这段程序创建的一个 64k  大小的栈空间， `la` `load addr` 将栈顶地址加载至  `sp` `stack pointer reg` 栈顶指针寄存器， `boot_stack_top` 这个符号就是栈顶地址的标记, 然后 `call rust_main`  调用 `rust_main` 函数， 其实这里， `call` 会将 `rust_main` 符号标注的地址加载至 `pc`， 实际 `call` 是一条伪指令，实际是这两指令 `auipc rd symbol[31:12]` `addi rd, rd, symbol[11:0]`, 因为指令是 32 位的， 单单一个  `op`  就要 7 位， 所以这里将指令拆两半分别写入

然后就是下面这两条指令，`jal` 和 `jalr` 都是伪指令

![image.png](https://s2.loli.net/2023/10/22/QsWBC6ziJr7HlaN.png)
