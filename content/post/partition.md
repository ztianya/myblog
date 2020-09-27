---
title: "分区格式探讨"
date: 2020-09-26T20:13:53+08:00
draft: false
---
分区格式目前有MBR和GPT两种，MBR是早期BIOS使用的磁盘分区格式，包含了操作系统的引导代码以及磁盘分区表信息。MBR信息位于硬盘第一个扇区，也就是LBA 0。众所周知，MBR格式只支持定义4个主分区，分区大小也被限制在2TB。随着UEFI标准的出现，作为UEFI的一部分，GPT分区方案也被广泛应用，支持更多的分区数量、更大的分区尺寸。同时为了兼容已有的工具，GPT分区也包含位于LBA 0的MBR。GPT格式中的MBR称为Protective MBR，之前的MBR称为Legacy MBR。

## Legacy MBR和Protective MBR
不管是Legacy MBR和Protective MBR，它们都位于LBA 0，并且格式都是一样的。整体结构如下图所示：
![mbr layout](/img/partition/mbr_layout.png)
- BootCode 是引导程序，由BIOS加载执行，UEFI不会执行这段程序
- Unique Disk Signature, 操作系统使用，用来区分磁盘
- Unknown, 2个字节，无用
- Partition Record 分区表，16*4字节，每条分区记录为16字节
- Signature, MBR的标志，固定值0xAA55

MBR包含4条分区表项，每条表项结构如下：
![partition entry](/img/partition/partition_entry.png)
- Boot Indicator, 0x80表示这是一个可引导的分区，UEFI不会使用
- Starting CHS, 分区的开始位置，CHS地址格式。
- OS Type，分区的类型，UEFI会使用，有两个值0xEF和0xEE.
  - 0xEF表示一个UEFI系统分区
  - 0xEE在Protective MRB中被使用，这条分区记录是GPT伪造的一个假分区，覆盖整个硬盘。因此不识别GPT结构的工具读取分区表时，只会看到这个分区。
- Ending CHS，分区的结束位置，CHS地址格式。
- Starting LBA，分区的起始LBA地址。
- Size In LBA，分区大小，包含的LBA数量。

Legacy MBR和Protective MBR的区别在于：
- Protective MBR的分区表中只有1条记录，OS Type为0xEE，分区大小覆盖整个磁盘。其他分区项都为0。
- Protective MBR的signature也是0xAA55，但其他字段都是0。
  
Legacy MBR中的分区表是实实在在的，真的会被使用，所有值都要保证正确。而Protective MBR的分区表是伪造的，不会被真正使用到，只是为了兼容一些不识别GPT格式的工具。

## 实践
有一个python的第三方包gpt可以解析gpt分区信息
查看MBR内容
```bash
pip3 install gpt
dd if=/dev/sda of=/root/mbr bs=512 count=1
cat /root/mbr | print_mbr
```

查看GPT header
```bash
dd if=/dev/sda of=/root/gpt_header bs=512 count=1 skip=1
cat /root/gpt_header | print_gpt_header
```

查看GPT分区表项
```bash
dd if=/dev/sda of=/root/partition_entry bs=512 count=32 skip=2
cat /root/partition_entry | print_gpt_partition_entry_array
```

如果把GPT磁盘第一个扇区的512字节破坏了，会有什么影响？破坏Protective mbr会影响系统内分区的识别，但由于Protective mbr并没有什么实质性的用途，内容也比较固定，所以容易恢复，直接dd写入固定的Protective mbr内容即可。

如何设置磁盘格式？
可以通过parted工具mklabel设置磁盘格式，除了gpt类型外，msdos表示Legacy mbr格式的硬盘类型。mklabel虽然不会删除数据，但会把分区表都清空掉。如果要恢复分区表，可以通过parted的rescue功能，rescue扫描磁盘中的内容，尝试识别其中的文件系统，从而判断是否存在分区。