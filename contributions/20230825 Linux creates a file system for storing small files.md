![](http://www.yanjun.pro/upload/2023/08/23082601.jpg)

今天群里一朋友遇到这样一个问题，明明硬盘只用了 30% 左右的空间，但是却无法写入文件，使用 `df -iT` 命令查看文件系统使用情况时，发现根目录的 inode 使用率确竟然是 100%，后来通过聊天得知，原来他的服务器主要用于存储 1KB 左右的小文件，这一下就破案了

inode 主要用来记录文件的属性，及此文件的数据所在的 block 编号，一个文件占用一个 inode，因此如果都是小文件的话，那么是存储不到多少文件就会把 inode 耗尽，出现文件系统明明还有很大的空闲空间，但无法写入文件的情况

其实在创建 ext4 文件系统时，我们可以使用 `-T small` 参数来获得更多的 inode，从而来优化对小文件的存储性能，接下来我们通过一个示例来看看效果

```bash
# 两块相同大小的硬盘
root@debian:~# lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sdb      8:16   0    1G  0 disk
└─sdb1   8:17   0 1023M  0 part
sdc      8:32   0    1G  0 disk
└─sdc1   8:33   0 1023M  0 part

# 使用默认参数给sdb创建文件系统
root@debian:~# /sbin/mkfs.ext4 /dev/sdb1
mke2fs 1.47.0 (5-Feb-2023)
Creating filesystem with 261888 4k blocks and 65536 inodes
Filesystem UUID: 8935c902-df71-4808-b547-c85b6fd37a46
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376

Allocating group tables: done
Writing inode tables: done
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done

# 使用-T参数sdc创建文件系统
root@debian:~# /sbin/mkfs.ext4 -T small /dev/sdc1
mke2fs 1.47.0 (5-Feb-2023)
Creating filesystem with 1047552 1k blocks and 262144 inodes
Filesystem UUID: f521096d-a5a1-41c9-bbf7-e6102e74e87a
Superblock backups stored on blocks:
        8193, 24577, 40961, 57345, 73729, 204801, 221185, 401409, 663553,
        1024001

Allocating group tables: done
Writing inode tables: done
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done

# 对比两个文件系统的inode数量
root@debian:~# mkdir default small
root@debian:~# mount /dev/sdb1 default/
root@debian:~# mount /dev/sdc1 small/
root@debian:~# df -iT
Filesystem     Type      Inodes IUsed   IFree IUse% Mounted on
/dev/sdb1      ext4       65536    11   65525    1% /root/default
/dev/sdc1      ext4      262144    11  262133    1% /root/small
```

从以上示例中我们可以看出，在使用 `-T small` 参数后，inode 数量多了进 20 万个

**注意：** 这样做也是有代价的，在使用默认参数创建 ext4 文件系统时，默认数据块大小为 4KB，而使用 `-T small` 参数后，数据块大小为 1KB，这就意味着我们存储一个同样大小的文件，使用 `-T small` 参数创建的文件系统存储该数据时，占用的数据块更多，数据更分散，如果文件较大，会直接影响文件的读取速度

> mke2fs（mkfs.ext4）的 `-T` 参数指定了如何使用该文件系统，以便 mke2fs 可以为该用途选择最佳的文件系统参数，其支持的使用类型在配置文件 ***/etc/mke2fs.conf*** 中定义，可以使用逗号分隔指定一个或多个使用类型

**inode 不足的解决方法**

- 删除文件大小为0的文件，可以使用 `find PATH -name "*" -type f -size 0c ` 查找

    **注意：** 使用 **-size** 参数时，不要用 **-size 1k**，这个表示占用空间为 1k，而不是文件大小为 1k，应该使用 **-size 1024c** 才表示文件大小为 1KB

- 定期对历史小文件进行打包、归档

---

作者简介：一个喜欢瞎折腾的 IT 技术人员，懂得不多，但是喜欢和有共同兴趣爱好的朋友交流、学习

------

via: http://www.yanjun.pro/?p=128
作者：[老颜随笔](http://www.yanjun.pro)
编辑：[wxy](https://github.com/wxy)

本文由贡献者投稿至 [Linux 中国公开投稿计划](https://github.com/LCTT/Articles/)，采用 [CC-BY-SA 协议](https://creativecommons.org/licenses/by-sa/4.0/deed.zh) 发布，[Linux中国](https://linux.cn/) 荣誉推出

