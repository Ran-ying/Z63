# 通过 Flash 救砖

## 主板芯片

芯片丝印	| 对应固件 
-|-
Micron RW265 |	EG + LOADER + SNOR_USER	
Micron NW800 |	LI	
ST STM32F412ZEJ6 |	EX	
Renesas R5F56519AD |	VR	

## 1. RW265 — EXPEED 主固件存储芯片

容量 128 MiB，1.8V Serial NOR。根据仓库提取的分区表：
- 0x000000–0x15FFFF：LOADER，1.375 MiB
- 0x160000–0x597FFFF：EG 主系统，88.125 MiB
- 0x5980000–0x7FFFFFF：SNOR_USER，38.5 MiB

### 1.1 芯片信息

| 项目   | 参数                      |
| ---- | ----------------------- |
| 制造商  | Micron（美光）              |
| 型号   | MT35XU01GBBA1G12-0SIT   |
| 芯片标记 | RW265                   |
| 存储类型 | NOR Flash               |
| 容量   | 1Gbit（128MiB）           |
| 工作电压 | 1.8V（1.7–2.0V）          |
| 封装   | T-PBGA24，6 × 8mm，5×5 阵列 |
| 读取接口 | Extended SPI（SDR x1）    |

### 1.2 SPI 接线表

| 芯片引脚   | BGA 焊球位置 | 编程器连接              |
| ------ | -------- | ------------------ |
| VCC    | B4、D1、E4 | 1.8V               |
| VSS    | B3、C1、E5 | GND                |
| C      | B2       | SCLK               |
| S#     | C2       | CS#                |
| DQ0    | D3       | MOSI               |
| DQ1    | D2       | MISO               |
| DQ2/W# | C4       | 通过 10kΩ 电阻上拉至 1.8V |

### 1.3 接线要求

1. 编程器使用 1.8V 电源及兼容 1.8V 的 SPI 信号电平。
2. 三个 VCC 焊球连接至 1.8V，三个 VSS 焊球连接至 GND。
3. DQ2/W# 在单线 SPI 模式下作为写保护控制输入，建议通过 10kΩ 电阻上拉至 1.8V。
4. BGA 焊球位置按照 Micron 数据手册第 7 页的 Balls Down（焊球朝下）视角定义。

### 1.4 读取参数

| 参数           | 设置                |
| ------------ | ----------------- |
| 协议           | SPI 1-1-1（SDR x1） |
| SPI 模式       | Mode 0            |
| 初始时钟频率       | 1MHz              |
| JEDEC ID     | `2C 5B 1B`        |
| 读取 ID 指令     | `0x9F`            |
| 数据读取指令       | `0x13`            |
| 地址长度         | 4 字节              |
| Dummy Cycles | 0                 |
| 总容量          | 134,217,728 字节    |

### 1.5 读取芯片 ID

首先通过 SPI 发送 `0x9F` 指令，读取 JEDEC ID。

发送：

```text
9F 00 00 00
```

其中 `9F` 为读取 ID 指令，后续三个字节用于提供读取时钟。

预期返回：

```text
2C 5B 1B
```

三个字节分别为：

- `2C`：Micron 制造商 ID。
- `5B`：存储器类型。
- `1B`：容量标识。

确认 ID 一致后，开始读取存储数据。

### 1.6 读取 Flash 数据

使用 `0x13` 指令（4-BYTE READ）。

SPI 事务格式：

```text
CS# ↓

0x13
ADDR[31:24]
ADDR[23:16]
ADDR[15:8]
ADDR[7:0]

DATA[0]
DATA[1]读取
DATA[2]
...

CS# ↑
```

命令和地址通过 MOSI 发送，数据从 MISO 接收。命令、地址及连续数据读取期间保持 CS# 为低电平。

#### 1.6.1 读取起始地址

读取 `0x00000000`：

```text
13 00 00 00 00
```

随后持续提供 SPI 时钟，从 MISO 接收数据。

#### 1.6.2 读取指定地址

例如读取 `0x01234567`：

```text
13 01 23 45 67
```

芯片从指定地址开始连续输出数据。

#### 1.6.3 分块读取

建议以 4KB 或其他编程器支持的块大小进行读取。

每个读取事务使用：

```text
0x13 + 4字节起始地址 + N字节数据
```

读取完成后将数据依照起始地址顺序保存为原始二进制文件。

### 1.7 完整容量与地址范围

芯片容量为 1Gbit（128MiB），由两个 512Mbit Die 组成。

| 存储区域  | 起始地址         | 结束地址         | 容量    |
| ----- | ------------ | ------------ | ----- |
| Die 0 | `0x00000000` | `0x03FFFFFF` | 64MiB |
| Die 1 | `0x04000000` | `0x07FFFFFF` | 64MiB |

需要读取整个地址范围：

```text
0x00000000 – 0x07FFFFFF
```

读取第二个 Die 时应确认高地址区域能够正常访问，而不是重复返回第一个 Die 的数据。

完整读取文件应为：

```text
文件名：RW265.bin
文件大小：134217728 字节
```

### 1.8 数据验证

建议完整读取两次：

```text
RW265_read1.bin
RW265_read2.bin
```

分别计算 SHA-256：

```bash
sha256sum RW265_read1.bin RW265_read2.bin
```

两次读取结果应具有相同的 SHA-256。

同时确认：

1. 两个文件大小均为 134217728 字节。
2. 两次读取结果完全一致。
3. 高地址区域（`0x04000000` 之后）已正确读取。

确认后保留原始 BIN 文件，用于后续固件分析和软件测试。

### 1.9 参考资料

1. Micron《Xccela Flash Memory Data Sheet Brief》，Rev. A，2018。
2. Linux 内核 `drivers/mtd/spi-nor/micron-st.c`，MT35XU01GBBA 支持代码。https://github.com/torvalds/linux/blob/master/drivers/mtd/spi-nor/micron-st.c
3. NXP Linux SPI-NOR 驱动测试记录，2025-11-12。https://lists.openwall.net/linux-kernel/2025/11/12/722
4. Micron 官方芯片信息 https://www.micron.com/products/storage/nor-flash/xccela-flash/part-catalog/part-detail/mt35xu01gbba1g12-0sit
5. Micron 官方手册 https://s3.amazonaws.com/snapeda/datasheet/MT35XU512ABA1G12-0SIT_Micron_Technology_Inc.pdf

## 2. NW800 — Linux 固件存储芯片

容量 256 MiB，1.8V、8 位并行 SLC NAND。

LI 组件中的主要内容包括：
- ARM64 Linux Kernel：约 6.1 MB
- gzip initramfs：约 21.3 MB
- 两份设备树 DTB

逆向确认 LI 写入器以 2048 字节为单位处理 NAND 数据。

## 3. STM32F412ZEJ6 — EX 控制固件

固件组件 ex2010_0200a0.bi，去除外层 CRC 后为 512 KiB。

在反汇编中：
- 复位向量指向 0x080002B4
- 主应用程序入口位于 0x08020084
- 存在版本标记检查与独立的 RAM 恢复程序
这与 STM32F412 的内部 Flash 架构一致，因此 EX 对应的是这颗 STM32 内部的 Flash，而非 RW265。

## 4. R5F56519AD — VR 控制固件

固件 vr2010_010000.bi 是一组分块写入记录，而不是普通的连续二进制文件。

去除记录头后，实际固件数据为 819,200 字节，目标地址覆盖：
- 0xFFF00000–0xFFFC7FFF
- 总共 3,200 个 256 字节数据块
结合 RX 指令集和主板上实际发现的 Renesas MCU，可以较高可信度地将 VR 映射到它的内部程序 Flash。