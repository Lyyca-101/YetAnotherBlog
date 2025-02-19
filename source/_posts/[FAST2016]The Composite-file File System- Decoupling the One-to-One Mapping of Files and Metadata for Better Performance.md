---
title: paper-reading[FAST2016-CFFS]
---
<h2>[FAST2016]The Composite-file File System:
Decoupling the One-to-One Mapping
of Files and Metadata for Better Performance</h2>

现状：

- 每个逻辑文件都被映射到它自己的一个物理元数据以及数据表示
- 也就是一个1-to-1的文件-元数据映射
- 对很多的文件系统机制而言，这是一个自然的粒度(granularity)

根据作者的观察(observation)，这种1-to-1的映射可能错失了很多优化的机会，主要的观察有四个：

- 对小文件的频繁访问：作者的桌面文件系统分析表明，>80%的访问落在大小<32B的小文件上，而在对这些盘上的小文件数据的访问时间中，对元数据的访问时间占了40%，因此，减少这部分的访问开销将带来巨大的性能提升
- 冗余的元数据信息：传统文件关联的元数据，会记录文件的数据块位置、访问权限等，许多文件其实拥有相同或类似的文件属性，对于一些文件属性(访问权限、所有者等)，其可能情况是明显有限的，因此会有大量的文件在这些属性上相似，一个研究表明，对一个工作站的元数据进行压缩，压缩率达到75%(per-inode,128B-->15-20B)，有很多机会去减少冗余的元数据，从而提高空间利用率
- 成组访问文件：很多研究表明，文件倾向于被一起访问，但是仅仅将文件分组并不能得到性能提升，应为识别可分租的文件以及将文件分组会带来额外的开销
- 预取的局限：尽管预取是一个有效的优化手段，但是提取文件数据-文件元数据的分离动作会导致极大的额外开销，比如，32个小文件和1个和32个小文件一样大的单文件，访问前者的延迟会比后者高50%，哪怕是在已经被缓存的情况下

那么，如何减少对元数据的额外访问开销，减少相似元数据占用的空间，低开销地成组访问文件，作者提出一个idea，同时地去解决这些问题，即：

- 将一起访问的小文件组合在一起，解耦1-to-1的文件-元数据映射，构造many-to-1的文件-元数据映射

作者的设计：
- 多个会被用户一起访问的文件，共享一个inode，在物理上这些文件形成一个组合文件(composite file)
- 这些小文件的属性被存储为组合文件的扩展属性(xattr)，因此每个小文件的属性仍然能够被重建、更新以及检查
- 决定哪些文件能够被组合在一起是一个依赖工作负载的策略决定问题，CFFS有基于目录的、基于嵌入在文件中的文件引用的、基于频度挖掘的三种文件组合策略
- 在同一个组合文件中的子文件，会有一个文件被指定为entry point，对这个文件的访问会触发对其他子文件的预取，从而使得之后对其余子文件的访问能够直接在缓存完成，提升访问性能
- 基于目录的文件组合策略最大问题在于，不能将跨目录的文件合并成组合文件，而确实存在一些跨目录的文件，它们是会被一起访问的
- 关于空间压缩，因为会对组合文件进行子文件的添加和删除，对于组合文件中被删除的子文件，它对应的数据区被标记为已释放，当组合文件数据区超过一半没有有效数据(被标记为已释放)时，组合文件会进行压缩，以重新组织有效数据的排列



作者的实现：

- 主要是两个部分，一个是组合文件关系生成器，识别可以组合在一起的文件，另一个是基于FUSE实现的文件系统，用于拦截所有文件系统相关的系统调用，以添加设计相关的机制
- 文件系统可以利用组合文件关系生成器来获知哪些文件可以组合，从而构建组合文件



|       配置       |                             描述                             |
| :--------------: | :----------------------------------------------------------: |
| Operating system |                         Linux 3.16.7                         |
|    Processor     | 2.8GHz Intel® Xeon® E5-1603, L1 cache 64KB, L2 cache 256KB, L3 cache 10MB |
|      Memory      |                2GBx4, Hyundai, 1067MHz, DDR3                 |
|       Disk       |         250GB, 7200 RPM, WD2500AAKX with 16MB cache          |
|      Flash       |                  200GB, Intel SSD DC S3700                   |

以上是实验平台的配置表

对评估的分析

- 性能的提升得益于IO次数的减少，SSD 和 HDD 之间性能提升的差异表明，由于磁盘寻道减少和数据布局优化带来的性能增益可达约 10%

CFFS 的重点在于改变逻辑文件到物理表示的映射，并可以采用多种挖掘算法来整合元数据和优化存储布局。

[xattr_user.c - fs/ext4/xattr_user.c - Linux source code v5.15 - Bootlin Elixir Cross Referencer](https://elixir.bootlin.com/linux/v5.15/source/fs/ext4/xattr_user.c#L33)

[ext4.h - fs/ext4/ext4.h - Linux source code v5.15 - Bootlin Elixir Cross Referencer](https://elixir.bootlin.com/linux/v5.15/source/fs/ext4/ext4.h#L1012)记录EA块的位置

[xattr.c - fs/ext4/xattr.c - Linux source code v5.15 - Bootlin Elixir Cross Referencer](https://elixir.bootlin.com/linux/v5.15/source/fs/ext4/xattr.c#L2124)分配一个EA块

考虑到presentation中google开发者对作者提出的一些疑问，尽管design of CFFS是好的，但是由于作者FUSE-based的implementation，可能导致evaluation并没有呈现出真实的情况:
来自google的开发者在FAST的presentation上，评价到作者可能并没有"go into the filesystem(fs kernel module)"