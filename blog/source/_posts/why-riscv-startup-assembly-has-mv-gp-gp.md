---
title: 為何RISC-V的開機assembly出現mv gp,gp?
date: 2025-10-19 20:31:32
tags: [RISC-V, Assembly, Embedded]
categories: 嵌入式系統
---

## 前言

在撰寫 RISC-V 的 bare-metal 啟動程式碼時，我們通常需要初始化全域指標暫存器（`gp`）。一開始，我使用了看似正常的寫法：

```c
__asm__ volatile("la gp, __global_pointer$ \n");
```

但反組譯後卻發現產生了一條奇怪的指令：`mv gp, gp`。這條指令將 `gp` 搬移到自己，看起來什麼都沒做。這是怎麼回事？

<!-- more -->

## 問題重現

### 錯誤的寫法

```c
void boot_init(void) {
    __asm__ volatile("la gp, __global_pointer$ \n");
}
```

### 反組譯結果

```asm
00000000 <boot_init>:
   0:   00000197          auipc   gp,0x0
   4:   00018193          mv      gp,gp      # ← 這是什麼？
```

等等，`mv gp, gp` 是什麼操作？這條指令毫無意義啊！

## 什麼是 gp 暫存器？

在深入問題之前，先理解 `gp` 的作用：

### Global Pointer 的用途
- `gp` (x3) 是 RISC-V 的**全域指標暫存器**
- 指向程式的全域資料區域（通常在 `.sdata` 附近）
- 編譯器用它來快速存取全域變數和靜態變數

### 為什麼需要 gp？

```c
// 沒有 gp：需要多條指令
int global_var = 100;

int get_value() {
    // 需要 lui + addi 載入完整位址
    return global_var;
}
```

```asm
# 沒有 gp-relative 定址
lui  a0, %hi(global_var)
lw   a0, %lo(global_var)(a0)   # 兩條指令

# 使用 gp-relative 定址
lw   a0, offset(gp)             # 只要一條！
```

## Linker Relaxation 的陷阱

問題的根源在於 **linker relaxation（連結器最佳化）**。

### 什麼是 Linker Relaxation？

RISC-V 連結器會自動優化某些指令序列：

```asm
# 原始指令
la gp, __global_pointer$
# 展開為：
auipc gp, %pcrel_hi(__global_pointer$)
addi  gp, gp, %pcrel_lo(__global_pointer$)

# 如果 __global_pointer$ 距離 PC 很近
# 連結器會「優化」成：
addi gp, gp, offset    # 只用一條指令
```

### 問題來了

當 `__global_pointer$` 恰好在當前位置附近時：

```asm
auipc gp, 0          # offset 的高位是 0
addi  gp, gp, 0      # offset 的低位也是 0
```

而 `addi gp, gp, 0` 就是 `mv gp, gp`！

### 為什麼這是錯的？

```c
// 你期望的：設定 gp 為全域資料區的位址
gp = &__global_pointer$;

// 實際發生的：gp 的值沒變
gp = gp + 0;  // 等於什麼都沒做！
```

## 正確的寫法

要解決這個問題，必須**禁止 linker relaxation**：

### 完整的正確版本

```c
void boot_init(void) {
    __asm__ volatile(
        ".option push \n"
        ".option norelax \n"
        "la gp, __global_pointer$ \n"
        ".option pop \n"
    );
}
```

### 反組譯結果

```asm
00000000 <boot_init>:
   0:   00001197          auipc   gp,0x1
   4:   80018193          addi    gp,gp,-2048  # ← 正確載入位址！
```

### 逐行解釋

```asm
.option push              # 保存當前編譯選項
.option norelax           # 禁止連結器最佳化
la gp, __global_pointer$  # 強制使用完整的指令序列
.option pop               # 恢復先前的編譯選項
```

## 深入理解

### 為什麼需要 .option push/pop？

```asm
.option push       # 儲存目前的設定
.option norelax    # 臨時改變設定
# ... 你的程式碼 ...
.option pop        # 恢復原本的設定（可能是 relax）
```

這樣可以：
- 只在必要時禁用最佳化
- 不影響其他程式碼的最佳化
- 保持程式碼的整體效率

### Linker Relaxation 的其他案例

除了 `gp` 之外，relaxation 還會影響：

```asm
# CALL 指令的優化
call far_function
# 可能被優化為：
jal  far_function   # 如果距離夠近

# LUI + ADDI 的優化
lui  a0, %hi(symbol)
addi a0, a0, %lo(symbol)
# 可能被優化為：
addi a0, zero, immediate  # 如果 symbol 是小數值
```

## 完整的 Boot Code 範例

```c
// crt0.S 或 startup.c

extern char __global_pointer$;
extern char __stack_top;

void _start(void) {
    // 1. 設定堆疊指標
    __asm__ volatile("la sp, __stack_top");
    
    // 2. 設定全域指標（正確寫法）
    __asm__ volatile(
        ".option push \n"
        ".option norelax \n"
        "la gp, __global_pointer$ \n"
        ".option pop \n"
    );
    
    // 3. 清除 BSS 段
    extern char __bss_start, __bss_end;
    for (char *p = &__bss_start; p < &__bss_end; p++) {
        *p = 0;
    }
    
    // 4. 初始化 data 段（如果需要）
    // copy_data_section();
    
    // 5. 跳轉到 main
    main();
    
    // 6. 停止（如果 main 返回）
    while(1) {
        __asm__ volatile("wfi");  // Wait For Interrupt
    }
}
```

## 除錯技巧

### 1. 檢查反組譯

```bash
riscv64-unknown-elf-objdump -d your_program.elf | grep -A5 "_start"
```

### 2. 查看連結器的 relaxation

```bash
riscv64-unknown-elf-ld --verbose your_program.o
```

### 3. 驗證 gp 的值

```c
// 在除錯器中確認
void check_gp(void) {
    unsigned long gp_val;
    __asm__ volatile("mv %0, gp" : "=r"(gp_val));
    printf("gp = 0x%lx\n", gp_val);
    printf("__global_pointer$ = 0x%lx\n", (unsigned long)&__global_pointer$);
}
```

## 常見錯誤

### ❌ 忘記 norelax

```c
__asm__ volatile("la gp, __global_pointer$");
// 可能產生 mv gp, gp
```

### ❌ 順序錯誤

```c
__asm__ volatile(
    "la gp, __global_pointer$ \n"
    ".option norelax \n"  // 太遲了！
);
```

### ❌ 沒有 pop

```c
__asm__ volatile(
    ".option norelax \n"
    "la gp, __global_pointer$ \n"
    // 缺少 .option pop，影響後續程式碼
);
```

## 總結

從一個看似簡單的 `mv gp, gp` 指令，我們學到了：

1. **RISC-V 的 linker relaxation** 會自動優化指令，但有時會出錯
2. **初始化 gp 必須使用 `.option norelax`** 來確保正確載入位址
3. **`.option push/pop`** 是良好的實踐，避免影響其他程式碼
4. **反組譯是你的好朋友**，遇到問題時要檢查實際產生的機器碼

### 記住這個模式

```c
__asm__ volatile(
    ".option push \n"
    ".option norelax \n"
    "la gp, __global_pointer$ \n"
    ".option pop \n"
);
```

這是 RISC-V bare-metal 開發的標準寫法，幾乎所有的啟動程式碼都應該這樣寫。

## 參考資料

- [RISC-V ELF psABI Specification](https://github.com/riscv-non-isa/riscv-elf-psabi-doc)
- [GNU Binutils Documentation - RISC-V](https://sourceware.org/binutils/docs/as/RISC_002dV_002dDirectives.html)
- [RISC-V Assembly Programmer's Manual](https://github.com/riscv-non-isa/riscv-asm-manual)
- [elfnn-riscv.c - Linker Relaxation 實作](https://github.com/bminor/binutils-gdb/blob/master/bfd/elfnn-riscv.c)

---

**系列文章：**
- RISC-V 暫存器約定完整解析
- 從零開始的 RISC-V Bare-metal 開發
- 理解 RISC-V 的記憶體模型