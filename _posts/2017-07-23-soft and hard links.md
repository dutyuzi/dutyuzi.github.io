---
layout:     post
title:      "Soft links differ from hard links"
subtitle:   ""
date:        2017/7/22 20:15:02 
author:     "MaJ"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - linux
---
---
# 什么是链接
LNIX文件系统提供了一种将不同文件链接至同一个文件的机制，我们称这种机制为链接。它可以使得单个程序对同一文件使用不同的名字。这样的好处是文件系统只存在一个文件的副本。系统简单地通过在目录中建立一个新的登记项来实现这种连接。该登记项具有一个新的文件名和要连接文件的inode（index node）号(inode与原文件相同)。不论一个文件有多少硬链接，在磁盘上只有一个描述它的inode，只要该文件的链接数不为0，该文件就保持存在。硬链接不能对目录建立硬链接。

**inode**

要解释清楚两者的区别和联系需要先说清楚 linux 文件系统中的 inode 这个东西。当划分磁盘分区并格式化的时候，整个分区会被划分为两个部分，即inode区和data block(实际数据放置在数据区域中）这个inode即是（目录、档案）文件在一个文件系统中的唯一标识，需要访问这个文件的时候必须先找到并读取这个 文件的 inode。 Inode 里面存储了文件的很多重要参数，其中唯一标识称作 Inumber, 其他信息还有创建时间（ctime）、修改时间(mtime) 、文件大小、属主、归属的用户组、读写权限、数据所在block号等信息。


# 软链接和硬链接
**软链接**：也称为符号链接，新建的文件以“路径”的形式来表示另一个文件，和Windows的快捷方式十分相似，新建的软链接可以指向不存在的文件。

**硬链接**：新建的文件是已经存在的文件的一个别名，当原文件删除时，新建的文件仍然可以使用。

# 区别
首先，从使用的角度讲，两者没有任何区别，都与正常的文件访问方式一样，支持读写，如果是可执行文件的话也可以直接执行。

那区别在哪呢？在底层的原理上。

为了解释清楚，我们首先在自己的一个工作目录下创建一个文件，然后对这个文件进行链接的创建：

    root@ubuntu:~# touch myfile && echo "hello,this is my test file">myfile
	root@ubuntu:~# cat myfile 
	hello,this is my test file
现在我们创建了一个普通地不能再普通的文件了。然后我们对它创建一个硬链接，并查看一下当前目录：

    root@ubuntu:~# ln myfile hard
	root@ubuntu:~# ls -li
	2103112 -rw-r--r--  2 root root   27 Jul 23 13:03 myfile
	2103112 -rw-r--r--  2 root root   27 Jul 23 13:03 hard

在 ls 结果的最左边一列，是文件的 inode 值，你可以简单把它想成 C 语言中的指针。它指向了物理硬盘的一个区块，事实上文件系统会维护一个引用计数，只要有文件指向这个区块，它就不会从硬盘上消失。
你也看到了，这两个文件就如同一个文件一样，inode 值相同，都指向同一个区块。然后我们修改一下刚才创建的 hard 链接文件：

	root@ubuntu:~# echo "change">hard
	root@ubuntu:~# cat myfile 
	change

可以看到这两个文件同时被修改了。
下面看一下软链接（又称**符号链接**）

	root@ubuntu:~# ln -s myfile soft
	root@ubuntu:~# ls 
	2103112 -rw-r--r--  2 root root    7 Jul 23 13:09 myfile
	2103113 lrwxrwxrwx  1 root root    6 Jul 23 13:11 soft -> myfile
你会发现，这个软链接的 inode 竟然不一样啊，并且它的文件属性上也有一个 l 的 flag，这就说明它与之前我们创建的两个文件根本不是一个类型。

下面我们试着删除 myfile 文件，然后分别输出软硬链接的文件内容：

	root@ubuntu:~# rm myfile 
	root@ubuntu:~# cat hard
	change
	root@ubuntu:~# cat soft
	cat: soft: No such file or directory
之前的硬链接没有丝毫地影响，因为它 inode 所指向的区块由于有一个硬链接在指向它，所以这个区块仍然有效，并且可以访问到。
然而软链接的 inode 所指向的内容实际上是保存了一个绝对路径，当用户访问这个文件时，系统会自动将其替换成其所指的文件路径，然而这个文件已经被删除了，所以自然就会显示无法找到该文件了。

接下来我们向软链接写入一些东西：

	root@ubuntu:~# echo "are you ok?">soft
	root@ubuntu:~# ls -li | grep myfile
	2103114 -rw-r--r--  1 root root   12 Jul 23 13:15 myfile
	2103113 lrwxrwxrwx  1 root root    6 Jul 23 13:11 soft -> myfile

可以看到，刚才删除的 myfile 文件竟然又出现了！这就说明，当我们写入访问软链接时，**系统自动将其路径替换为其所代表的绝对路径，并直接访问那个路径。**

# 总结

软链接可以看作是Windows中的快捷方式，可以让你快速链接到目标档案或目录。

硬链接则透过文件系统的inode来产生新档名，而不是产生新档案。

硬连接的作用是允许一个文件拥有多个有效路径名，这样用户就可以建立硬连接到重要文件，以防止“误删”的功能。

缺陷：

1.不允许给目录创建硬链接。

2.不可以在不同文件系统的文件间建立链接。因为 inode 是这个文件在当前分区中的索引值，是相对于这个分区的，当然不能跨越文件系统了。

软链接克服了硬链接的不足，没有任何文件系统的限制，任何用户可以创建指向目录的符号链接。因而现在更为广泛使用，它具有更大的灵活性，甚至可以跨越不同机器、不同网络对文件进行链接。

缺陷：

因为链接文件包含有原文件的路径信息，所以当原文件从一个目录下移到其他目录中，再访问链接文件，系统就找不到了，而硬链接就没有这个缺陷，你想怎么移就怎么移。

### 参考
[https://www.ibm.com/developerworks/cn/linux/l-cn-hardandsymb-links/](https://www.ibm.com/developerworks/cn/linux/l-cn-hardandsymb-links/)
[http://www.jianshu.com/p/dde6a01c4094](http://www.jianshu.com/p/dde6a01c4094)
[http://www.cnblogs.com/crazylqy/p/5821105.html](http://www.cnblogs.com/crazylqy/p/5821105.html)
