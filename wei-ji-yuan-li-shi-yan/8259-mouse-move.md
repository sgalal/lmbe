# 8259 鼠标移动实验

实验内容：

设定画面模式为为图形模式、320x200x8 位彩色，鼠标中断的中断类型码为 INT 2CH。以一个小矩形作为画面上的鼠标，当移动鼠标时，画面上的鼠标也跟着移动。

![](../.gitbook/assets/8259_mouse_move.jpg)

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

