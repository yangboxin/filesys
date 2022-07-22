# 7/11
通过对ssd中的磨损均衡（wear leveling）技术的了解，认识到存在动态和静态两种磨损均衡类型，动态类型即每次需要写入时寻找空闲块池中最少擦除计数的块，并通过逻辑地址在转换映射表上建立彼此的映射关系，这样即使对同一逻辑地址多次写入也会自动映射到不同的物理地址，避免了对同一物理地址的重复使用，实现了磨损均衡，但是这种方法没有考虑到一些写入后就没有变化的数据块，比如多媒体文件等；此时就需要引入静态的磨损均衡，这种均衡方式对于闲置的块设定了一个阈值，若超过这个阈值会将其中的内容与空闲块池中最少擦除计数的块互换，于是可以达到一种整体均衡的效果。

值得注意的是，在flash文件系统的架构中，底层 flash 设备分配块、地址转换、磨损均衡和垃圾收集等都是在FTL（Flash Translation Layer）中完成，在我的理解中，这一层主要是因为ssd和传统的磁盘有很多不同的点，比如：写入之前要先擦除、NAND flash设备的写入擦除次数有限等等，所以需要在文件系统和存储实体之间有一个磁盘管理软件抽象层，方便使用。

f2fs文件系统（flash friendly file system），是专门针对ssd闪存特性设计的文件系统，从其名字中也可见一斑，它是一种LFS（Log-Structured File System），这种文件系统（LFS）的特性是out-place更新和顺序写入，但存在Wandering Tree、没有对数据进行分类等问题，f2fs通过引入NAT（Node Address Table)和冷热数据分离很好的解决了雪崩问题和垃圾回收的开销问题，但其仍存在一定不足，如：数据的冷热分离还比较简单，仅仅通过数据的类型（是否为目录、文件或多媒体文件等）来分类，可以结合实际的应用场景进行一定的优化和改良，这也是我们本次研究中希望解决的一个问题。

# 7/13
为了减少垃圾回收的开销，f2fs采用了多日志头的记录方式实现冷热数据分离，具体分类如下：

在f2fs文件系统中，冷热数据具体分为六类（默认），分别是hot、warm、cold的node和hot、warm、cold的data。hot node主要是目录的inode、直接索引，warm node是文件的inode、直接索引，cold node是间接索引块（用于地址映射转换）；hot data是目录数据块，warm data是文件数据块，cold data则是多媒体文件或者被垃圾回收的块。

在挂载时，可以选择2、4、6三种不同的冷热数据分块（using mount option such as active_logs=x），在实现的代码中通过一个enum类型来指定具体数据块或节点块的冷热程度。

通过以上标准进行冷热分离可以使得各个区域数据更新的频率接近，存储空间中各个segment的有效块（valid block）数量接近二项分布（即：冷数据大多保持有效而无需搬移，热数据大多更新后处于无效状态，只需少量搬移）。

引用技术博客中的一张图片来说明为何分类数据并分区域存放可以减少垃圾回收的开销 image node block与data block分离, 冷热数据分离，将不同数据存放到不同的zone中，有利于FTL进行GC垃圾回收，
![image](https://user-images.githubusercontent.com/55615299/179023809-739b17f3-5dbe-42d3-869b-e70dd328361b.png)
如上图，假设zone对应flash中的一个block，则zone-aware allocation可以做到冷热数据分配到不同的zone，也就分配到不同的flash block中

# 7/14
下载f2fs-tools并在本地编译成功，准备进一步的代码阅读、优化等工作，并上传至仓库。

# 7/15
经过对源代码的初步研究，定位到数据冷热分离的具体实现代码，具体过程仍待研究。

# 7/21
由于wsl是微软旗下一款使用windows系统调用来模拟linux内核的模拟器，其无法完成对linux内核模块的调用（包括f2fs、loop、fuse等），经过对虚拟机环境的调整（从wsl->wsl2->vmware），可在loop设备上挂载f2fs文件系统，并完成对文件的读写等操作。
具体步骤如下：
sudo get install f2fs-tools
使用f2fs-tools完成对块设备的格式化，使其可以以f2fs文件系统格式挂载。

dd if=/dev/zero of=fs.img bs=4K count=128000

losetup /dev/loop0 fs.img
创建一个块大小为4k（与f2fs吻合），总大小为500M的块设备

mkfs.f2fs -l f2fs /dev/loop0
格式化块设备

mkdir /mnt/f2fs

mount -t f2fs /dev/loop0 /mnt/f2fs
挂载到系统中

可以在/mnt/f2fs中使用touch、mkdir等操作
