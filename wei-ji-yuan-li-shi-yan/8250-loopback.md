# 8250 自发自收实验



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

