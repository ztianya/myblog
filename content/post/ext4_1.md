---
title: "走进Ext4文件系统"
date: 2019-12-16T21:42:31+08:00
draft: false
---

最近开始研究Ext4文件系统，在此记录相关笔记，从Ext4的结构开始，走进文件系统的世界。

## Ext4文件系统结构

Ext4文件系统的最小存储单元是一个block，block大小通常为4KB，与x86的内存页大小保持一致。ext4文件系统将所有的block均分为不同的组，称为block group，所以一个ext4文件系统由一系列的block group组成。以下是一个block group的标准结构

| Group 0 Padding | ext4 Super Block | Group Descriptors | Reserved GDT Blocks | Data Block Bitmap | inode Bitmap | inode Table |      Data Blocks  |
| -------------- | -------------- | --------------- | ----------------- | --------------- | --------------- | --------------- | --------------- |
| 1024 bytes      |     1 block      |    many blocks    |    many blocks     |      1 block      |   1 block    | many blocks | many more blocks |

- Group 0 Padding: 每个ext4文件系统在开头会预留出1024bytes空间，可用于安装bootloader。因此，只有group 0的开头会预留这一段空间。
- ext4 Super Block: 超级块作为一个ext4文件系统的核心内容，是非常重要的。超级块中记录了inode数量，block数量，文件系统支持的feature等关键信息，占用一个block大小。如果Super block被破坏，这个文件系统就废掉了。既然super block如此重要，肯定要有备份以防万一。ext4有一个叫sparse_super的feature，与超级块的备份相关。如果初始化文件系统没有指定这个feature的话，所有group都会保存一份super block，这样显然很浪费。如果指定了sparse_super（默认都会指定的），super block只会在group 0以及编号为3，5,7的整数次幂的group上保存副本
- Group Descriptors: 每个block group都有一条对应的group descriptor,描述了该block group的inode table、inode bitmap以及block bitmap地址。Group Descriptors和后面的Resvered GDT Blocks同Super block一起备份。
- Reserved GDT Blocks: Group Descriptor的预留块，为文件系统扩容预留空间。
- Data Block Bitmap: 记录该block group中哪些block已被使用，哪些block空闲。大小为1个block
- inode Bitmap：记录该block group中哪些inode号已经被分配，哪些inode号空闲。大小为1个block
- inode Table: inode表，保存了该block group所有inode信息
- Data Blocks: 真正存储文件数据的地方。

Ext4文件系统有一个称为flexible block groups (flex_bg)的特性，如果flex_bg特性打开的话，文件系统会将多个block group关联在一起，组成一个更大的逻辑block group。在逻辑块组中，所有成员block group的bitmap和inode tables都会被集中存放在第一个成员block group中。这种方式将block group元数据放在一起，便于快速加载，同时大文件能在磁盘上连续存储。

flex_bg 打开之后，Ext4整体的结构如下图所示：

![ext4 layout](/img/ext4/ext4_layout.png)

## 实践：从inode号到数据块

inode是文件的索引，保存了文件数据块地址。要读取一个文件的数据块内容，前提是找到该文件的inode位置。由于EXT4文件系统以block作为最小存储单元，因此，要知道inode在哪儿，也就是要找出inode所在的block号以及inode在该block中的偏移。从上述EXT4的结构分析可以知道，inode又是保存在inode table中，每个文件在inode table里有一条inode 项。总结起来，要找到一个文件的inode所在，需要这三条信息：
1. inode所在inode tables的位置，也就是inode table的起始block号
2. inode在该inode tables中的偏移block
3. inode在block中的偏移地址

接下来，我们从一个文件inode号开始，走进EXT4内部，寻找该文件最终的数据块。在开始之前，首先通过dumpe2fs获取当前文件系统的元信息，包括block大小为4096字节，inode size为256字节，Flex block group size为16，Inodes per group为8192，Blocks per group为32768。
![super block](/img/ext4/super_block.jpg)

### 寻找到inode

通过stat可以获取到文件的inode号为3407888。

![stat](/img/ext4/stat.jpg)

文件所在的block group号为（3407888-1） / （Inodes per group）= 416 ，该block group的起始block号为416 * （Blocks per group）= 13631488。416正好是Flex blocks group size（16）的倍数，因此，该block group是一个flex group的起始group。416也不是3，5，7的整数次幂，这个group的开头不会保存超级块的副本。所以，该group的结构直接从Data Block Bitmap开始。由于flex group的第一个group保存了整个flex group中所有的Data Block Bitmap和inode Bitmap, 一共占用2* （Flex blocks group size）= 32个block。可以推算出，该block group的inode tables起始block号为 13631488 + 32 = 13631520。inode tables的起点找到了，接下来看偏移。inode在inode tables中的偏移block = ((3407888-1) % (Inodes per group * Flex block group size))/(block size / inode size) = 0，分子为inode在整个inode tables中的偏移数。至此，我们知道了inode所在的block号为13631520。最后，看下inode在block中的偏移，（（3407888-1） % (block size / inode size)) * inode size = 3840。因此，3407888这个inode位于13631520号block，且偏移为3840。注意，上述计算中将inode号减1是因为0号inode不存在。

### 寻找数据block地址
先贴上inode数据结构
```c
struct ext4_inode {
	__le16	i_mode;		/* File mode */
	__le16	i_uid;		/* Low 16 bits of Owner Uid */
	__le32	i_size_lo;	/* Size in bytes */
	__le32	i_atime;	/* Access time */
	__le32	i_ctime;	/* Inode Change time */
	__le32	i_mtime;	/* Modification time */
	__le32	i_dtime;	/* Deletion Time */
	__le16	i_gid;		/* Low 16 bits of Group Id */
	__le16	i_links_count;	/* Links count */
	__le32	i_blocks_lo;	/* Blocks count */
	__le32	i_flags;	/* File flags */
	union {
		struct {
			__le32  l_i_version;
		} linux1;
		struct {
			__u32  h_i_translator;
		} hurd1;
		struct {
			__u32  m_i_reserved1;
		} masix1;
	} osd1;				/* OS dependent 1 */
	__le32	i_block[EXT4_N_BLOCKS];/* Pointers to blocks */
	__le32	i_generation;	/* File version (for NFS) */
	__le32	i_file_acl_lo;	/* File ACL */
	__le32	i_size_high;
	__le32	i_obso_faddr;	/* Obsoleted fragment address */
	union {
		struct {
			__le16	l_i_blocks_high; /* were l_i_reserved1 */
			__le16	l_i_file_acl_high;
			__le16	l_i_uid_high;	/* these 2 fields */
			__le16	l_i_gid_high;	/* were reserved2[0] */
			__le16	l_i_checksum_lo;/* crc32c(uuid+inum+inode) LE */
			__le16	l_i_reserved;
		} linux2;
		struct {
			__le16	h_i_reserved1;	/* Obsoleted fragment number/size which are removed in ext4 */
			__u16	h_i_mode_high;
			__u16	h_i_uid_high;
			__u16	h_i_gid_high;
			__u32	h_i_author;
		} hurd2;
		struct {
			__le16	h_i_reserved1;	/* Obsoleted fragment number/size which are removed in ext4 */
			__le16	m_i_file_acl_high;
			__u32	m_i_reserved2[2];
		} masix2;
	} osd2;				/* OS dependent 2 */
	__le16	i_extra_isize;
	__le16	i_checksum_hi;	/* crc32c(uuid+inum+inode) BE */
	__le32  i_ctime_extra;  /* extra Change time      (nsec << 2 | epoch) */
	__le32  i_mtime_extra;  /* extra Modification time(nsec << 2 | epoch) */
	__le32  i_atime_extra;  /* extra Access time      (nsec << 2 | epoch) */
	__le32  i_crtime;       /* File Creation time */
	__le32  i_crtime_extra; /* extra FileCreationtime (nsec << 2 | epoch) */
	__le32  i_version_hi;	/* high 32 bits for 64-bit version */
	__le32	i_projid;	/* Project ID */
};
```

用blkcat分析一下inode信息，偏移量为3840。前16bit为mode信息，接下来16bit为uid，接下来的32bit是文件大小，0x23转成10进制为35, 与前面stat查看的文件大小一致。inode将data block相关信息保存在i_block字段中，接下来定位i_block的位置，根据数据结构定义得知，i_block在inode的偏移是320个字节。ext4使用extent结构记录数据块，这里不再赘述extent的具体细节。由于该文件较小，只用到了一个块保存数据。数据块地址的16进制为图中0xf14811,转换成10进制为15812625。

![inode](/img/ext4/inode.png)

通过blkcat查看15812625号块的内容，就是要找的测试文件。

![block content](/img/ext4/blk.jpg)

本文介绍了Ext4文件系统的相关概念和组成结构，并从一个inode号开始，通过层层计算，读取到最终的文件内容。一遍走下来，基本上对Ext4的整体结构有了深刻的理解。在利用i_block定位data block时，需要了解Ext4的extent原理，这块较为复杂，后续会重点介绍。

## 参考
- https://ext4.wiki.kernel.org/index.php/Ext4_Disk_Layout
- https://www.junmajinlong.com/linux/ext_filesystem/
