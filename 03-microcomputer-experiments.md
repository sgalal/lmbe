# 03-微机原理实验

#### 段式存储管理

#### 进入 32 位保护模式

### 芯片

#### 8254 数字跳跃实验

实验内容：

文本模式下在屏幕中央显示一个数字，数值从 0-9 跳跃，每秒 1 次。

![8254 &#x6570;&#x5B57;&#x8DF3;&#x8DC3;&#x5B9E;&#x9A8C;](./files_lmsitbe/8254_number_skip.gif)

思路：

8254 的 `OUT0` 与 8259 的 `IRQ0` 连接，因此，控制 8254 的 `CNT0` 产生每秒 1 次的脉冲信号，就相当于每秒产生 1 次 `IRQ0` 中断，BIOS 初始化时，`IRQ0` 对应 `INT8`，在中断向量表的地址是 8&lt;&lt;2=0x20，自定义该中断向量的中断服务程序即可。

实际上由于数值大小的限制，8254 的 `CNT0` 并不能产生每秒 1 次的脉冲信号，但是可以产生每秒 20 次的脉冲信号，然后在程序中设置一个计数器进行分频即可。

```text
    Screen_center equ ((12*80+39)<<1)

[org    0x7c00]

start:
    ; 设置初始值
    mov    ax,0xb800                     ; 文字模式的显存地址
    mov    es,ax
    mov    byte [es:Screen_center],'0'

    ; 设置中断向量
    cli                                    ; 关中断
    xor    ax,ax
    mov    es,ax
    mov    bx,0x20                        ; 中断向量号 0x20
    mov    word [es:bx],new_int_0x20      ; 自定义的中断处理程序的偏移地址
    mov    [es:bx+2],cs                   ; 段地址
    sti                                    ; 开中断

    mov    al,0b_00_11_011_0       ; 计数器 0，先低后高，方式 3，二进制计数
    out    0x43,al                 ; 写入控制寄存器
    mov    ax,59659                ; 设定计数初值
    out    0x40,al                 ; 写入 CNT0 端口
    mov    al,ah
    out    0x40,al

fin:
    hlt
    jmp    fin

; 自定义的中断处理程序
new_int_0x20:
        push    ax
        push    es

        cmp    byte [count],19
        jne    .new_int_0x20.count_noteq_19   ; 计数到 19 则改为 0，然后变化屏幕上的数字
        mov    byte [count],0

        mov    ax,0xb800                      ; 文字模式的显存地址
        mov    es,ax
        cmp    byte [es:Screen_center],'9'
        je    .new_int_0x20.al_equ_9
        inc    byte [es:Screen_center]         ; 屏幕数字不为 '9' 则加一
        jmp    .new_int_0x20.fin

    .new_int_0x20.al_equ_9:
        mov    byte [es:Screen_center],'0'     ; 为 '9' 则改为 '0'
        jmp    .new_int_0x20.fin

    .new_int_0x20.count_noteq_19:             ; 计数未到 19 则加一
        inc    byte [count]

    .new_int_0x20.fin:
        ; EOI
        mov    al,0x20
        out    0xa0,al
        out    0x20,al

        pop    es
        pop    ax
        iret

; 计数器，用于分频，范围 0-19，19 -> 0 时变化数字
count:
    db    0

signature:
    %if    $-$$>510
        %fatal    "stage1 code exceed 512 bytes."
    %endif

    times    510-($-$$) \
        db    0
    db    0x55,0xaa
```

> Overall, life would have been better if IBM had originally programmed the timer to interrupt after 59659 hardware ticks. This would have produced an interrupt 20 times per second, which would have been more convenient for everyone, with very little change in performance. \(Isn’t 20/20 hindsight wonderful?\)
>
> _Jim Lyon_'s rewiew on [https://blogs.msdn.microsoft.com/oldnewthing/20041202-00/?p=37153](https://blogs.msdn.microsoft.com/oldnewthing/20041202-00/?p=37153), December 2, 2004 at 9:02 am.

参考资料：

* [https://wiki.osdev.org/Interrupt](https://wiki.osdev.org/Interrupt)

#### 8254 基本音级实验

实验内容：

控制扬声器发出 do re mi fa so la si do. 的声音，每个音持续 0.25 秒，两个音之间的间隔也为 0.25 秒。

思路：

首先使用与上一个实验类似的方法产生每 0.25 秒一次的中断。在中断服务程序中调用处理函数。

设置一个 `state` 变量，初值为 0，每调用一次处理函数，变量的值加 1。

处理函数的具体功能是：`state` 变量的值若为偶数，则使用相应的频率调用播放声音函数；若为奇数，则调用停止声音函数。

8254 芯片的 `GATE2` 以 8255 的 `PB0` 作为输入，8254 的 `OUT2` 与 8255 的 `PB1` 通过一个与门连接到扬声器，因此播放声音函数的实现方法是：首先初始化 8254 的 `CNT2`，设定发声频率，然后打开 8255 的 `PB0` 和 `PB1`。相应地，停止声音函数的实现方法是关闭 8255 的 `PB0` 和 `PB1`。

```text
[org    0x7c00]

start:
    ; 设置中断向量
    cli                                    ; 关中断
    xor    ax,ax
    mov    es,ax
    mov    bx,0x20                        ; IRQ0，中断向量表的偏移量 0x20
    mov    word [es:bx],new_int_0x20      ; 自定义的中断处理程序的偏移地址
    mov    [es:bx+2],cs                   ; 段地址
    sti                                    ; 开中断

    mov    al,0b_00_11_011_0       ; 计数器 0，先低后高，方式 3，二进制计数
    out    0x43,al                 ; 写入控制寄存器
    mov    ax,59659                ; 设定计数初值
    out    0x40,al                 ; 写入 CNT0 端口
    mov    al,ah
    out    0x40,al

fin:
    hlt
    jmp    fin

; 自定义的中断处理程序
new_int_0x20:
        push    ax
        push    es

        cmp    byte [count],4
        jne    .new_int_0x20.count_noteq_9    ; 计数到 4 则改为 0，然后变化屏幕上的数字，0.25 秒
        mov    byte [count],0

        cmp    byte [state],16                ; state 计到 16，说明播放声音的任务圆满完成，直接结束
        je    .new_int_0x20.fin

        call    state_handling                 ; 真实处理函数
        inc    byte [state]

    .new_int_0x20.count_noteq_9:                   ; 计数未到 4 则加一
        inc    byte [count]

    .new_int_0x20.fin:
        ; EOI
        mov    al,0x20
        out    0xa0,al
        out    0x20,al

        pop    es
        pop    ax
        iret

; 根据 state 的值作出相应反应
state_handling:
        push    ax

        mov    al,[state]
        or    al,al
        jz    .state_handling.state0
        dec    al
        jz    .state_handling.state1
        dec    al
        jz    .state_handling.state2
        dec    al
        jz    .state_handling.state3
        dec    al
        jz    .state_handling.state4
        dec    al
        jz    .state_handling.state5
        dec    al
        jz    .state_handling.state6
        dec    al
        jz    .state_handling.state7
        dec    al
        jz    .state_handling.state8
        dec    al
        jz    .state_handling.state9
        dec    al
        jz    .state_handling.state10
        dec    al
        jz    .state_handling.state11
        dec    al
        jz    .state_handling.state12
        dec    al
        jz    .state_handling.state13
        dec    al
        jz    .state_handling.state14

    .state_handling.state1:         ; 停音
    .state_handling.state3:
    .state_handling.state5:
    .state_handling.state7:
    .state_handling.state9:
    .state_handling.state11:
    .state_handling.state13:
        call    stop_sound

    .state_handling.end:
        pop    ax
        ret

    .state_handling.state0:         ; state0，响 do 音，256 Hz
        mov    ax,4648
        call    make_sound
        jmp    .state_handling.end

    .state_handling.state2:         ; state2，响 re 音，288 Hz
        mov    ax,4131
        call    make_sound
        jmp    .state_handling.end

    .state_handling.state4:         ; state4，响 mi 音，320 Hz
        mov    ax,3719
        call    make_sound
        jmp    .state_handling.end

    .state_handling.state6:         ; state6，响 fa 音，341又1/3 Hz
        mov    ax,3486
        call    make_sound
        jmp    .state_handling.end

    .state_handling.state8:         ; state8，响 so 音，384 Hz
        mov    ax,3099
        call    make_sound
        jmp    .state_handling.end

    .state_handling.state10:         ; state10，响 la 音，426又2/3 Hz
        mov    ax,2789
        call    make_sound
        jmp    .state_handling.end

    .state_handling.state12:         ; state12，响 si 音，480 Hz
        mov    ax,2479
        call    make_sound
        jmp    .state_handling.end

    .state_handling.state14:         ; state14，响 do. 音，512 Hz
        mov    ax,2324
        call    make_sound
        jmp    .state_handling.end

; 控制扬声器发出声音
; ax 频率
make_sound:
    push    ax
    mov    al,0b_10_11_011_0      ; CNT2 控制字，先写低字节后写高字节，方式 3，二进制计数
    out    0x43,al

    pop    ax
    out    0x42,al
    mov    al,ah
    out    0x42,al

    in    al,0x61    ; 取 8255 PB 口
    or    al,0x03    ; 打开 PB0 和 PB1
    out    0x61,al    ; 送回 8255 PB 口
    ret

; 控制扬声器停止声音
stop_sound:
    push    ax
    in    al,0x61
    and    al,0xfc
    out    0x61,al
    pop    ax
    ret

; 计数器，用于分频，范围 0-19，19 -> 0 时变化数字
count:
    db    0

; 状态
state:
    db    0

signature:
    %if    $-$$>510
        %fatal    "stage1 code exceed 512 bytes."
    %endif

    times    510-($-$$) \
        db    0
    db    0x55,0xaa
```

#### 8255

#### 8250 自发自收实验

实验内容：

用 8250 实现串行口异步通信，自发自收，波特率1200bps。采用查询方式发送，采用中断方式接收。发送程序连续将符号 '\*' 从串口 `COM1` 发送出去，`COM1` 接收到的字符显示在屏幕上，当线路出错时屏幕输出 '\#'。

**（本程序还不完善！）**

```text
[org    0x7c00]

start:
    cli
    mov    bx,0x30                   ; INT 0CH，中断向量表的偏移量 0x30
    mov    word [cs:bx],int_recv     ; 自定义的中断处理程序的偏移地址
    mov    [cs:bx+2],cs              ; 段地址

    ; 开放 COM1 的中断请求
    in    al,0x21             ; 读 8259 主片 OCW1
    and    al,0xef             ; COM1 对应 IRQ4
    out    0x21,al

    ; 设置波特率 9600 b/s
    mov    al,0x80             ; 设 DLAB 为 1，访问除数锁存器
    mov    dx,0x03fb           ; COM1 线路控制寄存器（LCR）
    out    dx,al

    mov    al,0x0c
    mov    dx,0x03f8           ; COM1 的除数锁存器低字节（DLL）
    out    dx,al

    mov    al,0x00
    mov    dx,0x03f9           ; COM1 的除数锁存器低字节（DLH）
    out    dx,al

    mov    al,0b_00_001_0_11   ; 设 DLAB 为 0，为正常模式；8 位数据位，1 位停止位，1 位奇校验位
    mov    dx,0x03fb           ; COM1 线路控制寄存器（LCR）
    out    dx,al

    mov    al,0b_000_0_1011    ; 令 DTR、RTS 和 OUT2 有效
    mov    dx,0x03fc           ; COM1 Modem 控制器（MCR）
    out    dx,al

    mov    al,0x01             ; 只允许接收数据中断
    mov    dx,0x03f9           ; COM1 中断允许寄存器（IER）
    out    dx,al
    sti

infi:
    mov    dx,0x03fd           ; COM1 线路状态寄存器（LSR）
    in    al,dx
    test    al,0x20             ; 测试发送保持寄存器是否为空
    jz    infi                ; THR 不空则等待

    mov    al,'X'              ; 发送字符 'X'
    mov    dx,0x03f8           ; COM1 的寄存器地址
    out    dx,al
    jmp    infi

int_recv:
        pusha
        call    beep

    .int_recv.try:
        mov    dx,0x03fd         ; COM1 线路状态寄存器（LSR）
        in    al,dx             ; 取 COM1 线路状态
        test    al,0x1e
        jnz    .int_recv.err

        mov    dx,0x03f8         ; COM1 的寄存器地址
        in    al,dx
        call    printc

    .int_recv.fin:
        ; EOI
        mov    al,0x20           ; 8259 OCW2，一般 EOI
        out    0x20,al

        popa
        iret

    .int_recv.err:
        mov    dx,0x03f8         ; 清除读接收缓冲寄存器（RBR），以免溢出错
        in    al,dx
        mov    al,'#'
        call    printc
        jmp    .int_recv.fin

; Print a character
; al character
printc:
    push    ax
    mov    ah,0x0e
    int    0x10
    pop    ax
    ret

; Make a beep sound
beep:
    pusha

    mov    al,0xb6
    out    0x43,al          ; 8254 CNT2
    mov    ax,0x0533        ; Sound frequency 896 Hz
    out    0x42,al
    mov    al,ah
    out    0x42,al

    in    al,0x61          ; 8255 PB
    mov    ah,al
    or    al,0x03          ; Enable 8254 CNT2
    out    0x61,al

    mov    cx,0x0055        ; Last for some time
    loop2:
        push    cx
        mov    cx,0xffff
        loop1:
            nop
            loop    loop1
        pop    cx
        loop    loop2

    mov    al,ah
    out    0x61,al          ; Restore 8255 PB
    popa
    ret

signature:
    %if    $-$$>510
        %fatal    "stage1 code exceed 512 bytes."
    %endif

    times    510-($-$$) \
        db    0
    db    0x55,0xaa
```

参考资料：

* 教材 P192 表 4.10

#### 8259 鼠标移动实验

实验内容：

设定画面模式为为图形模式、320x200x8 位彩色，鼠标中断的中断类型码为 INT 2CH。以一个小矩形作为画面上的鼠标，当移动鼠标时，画面上的鼠标也跟着移动。

![8259 &#x9F20;&#x6807;&#x79FB;&#x52A8;&#x5B9E;&#x9A8C;](./files_lmsitbe/8259_mouse_move.jpg)

思路：

PS/2 鼠标连接在 8259 的从片 `IRQ4` 引脚上，即 `IRQ12`。通过对 8259 初始化，将鼠标中断的中断类型码为设定为 INT 2CH，然后编写相应的中断处理程序。在中断处理程序中，获取鼠标的信息，根据内存中的记录鼠标左上角的横纵坐标完成鼠标的移动。

**（说明：此程序可以正常运行，但在鼠标的端口方面还有一些疑问！）**

```text
    Square_len equ 4
    Background_color equ 0x0
    Foreground_color equ 0x3

[org    0x7c00]

start:
    mov    ax,0x0013          ; 设置显示器模式为图形模式、320x200x8 位彩色
    int    0x10

    mov    ax,0xa000          ; 设置 es 段基地址为显存区域 0xa0000 处
    mov    es,ax

    ; 填充背景色
    mov    di,(320*200)
fill_background_loop:
    mov    byte [es:di],Background_color
    dec    di
    jnz    fill_background_loop

    ; 绘制鼠标
    mov    al,Foreground_color
    call    fill_mouse

    ; 设置中断向量
    cli                                   ; 关中断
    mov    bx,0xb0                       ; INT2C，中断向量表的偏移量 0xb0
    mov    word [cs:bx],new_int_0xb0     ; 自定义的中断处理程序的偏移地址
    mov    [cs:bx+2],cs                  ; 段地址
    mov    bx,0x84                       ; INT21，中断向量表的偏移量 0x84
    mov    word [cs:bx],new_int_0x84     ; 自定义的中断处理程序的偏移地址
    mov    [cs:bx+2],cs                  ; 段地址
    sti                                   ; 开中断

    ; 初始化 8259
    mov    al,0xff     ; 禁止主片和从片所有中断
    out    0x21,al     ; 主片 OCW1
    out    0xa1,al     ; 从片 OCW1

    ; 主片初始化
    mov    al,0b_000_1_0_0_0_1   ; 边沿触发，级联使用
    out    0x20,al               ; 主片，初始化命令字 ICW1
    mov    al,0x20               ; 设置 IRQ0-7 产生中断类型码 INT20-27
    out    0x21,al               ; 主片，初始化命令字 ICW2
    mov    al,(1<<2)             ; PIC1 由 IRQ2 连接
    out    0x21,al               ; 主片，初始化命令字 ICW3
    mov    al,0b_000_0_00_0_1    ; 一般全嵌套方式，非缓冲方式，不自动结束中断，8259 工作在 8088/8086 系统中
    out    0x21,al               ; 主片，初始化命令字 ICW4

    ; 从片初始化
    mov    al,0b_000_1_0_0_0_1   ; 边沿触发，级联使用
    out    0xa0,al               ; 从片，初始化命令字 ICW1
    mov    al,0x28               ; 设置 IRQ8-15 产生中断类型码 INT28-2f
    out    0xa1,al               ; 从片，初始化命令字 ICW2
    mov    al,0x02               ; PIC1 连接主片 IRQ2
    out    0xa1,al               ; 从片，初始化命令字 ICW3
    mov    al,0b_000_0_00_0_1    ; 一般全嵌套方式，非缓冲方式，不自动结束中断，8259 工作在 8088/8086 系统中
    out    0xa1,al               ; 从片，初始化命令字 ICW4

    mov    al,0b_11111001        ; 禁止主片所有中断，只留键盘（因为似乎也要处理键盘中断？）
    out    0x21,al               ; 主片 OCW1
    mov    al,0b_11101111        ; 禁止除 IRQ12 外所有中断
    out    0xa1,al               ; 从片 OCW1

    call    kb_wait
    mov    al,0x60         ; 键盘写模式
    out    0x64,al         ; 8042 端口

    call    kb_wait
    mov    al,0x47         ; 键盘写模式
    out    0x60,al         ; 键盘数据端口

    call    kb_wait
    mov    al,0xd4         ; 操作鼠标
    out    0x64,al

    call    kb_wait
    mov    al,0xf4         ; 激活鼠标
    out    0x60,al

fin:
    hlt
    jmp    fin

; 自定义的中断处理程序，鼠标
new_int_0xb0:
        pusha

        mov    al,[mouse_read_state]
        or    al,al
        jz    .new_int_0xb0.read_state0
        dec    al
        jz    .new_int_0xb0.read_state1
        dec    al
        jz    .new_int_0xb0.read_state2

    .new_int_0xb0.read_state3:
        in    al,0x64                        ; 读 8042 的状态寄存器
        test    al,0x01                        ; 检测第一位
        jz    .new_int_0xb0.fin              ; 若无数据可读，结束
        in    al,0x60                        ; 读出鼠标的纵坐标偏移量
        neg    al                             ; 需要取反
        mov    [square_pos_offset_y],al       ; 将更改后的偏移量存入内存
        mov    byte [mouse_read_state],1  ; 本来应该跳到 0（接收 ACK 信号）的，结果不对，改成跳到 1 就好了

        ; 测试鼠标是否超出边界
        mov    bx,[square_pos_x]
        mov    al,[square_pos_offset_x]
        cbw
        add    bx,ax
        js    .new_int_0xb0.fin
        cmp    bx,(320-Square_len)
        jge    .new_int_0xb0.fin

        mov    cx,[square_pos_y]
        mov    al,[square_pos_offset_y]
        cbw
        add    cx,ax
        js    .new_int_0xb0.fin
        cmp    cx,(200-Square_len)
        jge    .new_int_0xb0.fin

        ; 清除原鼠标
        mov    al,Background_color
        call    fill_mouse

        ; 重绘鼠标
        mov    [square_pos_x],bx
        mov    [square_pos_y],cx
        mov    al,Foreground_color
        call    fill_mouse

    .new_int_0xb0.fin:
        ; EOI
        mov    al,0b_011_00_100       ; IRQ12 对应 IR4
        out    0xa0,al                ; 从片 OCW2
        mov    al,0b_011_00_010       ; IRQ2 对应 IR2
        out    0x20,al                ; 主片 OCW2

        popa
        iret

    .new_int_0xb0.read_state0:
        in    al,0x60                     ; 等待鼠标 ACK 信号
        cmp    al,0xfa
        jne    .new_int_0xb0.fin           ; 等待直到收到 ACK 信号
        mov    byte [mouse_read_state],1
        jmp    .new_int_0xb0.fin

    .new_int_0xb0.read_state1:
        in    al,0x64                        ; 读 8042 的状态寄存器
        test    al,0x01                        ; 检测第一位
        jz    .new_int_0xb0.fin              ; 若无数据可读，结束
        in    al,0x60                        ; 有数据则读出
        mov    byte [mouse_read_state],2
        jmp    .new_int_0xb0.fin

    .new_int_0xb0.read_state2:
        in    al,0x64                        ; 读 8042 的状态寄存器
        test    al,0x01                        ; 检测第一位
        jz    .new_int_0xb0.fin              ; 若无数据可读，结束
        in    al,0x60                        ; 读出鼠标的横坐标偏移量
        mov    [square_pos_offset_x],al       ; 将更改后的偏移量存入内存
        mov    byte [mouse_read_state],3
        jmp    .new_int_0xb0.fin

; 自定义的中断处理程序，键盘
new_int_0x84:
        call    beep

        ; EOI
        mov    al,0b_011_00_001  ; IRQ1 对应 IR1
        out    0x20,al           ; 主片 OCW2

    .new_int_0x84.kb_wait:
        in    al,0x64            ; 读 8042 的状态寄存器
        test    al,0x01            ; 检测第一位
        jz    .new_int_0x84.fin  ; 若无数据可读，结束
        in    al,0x60            ; 有数据则读出
        jmp    .new_int_0x84.kb_wait

    .new_int_0x84.fin:
        iret

; Draw mouse
; al draw color
fill_mouse:
        pusha
        push    ax                      ; 暂存 al

        mov    ax,[square_pos_y]       ; 鼠标左上角纵坐标乘以 320 放入 di
        mov    di,ax
        shl    di,8                    ; 先 256 倍
        shl    ax,6                    ; 再加 64 倍
        add    di,ax

        mov    ax,[square_pos_x]       ; 加上鼠标左上角横坐标
        add    di,ax

            pop    ax                  ; 恢复 al
            mov    cx,Square_len       ; 绘制 Square_len 列

    .fill_mouse.draw_y:
            push    cx               ; 暂存外层循环的计数值
            push    di               ; 暂存行绘制开始时的小方块的地址
            mov    cx,Square_len    ; 绘制 Square_len 行

        .fill_mouse.draw_x:
            mov    [es:di],al
            inc    di
            loop    .fill_mouse.draw_x

            pop    di                 ; 恢复行绘制开始时的小方块的地址指针
            add    di,320             ; 跳到下一行
            pop    cx                 ; 恢复外层循环的计数值
            loop    .fill_mouse.draw_y

        popa
        ret

; 等待 8042 键盘电路响应
kb_wait:
    in    al,0x64          ; 读键盘电路
    test    al,0x02          ; 检测 8042 状态寄存器的第二位
    jnz    kb_wait
    ret

; Make a beep sound
beep:
    pusha

    mov    al,0xb6
    out    0x43,al          ; 8254 CNT2
    mov    ax,0x0533        ; Sound frequency 896 Hz
    out    0x42,al
    mov    al,ah
    out    0x42,al

    in    al,0x61          ; 8255 PB
    mov    ah,al
    or    al,0x03          ; Enable 8254 CNT2
    out    0x61,al

    mov    cx,0x0001        ; Last for some time
    loop2:
        push    cx
        mov    cx,0xffff
        loop1:
            nop
            loop    loop1
        pop    cx
        loop    loop2

    mov    al,ah
    out    0x61,al          ; Restore 8255 PB
    popa
    ret

; 鼠标位置信息，初始值在屏幕中央
square_pos_x:
    dw    ((320-Square_len)>>1)

square_pos_y:
    dw    ((200-Square_len)>>1)

square_pos_offset_x:
    db    0

square_pos_offset_y:
    db    0

mouse_read_state:
    db    0

signature:
    %if    $-$$>510
        %fatal    "stage1 code exceed 512 bytes."
    %endif

    times    510-($-$$) \
        db    0
    db    0x55,0xaa
```

参考资料：

* [https://wiki.osdev.org/Interrupt](https://wiki.osdev.org/Interrupt)
* [https://wiki.osdev.org/%228042%22\_PS/2\_Controller](https://wiki.osdev.org/%228042%22_PS/2_Controller)
* [https://wiki.osdev.org/Mouse\_Input](https://wiki.osdev.org/Mouse_Input)

#### 8237

### 附录

#### 开发环境配置

**必要程序：QEMU（模拟器）、NASM（汇编器）、GNU Make（生成工具）、Bochs（调试器）**

**其他程序：Notepad++（编辑器）、objdump（查看反汇编）**

二进制文件可以使用 xxd 命令查看。

Boot

```text
[org    0x7c00]

start:

    ; Add scripts here

fin:
    hlt
    jmp    fin

signature:
    %if    $-$$>510
        %fatal    "stage1 code exceed 512 bytes."
    %endif

    times    510-($-$$) \
        db    0
    db    0x55,0xaa
```

Makefile

```text
main.bin : main.asm Makefile
    nasm -fbin main.asm -o main.bin -l main.lst

run : main.bin
    qemu-system-x86_64 -soundhw all -rtc base=localtime -drive file=main.bin,format=raw,index=0,media=disk

dmp : main.bin
    objdump -M intel -b binary -m i8086 -D main.bin
```

#### 基本函数实现

**扬声器发声**

beep

```text
; Make a beep sound
beep:
    pusha

    mov    al,0xb6
    out    0x43,al          ; 8254 CNT2
    mov    ax,0x0533        ; Sound frequency 896 Hz
    out    0x42,al
    mov    al,ah
    out    0x42,al

    in    al,0x61          ; 8255 PB
    mov    ah,al
    or    al,0x03          ; Enable 8254 CNT2
    out    0x61,al

    mov    cx,0x0055        ; Last for some time
    loop2:
        push    cx
        mov    cx,0xffff
        loop1:
            nop
            loop    loop1
        pop    cx
        loop    loop2

    mov    al,ah
    out    0x61,al          ; Restore 8255 PB
    popa
    ret
```

#### Bochs 调试方法

