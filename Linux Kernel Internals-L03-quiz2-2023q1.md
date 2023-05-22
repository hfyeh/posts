---
layout: post
title: "Linux 核心設計 L03: quiz2 (2023q1)"
date: 2023-05-11
tags: [blog, draft]
draft: false
---

This is the homework of [jserv](https://github.com/jserv)'s [Linux Kernel Internals](https://wiki.csie.ncku.edu.tw/linux/schedule).

Reference: [L03: quiz2](https://hackmd.io/@sysprog/H143OpNCo)

<!--more-->

## 測驗 1

### 題目

原題目提供此程式碼，用來取得大於等於 `x` 的最接近的 2 的冪：

```c
uint64_t next_pow2(uint64_t x)
{
    x |= x >> 1;
    x |= x >> 1;
    x |= x >> 1;
    x |= x >> 1;
    x |= x >> 1;
    x |= x >> 1;
    x |= x >> 1;
    x |= x >> AAAA;
    x |= x >> 16;
    x |= x >> BBBB;
    return CCCC;
}
```

其效用為填補位元表示中的 1：

```
x = 0010000000000000
x = 0011000000000000
x = 0011110000000000
x = 0011111111000000
x = 0011111111111111
```

最初 `x` 是 `0010000000000000`，經過一系列操作後，成為 `0011111111111111`，亦即設定 (set，即指派為 `1`) 自原本最高位元到最低位元中，所有的位元。

請補完程式碼，使其符合預期。作答規範:

-   `AAAA` 和 `BBBB` 皆為數值，且 `AAAA` 小於 `BBBB`
-   `CCCC` 為表示式

### 作答

`x |= x>>1;` 是將 `x` 做 bit-wise 向右移動一位後，再與原本的數值取邏輯或。

```c
// x = 01001001
x |= x >> 1;
// x>>1 00100100
// x|x>>1 01101101
```

該程式連續進行相同的操作共七次，效用為將 `x` 之非零的最高位元設定到其後的 7 個位元，連同最高位本身，共有 8 個連續的 1。

最終目標是將最高非零位以下的所有位都填 1，可知 `AAAA` 為 `8`，作用為一次將最高非零以降的 8 個 1 向右位移 8 位。

其後經過 `x |= x >> 16` 操作後，最高非零位以下的 32 位皆為 1，故 `BBBB` 為 `32`，將該 32 位 1 往右位移 32 位並覆蓋。

考慮題目提供的幾個數值
-   `next_pow2(7)` = 8
-   `next_pow2(13)` = 16
-   `next_pow2(42)` = 64

```c
7  => 0000 0111
13 => 0000 1101
42 => 0010 1010
```

在執行到 `CCCC` 之際，其值為

```c
7  => 0000 0111
15 => 0000 1111
63 => 0011 1111
```

經過 `CCCC` 此表達式後，將得到

```c
8  => 0000 1000
16 => 0001 0000
64 => 0100 0000
```

可知 `CCCC` 為 `x+1`。

> [!question] 延伸問題
> 1.  解釋上述程式碼原理，並用 [`__builtin_clzl`](https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html) 改寫    
>> [!info]    > `int __builtin_clz (unsigned int x)`  
>> Returns the number of leading 0-bits in x, starting at the most significant bit position. If x is 0, the result is undefined.  
>> `int __builtin_clzl (unsigned long)`  
>> Similar to `__builtin_clz`, except the argument type is unsigned long.
    > 2.  在 Linux 核心原始程式碼找出類似的使用案例並解釋
    > 3. 當上述 `clz` 內建函式已運用時，編譯器能否產生對應的 x86 指令？
    > 提示: 可執行 `cc -O2 -std=c99 -S next_pow2.c` 並觀察產生的 `next_pow2.s` 檔案，確認 `bsrq` 指令 (表示 “[Bit Scan Reverse](https://c9x.me/x86/html/file_module_x86_id_20.html)”)
 
1. 

由於當 `x` 為 0 時，`__builtin_clzl` 的實作是 undefined behavior，故先對 `x==0` 的情況做處理。其餘情況，先計算最高非零位以下的位數，再將 `1UL` 右移該位數，即可得到最接近 `x` 且大於等於 `x` 的 2 的冪。

```c
uint64_t next_pow2_by_builtin_clzl(uint64_t x) {
    if (!x)
        return 1;
    return 1UL << (64 - __builtin_clzl(x));
}
```

2. 

```c
/**
 * seq_put_hex_ll - put a number in hexadecimal notation
 * @m: seq_file identifying the buffer to which data should be written
 * @delimiter: a string which is printed before the number
 * @v: the number
 * @width: a minimum field width
 *
 * seq_put_hex_ll(m, "", v, 8) is equal to seq_printf(m, "%08llx", v)
 *
 * This routine is very quick when you show lots of numbers.
 * In usual cases, it will be better to use seq_printf(). It's easier to read.
 */
void seq_put_hex_ll(struct seq_file *m, const char *delimiter,
				unsigned long long v, unsigned int width)
{
	unsigned int len;
	int i;

	if (delimiter && delimiter[0]) {
		if (delimiter[1] == 0)
			seq_putc(m, delimiter[0]);
		else
			seq_puts(m, delimiter);
	}

	/* If x is 0, the result of __builtin_clzll is undefined */
	if (v == 0)
		len = 1;
	else
		len = (sizeof(v) * 8 - __builtin_clzll(v) + 3) / 4;

	if (len < width)
		len = width;

	if (m->count + len > m->size) {
		seq_set_overflow(m);
		return;
	}

	for (i = len - 1; i >= 0; i--) {
		m->buf[m->count + i] = hex_asc[0xf & v];
		v = v >> 4;
	}
	m->count += len;
}

```

在 fs/seq_file.c 找到 `__builtin_clzll`。此函式之作用為，將整數 `v` 以十六進位表示。

`unsigned long long` 位元長度至少為 64 bits，以 64 bits，值域範圍為 $0$ 至 $2^{64}$。

其中使用到 `__builtin_clzll` 的附近程式碼為
```c
	if (v == 0)
		len = 1;
	else
		len = (sizeof(v) * 8 - __builtin_clzll(v) + 3) / 4;
```

在 else 分支，`sizeof(x) * 8` 即為 `v` 實際的位元長度（以 bit 為單位）。`v` 的位元長度，減去 `__builtin_clzll(V)` 即為 MSB 算起的第一個非零位直到 LSB 的位元長度。其後除以 4 是為了將前述的位元長度轉換為十六進制值對應的位元長度，而 +3 是為了無條件進位。

假設 `v==0000 0000 0000 0111`，`sizeof(v) * 8` 得到 64，而 `__builtin_clzll(v)` 為 61，最後得到 `len = (64-61+3)/4 = 1`，可知只需要一個十六進制的位即可表達 `v`。

3. 

```c
next_pow2_by_builtin_clzl:              # @next_pow2_by_builtin_clzl
	.cfi_startproc
# %bb.0:
	testq	%rdi, %rdi
	je	.LBB1_1
# %bb.2:
	bsrq	%rdi, %rcx
	xorl	$63, %ecx
	negb	%cl
	movl	$1, %eax
                                        # kill: def $cl killed $cl killed $rcx
	shlq	%cl, %rax
	retq
.LBB1_1:
	movl	$1, %eax
	retq
.Lfunc_end1:
	.size	next_pow2_by_builtin_clzl, .Lfunc_end1-next_pow2_by_builtin_clzl
	.cfi_endproc
                                        # -- End function
	.globl	main                            # -- Begin function main
	.p2align	4, 0x90
	.type	main,@function

```

`bsrq` 為 bit scan reverse，參照[文件](https://c9x.me/x86/html/file_module_x86_id_20.html)，其作用為

> Description
> Searches the source operand (second operand) for the most significant set bit (1 bit). If a most significant 1 bit is found, its bit index is stored in the destination operand (first operand). The source operand can be a register or a memory location; the destination operand is a register. The bit index is an unsigned offset from bit 0 of the source operand. If the content source operand is 0, the content of the destination operand is undefined.


## 測驗 2

### 題目

給定一整數 $n$ ，回傳將 1 到 $n$ 的二進位表示法依序串接在一起所得到的二進位字串，其所代表的十進位數字 mod $10^9$+7 之值。

以下是可能的實作:

```c
int concatenatedBinary(int n)
{
    const int M = 1e9 + 7;
    int len = 0; /* the bit length to be shifted */
    /* use long here as it potentially could overflow for int */
    long ans = 0;
    for (int i = 1; i <= n; i++) {
        /* removing the rightmost set bit
         * e.g. 100100 -> 100000
         *      000001 -> 000000
         *      000000 -> 000000
         * after removal, if it is 0, then it means it is power of 2
         * as all power of 2 only contains 1 set bit
         * if it is power of 2, we increase the bit length
         */
        if (!(DDDD))
            len++;
        ans = (i | (EEEE)) % M;
    }
    return ans;
}
```

請補完程式碼，使其符合預期。作答規範:

-   `DDDD` 和 `EEEE` 皆為表示式

### 作答

考慮下列數值的二進位值，以及所需位數。

1 => 0001 => 1
2 => 0010 => 2
3 => 0011 => 2
4 => 0100 => 3
5 => 0101 => 3
6 => 0110 => 3
7 => 0111 => 3
8 => 1000 => 4

`len` 之目的為當前數字 `i` 的 bit length。

因為每逢 2 的冪都需要增加長度，故 `DDDD` 須判斷是否為 2 的冪。
參考 [Bit Twiddling Hacks](https://graphics.stanford.edu/~seander/bithacks.html) 可得 `DDDD` 為 `i & (i - 1)`。（`i` 不為零）

得到當前數字的 bit length 後，再將當前數字 `i` 與串接數字 `ans` 串接起來。可知 `EEEE` 為 `ans << len`。

取 modulo 的步驟放在迴圈內每一輪都執行，與放在迴圈外只在回傳前執行，結果是相同的。這是因為對於 `ans` 做乘和加，與對 `ans % M` 做相同的乘和加結果相同。放在迴圈內的好處是，可以避免溢位。

> [!question] 延伸問題
> 1.  解釋上述程式碼運作原理
> 2. 嘗試使用 [`__builtin_{clz,ctz,ffs}`](https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html) 改寫，並改進 mod $10^9$+7 的運算

1. 已於作答區說明。
2. `__builtin_{clz,ctz,ffs}` 的作用分別如下

>Built-in Function: `int` **__builtin_ffs** `(int x)`[](https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html#index-_005f_005fbuiltin_005fffs)
>Returns one plus the index of the least significant 1-bit of x, or if x is zero, returns zero.

> Built-in Function: `int` **__builtin_clz** `(unsigned int x)`[](https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html#index-_005f_005fbuiltin_005fclz)
> Returns the number of leading 0-bits in x, starting at the most significant bit position. If x is 0, the result is undefined.

> Built-in Function: `int` **__builtin_ctz** `(unsigned int x)`[](https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html#index-_005f_005fbuiltin_005fctz)
> Returns the number of trailing 0-bits in x, starting at the least significant bit position. If x is 0, the result is undefined.

`__builtin_ctz` 可以回傳 LSB 起至最低非零位之後的零的個數。搭配 `__builtin_clz`，可將程式改為

```c
// Original
if (!(i & (i-1)))
    len++;

// Modified
if (__builtin_clz(i) + __builtin_ctz(i) == (sizeof(i)<<3) - 1)
    len++;
```

## 測驗 3

### 題目

UTF-8 字元可由 1, 2, 3, 4 個位元組構成。其中單一位元組的 UTF-8 由 ASCII 字元構成，其 MSB 必為 0。

UTF-8 的多位元組字元是由一個首位元組和 1, 2 或 3 個後續位元組 (continuation byte(s)) 所構成。後續位元組的最高 2 個位元會設定為 10。對於首位元組，最高的 2 個位元始終為 11，下表展現多位元組字元的規則:

| Number of bytes | MSB Pattern                             |
| --------------- | --------------------------------------- |
| ASCII           | 0xxx.xxxx                               |
| 2 bytes         | 110x.xxxx 10xx.xxxx                     |
| 3 bytes         | 1110.xxxx 10xx.xxxx 10xx.xxxx           |
| 4 bytes         | 1111.0xxx 10xx.xxxx 10xx.xxxx 10xx.xxxx |

若輸入的字串是一個有效的 UTF-8 序列，則計算其中的後續位元組數量，並將此數字從總字串長度中減去，即可確定字元數量。

```c
#include <stddef.h>
#include <stdint.h>

size_t count_utf8(const char *buf, size_t len)
{
    const int8_t *p = (const int8_t *) buf;
    size_t counter = 0;
    for (size_t i = 0; i < len; i++) {
        /* -65 is 0b10111111, anything larger in two-complement's should start
         * new code point.
         */
        if (p[i] > -65)
            counter++;
    }
    return counter;
}

size_t swar_count_utf8(const char *buf, size_t len)
{
    const uint64_t *qword = (const uint64_t *) buf;
    const uint64_t *end = qword + len >> 3;

    size_t count = 0;
    for (; qword != end; qword++) {
        const uint64_t t0 = *qword;
        const uint64_t t1 = ~t0;
        const uint64_t t2 = t1 & 0x04040404040404040llu;
        const uint64_t t3 = t2 + t2;
        const uint64_t t4 = t0 & t3;
        count += __builtin_popcountll(t4);
    }

    count = (1 << AAAA) * (len / 8)  - count;
    count += (len & BBBB) ? count_utf8((const char *) end, len & CCCC) : DDDD;

    return count;
}
```

請補完程式碼，使其運作符合預期。作答規範:
-   `AAAA`, `BBBB`, `CCCC`, `DDDD` 皆為常數
-   以最精簡的方式展現

### 作答

`qword` 為指向 64 bits unsigned int 的指標，一開始指向 `buf`。隨後在迴圈內，一次偏移 8 bytes。`end` 指向包含 `buf` 最末端的 8 bytes，為迴圈終止條件。如 `buf` 總長度無法被 8 bytes 整除，餘下的最後一節在迴圈中不會被處理。

迴圈結束時，`count` 值為 continuation byte 的總數，題目要求的是 UTF-8 字元的個數（即非 continuation byte 的總數）。

計算迴圈掃過的 bytes 數，減去 count 就是迴圈內的 UTF-8 字元數。故 `AAAA` 為 `3`。

接著處理 `buf` 最末段無法被 8 bytes 整除的部份（如果有的話）。首先 `BBBB` 為輔助判斷是否能被整除，故為 `0x111`，如 `(len & 0x111)` 不為零，代表 `len` 無法被 8 整除。可以被整除的情況下，不應累加任何值，故 `DDDD` 為零。

`count_utf8` 第二個參數為 byte 數，故 `CCCC` 亦為 `0x111`。

> [!question] 延伸問題
> 1.  解釋上述程式碼運作原理，比較 SWAR 和原本的實作效能落差
> 2.  在 Linux 核心原始程式碼找出 UTF-8 和 Unicode 相關字串處理的程式碼，探討其原理，並指出可能的改進空間

1. 
![](https://i.imgur.com/dKGAoig.png)

2. 

## 測驗 4

### 題目

以下程式碼可判定 16 位元無號整數是否符合特定樣式 (pattern):

```c
#include <stdint.h>
#include <stdbool.h>

bool is_pattern(uint16_t x)
{
    if (!x)
        return 0;

    for (; x > 0; x <<= 1) {
        if (!(x & 0x8000))
            return false;
    }

    return true;
}
```

改寫上述程式碼，使其達到等價行為，但更精簡有效。

```c
bool is_pattern(uint16_t x)
{
    const uint16_t n = EEEE;
    return (n ^ x) < FFFF;
}
```

EEEE = ?  
FFFF = ?

### 作答

在迴圈中，每次 `x` 都會左移一位，左移至全部為零為止。其中只要任何一次左移後的 `x` 的 MSB 為 0 時，if 條件式便會成立，並回傳 false。由於輸入 0 也會回傳 false，可知只有滿足 `x = 0xf..f0..0`  的樣式時（從 MSB 起，至最低位的 1 之間都必須是 1），才會回傳 true。

當 `EEEE` 為 `~x + 1` 時，若滿足樣式，則 `n` 僅有一個 1，且與 `x` 最低位的 1 位置相同；反之，會有複數個 1。

舉例來說

```c
// 滿足樣式
x = 1111 1111 1100 0000
n = 0000 0000 0100 0000

// 不滿足樣式
x = 1111 1011 0000 0000
n = 0000 0101 0000 0000

x = 0000 1111 0000 0000
n = 1111 0001 0000 0000
```

取 `(x ^ n)`，以上述例子來說，會得到下列結果

```c
// 滿足樣式
x = 1111 1111 1100 0000
n = 0000 0000 0100 0000
(x ^ n) = 1111 1111 1000 0000

// 不滿足樣式
x = 1111 1011 0000 0000
n = 0000 0101 0000 0000
(x ^ n) = 1111 1110 0000 0000

x = 0000 1111 0000 0000
n = 1111 0001 0000 0000
(x ^ n) = 1111 1110 0000 0000
```

可知不滿足的情況下，`x ^ n` 一定比 `x` 大。故 `FFFF` 為 `x`。

> [!question] 延伸問題:
> 1.  解釋上述程式碼運作原理
> 2. 在 Linux 核心原始程式碼找出上述 bitmask 及產生器，探討應用範疇
>> 參見 [Data Structures in the Linux Kernel](https://0xax.gitbooks.io/linux-insides/content/DataStructures/linux-datastructures-3.html)
