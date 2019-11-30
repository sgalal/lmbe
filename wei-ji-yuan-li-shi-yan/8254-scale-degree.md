# 8254 基本音级实验

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

