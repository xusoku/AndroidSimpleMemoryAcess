
# Android简单内存映射与访问
-------
在数据访问中，内存的访问速度肯定是最快的，所以对于有些文件需要频繁高效访问的时候就可以考虑使用内存映射进行直接读写操作，代替IO读写，达到更高的效率。下面就是要简单说下，怎么来实现简单的Android内存映射。

大致需求

先说下，我这边的需求，多个应用需要读写同一个标记位，且在Android的Input事件系统层也要去读取此标记位，而且频率非常大。所以之前的读写文件法，写系统属性法，对于这种高效率要求的明显已经不能满足了。所以就得考虑直接读取内存值的方法了。

C++层映射

C++已经很久没有接触了，好多都已经还给老师了。（忧伤。。。）所以下面有些不对的地方请指出，谢谢。

基础准备

C++ open函数介绍

打开文件是很常见的一个函数了，但是里面有很多的参数需要注意，所以在这里进行大概的介绍下。
头文件：#include #include #include 
函数定义：

```c
int open(const char * pathname, int flags);
int open(const char * pathname, int flags, mode_t mode);
```

函数说明：

参数 pathname： 指向欲打开的文件路径字符串.
参数flags 所能使用的标志，想读写权限，是否新建等。
参数mode_tmodel 主要文件权限，只有新建文件才有效。
参考地址：http://c.biancheng.net/cpp/html/238.html

C++ mmap函数介绍

在c++中有一个专门的函数是做内存映射的，函数为mmap，它的方法的全称及参数如下:
void *mmap(void *start,size_t length,int prot,int flags,int fd,off_t offsize);
下面是各个参数的意义参考地址mmap参数介绍,我这里搬过来方便查看。

参数start：指向欲映射的内存起始地址，通常设为 NULL，代表让系统自动选定地址，映射成功后返回该地址。

参数length：代表将文件中多大的部分映射到内存。

参数prot：映射区域的保护方式。可以为以下几种方式的组合：
PROT_EXEC 映射区域可被执行
PROT_READ 映射区域可被读取
PROT_WRITE 映射区域可被写入
PROT_NONE 映射区域不能存取

参数flags：影响映射区域的各种特性。
在调用mmap()时必须要指定MAP_SHARED 或MAP_PRIVATE。
MAP_FIXED
如果参数start所指的地址无法成功建立映射时，则放弃映射，不对地址做修正。通常不鼓励用此旗标。
MAP_SHARED
对映射区域的写入数据会复制回文件内，而且允许其他映射该文件的进程共享。
MAP_PRIVATE 对映射区域的写入操作会产生一个映射文件的复制，即私人的“写入时复制”（copy on write）对此区域作的任何修改都不会写回原来的文件内容。
MAP_ANONYMOUS 建立匿名映射。此时会忽略参数fd，不涉及文件，而且映射区域无法和其他进程共享。
MAP_DENYWRITE只允许对映射区域的写入操作，其他对文件直接写入的操作将会被拒绝。
MAP_LOCKED 将映射区域锁定住，这表示该区域不会被置换（swap）。

参数fd：要映射到内存中的文件描述符。如果使用匿名内存映射时，即flags中设置了MAP_ANONYMOUS，fd设为-1。有些系统不支持匿名内存映射，则可以使用fopen打开/dev/zero文件，然后对该文件进行映射，可以同样达到匿名内存映射的效果。

参数offset：文件映射的偏移量，通常设置为0，代表从文件最前方开始对应，offset必须是分页大小的整数倍。

返回值：若映射成功则返回映射区的内存起始地址，否则返回MAP_FAILED(－1)，错误原因存于errno 中。

Android FileChannel 及 MappedByteBuffer介绍

FileChannel和MappedByteBuffer实际为nio部分的函数，它提供了映射map相应的方法，相关的参考地址如下：

FileChannel： FileChannel map方法介绍)
MappedByteBuffer MappedByteBuffer介绍 想要查看get和put方法，请查看抽象父类ByteBuffer的get方法参考地址)
打开文件并映射到内存

首先定义文件描述符，int offset_fd;,然后打开文件，代码如下：
```c
offset_fd = open("/tmp/memory_map", O_RDWR | O_CREAT, S_IRWXU | S_IRWXG | S_IRWXO);
```
第一个参数：指定要打开的文件名
第二个参数：指定打开方式为读写，且不存在文件则创建。
第三个参数：创建文件时指定文件的权限。
很多参数请参考上面C++ open的参考链接。
如果offset_fd 不等于 -1 则表示打开成功，打开文件后使用mmap进行文件的内存映射，先定义内存映射的起始地址指针以及需要读取标记位的指针，然后用mmap进行映射代码如下（mmap参数参考上面链接）：

```c
int* yoffset;
int* mFlag;
yoffset = (int *)mmap(0, 8, PROT_READ|PROT_WRITE, MAP_SHARED, offset_fd, 0);
mFlag = yoffset + 1;
```
// +1 为往后移动1位，因为是int型指针也就是往后移动4位。当然不移动也没有问题。
想访问标记为mFlag的值，则使用*mFlag则可以进行赋值以及取值。所以在c++层的读取赋值就这么简单的完成了，主要是对函数的参数要了解。

Android应用读写指定内存值

由于下层已经是把文件映射到内存中的，所以上层也可以直接文件读到内存中，然后读取相应位的值就可以了，也比较简单。
直接看下代码：

```java
private MappedByteBuffer memoryMap = null;
private void initMemoryMap() {
		if (memoryMap == null) {
			RandomAccessFile raf = null;
			try {
				// 和前面c++映射的文件名一致。
				raf = new RandomAccessFile("/tmp/memory_map", "rw");
				FileChannel fc = raf.getChannel();
				memoryMap = fc.map(FileChannel.MapMode.READ_WRITE, 0, 16);
			} catch (FileNotFoundException e) {
				e.printStackTrace();
			} catch (IOException e) {
				e.printStackTrace();
			} finally {
				if (raf != null) {
					try {
						raf.close();
					} catch (IOException e) {
						e.printStackTrace();
					}
				}
			}
		}
	}
```
通过FileChannel的map(),可以将指定区域范围的文件直接读到内存中，返回MappedByteBuffer类型,这里称之为内存映射.然后通过MappedByteBuffer取或写对应标记位数据。
如何取呢？通过memoryMap.get(index) 来取指定位置的字节数据，index根据标记位的位置来确认，比如前面mFlag的标记位为是在文件头向后偏移了一个4个字节，所以这里要取相同的值则是要使用memoryMap.get(4)即可，如果要设置标记位的值可以使用put(index,value)函数，例如：memoryMap.put(4,(byte)1);
其实在Android上层也很简单，相当于读文件，把文件描述符映射到内存中，这种方式比每次进行文件IO操作肯定快很多。

总结

实际看下来内容上很少，只是对有些函数进行了一些的介绍。

C++ open函数
C++ mmap函数
Java FileChannel类
Java MappedByteBuffer使用
其实说白了，就是要多掌握写知识点，把这些小小的知识积累起来能够串联起来，就能够做很多的优化。

