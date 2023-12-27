---
title: rCore设备管理(三)virtio设备驱动实现

date: 2023-11-27 17:00:00 +0800

categories: [读书笔记, rCore指导文档]

tags: [rCore, device, virtio]

description: 
---

# 1 virtio_blk块设备驱动

本节主要介绍了：

* 与操作系统无关的基本 `virtio-driver: virtio_blk` 设备驱动程序的设计与实现；
* 如何在操作系统中封装 `virtio_blk` 设备驱动程序，实现基于中断机制的I/O操作，提升计算机系统的整体性能。

`virtio-blk` 设备是一种virtio存储设备，在QEMU模拟的 RISC-V 64 计算机中，以MMIO和中断等方式来与驱动程序进行交互。这里我们以Rust语言为例，给出 `virtio-blk` 设备驱动程序的设计与实现。主要包括如下内容：

- `virtio-blk` 设备的关键数据结构
- `virtio-blk` 设备初始化（OS无关/OS相关）
- `virtio-blk` 设备的I/O处理（OS无关/OS相关）

## 1.1 virtio-blk设备的关键数据结构

这里我们首先需要定义 `virtio-blk` 设备的结构：

```rust
// virtio-drivers/src/blk.rs
pub struct VirtIOBlk<'a, H: Hal> {
   header: &'static mut VirtIOHeader,
   queue: VirtQueue<'a, H>,
   capacity: usize,
}
```

* `header` 成员对应的 `VirtIOHeader` 数据结构是virtio设备的共有属性，包括版本号、设备id、设备特征等信息，其内存布局和成员变量的含义与上一节描述 [virt-mmio设备的寄存器内存布局](https://rcore-os.cn/rCore-Tutorial-Book-v3/chapter9/2device-driver-2.html#term-virtio-mmio-regs) 是一致的；
* 而 `VirtQueue` 数据结构与上一节描述的 [virtqueue](https://rcore-os.cn/rCore-Tutorial-Book-v3/chapter9/2device-driver-2.html#term-virtqueue) 在表达的含义上基本一致的。

---

```rust
#[repr(C)]
pub struct VirtQueue<'a, H: Hal> {
   dma: DMA<H>, // DMA guard
   desc: &'a mut [Descriptor], // 描述符表
   avail: &'a mut AvailRing, // 可用环 Available ring
   used: &'a mut UsedRing, // 已用环 Used ring
   queue_idx: u32, //虚拟队列索引值
   queue_size: u16, // 虚拟队列长度
   num_used: u16, // 已经使用的队列项目数
   free_head: u16, // 空闲队列项目头的索引值
   avail_idx: u16, //可用环的索引值
   last_used_idx: u16, //上次已用环的索引值
}
```

* 其中成员变量 `free_head` 指空闲描述符链表头，初始时所有描述符通过 `next` 指针依次相连形成空闲链表，成员变量 `last_used_idx` 是指设备上次已取的已用环元素位置。成员变量 `avail_idx` 是可用环的索引值。
* 这里出现的 `Hal` trait是 virtio_drivers 库中定义的一个trait，用于**抽象出与具体操作系统相关的操作，主要与内存分配和虚实地址转换相关。**这里我们只给出 `Hal` trait的定义，对应操作系统的具体实现在后续的章节中会给出。

```rust
pub trait Hal {
   /// Allocates the given number of contiguous physical pages of DMA memory for virtio use.
   fn dma_alloc(pages: usize) -> PhysAddr;
   /// Deallocates the given contiguous physical DMA memory pages.
   fn dma_dealloc(paddr: PhysAddr, pages: usize) -> i32;
   /// Converts a physical address used for virtio to a virtual address which the program can
   /// access.
   fn phys_to_virt(paddr: PhysAddr) -> VirtAddr;
   /// Converts a virtual address which the program can access to the corresponding physical
   /// address to use for virtio.
   fn virt_to_phys(vaddr: VirtAddr) -> PhysAddr;
}
```

## 1.2 virtio-blk设备初始化

### virtio-drivers中的初始化过程

virtio-blk设备的初始化过程与virtio规范中描述的一般virtio设备的初始化过程大致一样，对其实现的初步分析在 [virtio-blk初始化代码](https://rcore-os.cn/rCore-Tutorial-Book-v3/chapter9/2device-driver-2.html#term-virtio-blk-init) 中。在设备初始化过程中读取了virtio-blk设备的配置空间的设备信息：

```rust
// virtio_drivers/src/blk.rs
//virtio_blk驱动初始化：调用header.begin_init方法
impl<H: Hal> VirtIOBlk<'_, H> {
   /// Create a new VirtIO-Blk driver.
   pub fn new(header: &'static mut VirtIOHeader) -> Result<Self> {
      header.begin_init(|features| {
            ...
            (features & supported_features).bits()
      });
      //读取virtio_blk设备的配置空间
      let config = unsafe { &mut *(header.config_space() ...) };
      //建立1个虚拟队列
      let queue = VirtQueue::new(header, 0, 16)?;
      //结束设备初始化
      header.finish_init();
      ...
   }
// virtio_drivers/src/header.rs
// virtio设备初始化的第1~4步骤
impl VirtIOHeader {
   pub fn begin_init(&mut self, negotiate_features: impl FnOnce(u64) -> u64) {
      self.status.write(DeviceStatus::ACKNOWLEDGE);
      self.status.write(DeviceStatus::DRIVER);
      let features = self.read_device_features();
      self.write_driver_features(negotiate_features(features));
      self.status.write(DeviceStatus::FEATURES_OK);
      self.guest_page_size.write(PAGE_SIZE as u32);
   }

   // virtio设备初始化的第5步骤
   pub fn finish_init(&mut self) {
      self.status.write(DeviceStatus::DRIVER_OK);
   }

//...
capacity: Volatile<u64> = 32  //32个扇区，即16KB
blk_size: Volatile<u32> = 512 //扇区大小为512字节
```

为何能看到扇区大小为 `512` 字节，容量为 `16KB` 大小的virtio-blk设备？这当然是我们让 Qemu 模拟器建立的一个虚拟硬盘。下面的命令可以看到**虚拟硬盘创建和识别过程：**

```shell
# 在virtio-drivers仓库的example/riscv目录下执行如下命令
make run
# 可以看到与虚拟硬盘创建相关的具体命令
## 通过 dd 工具创建了扇区大小为 ``512`` 字节，容量为 ``16KB`` 大小的硬盘镜像（disk img）
dd if=/dev/zero of=target/riscv64imac-unknown-none-elf/release/img bs=512 count=32
   记录了32+0 的读入
   记录了32+0 的写出
   16384字节（16 kB，16 KiB）已复制，0.000439258 s，37.3 MB/s
## 通过 qemu-system-riscv64 命令启动 Qemu 模拟器，创建 virtio-blk 设备
qemu-system-riscv64 \
     -drive file=target/riscv64imac-unknown-none-elf/release/img,if=none,format=raw,id=x0 \
     -device virtio-blk-device,drive=x0 ...
## 可以看到设备驱动查找到的virtio-blk设备色信息
...
[ INFO] Detected virtio MMIO device with vendor id 0x554D4551, device type Block, version Modern
[ INFO] device features: SEG_MAX | GEOMETRY | BLK_SIZE | FLUSH | TOPOLOGY | CONFIG_WCE | DISCARD | WRITE_ZEROES | RING_INDIRECT_DESC | RING_EVENT_IDX | VERSION_1
[ INFO] config: 0x10008100
[ INFO] found a block device of size 16KB
...
```

---

virtio-blk设备驱动程序了解了virtio-blk设备的扇区个数，扇区大小和总体容量后，还需调用 `` VirtQueue::new`` 成员函数来创建虚拟队列 `VirtQueue` 数据结构的实例，这样才能进行后续的磁盘读写操作。这个函数主要完成的事情是分配虚拟队列的内存空间，并进行初始化：

- 设定 `queue_size` （即虚拟队列的描述符条目数）为16；
- 计算满足 `queue_size` 的描述符表 `desc` ，可用环 `avail` 和已用环 `used` 所需的物理空间的大小 – `size` ；
- 基于上面计算的 `size` 分配物理空间； //VirtQueue.new()
- `VirtIOHeader.queue_set` 函数把虚拟队列的相关信息（内存地址等）写到virtio-blk设备的MMIO寄存器中；
- 初始化VirtQueue实例中各个成员变量（主要是 `dma` ， `desc` ，`avail` ，`used` ）的值。

做完这一步后，virtio-blk设备和设备驱动之间的虚拟队列接口就打通了，可以进行I/O数据读写了。下面简单代码完成了**对虚拟硬盘的读写操作和读写正确性检查：**

```rust
// virtio-drivers/examples/riscv/src/main.rs
fn virtio_blk(header: &'static mut VirtIOHeader) { {
   // 创建blk结构
   let mut blk = VirtIOBlk::<HalImpl, T>::new(header).expect("failed to create blk driver");
   // 读写缓冲区
   let mut input = vec![0xffu8; 512];
   let mut output = vec![0; 512];
   ...
   // 把input数组内容写入virtio-blk设备
   blk.write_block(i, &input).expect("failed to write");
   // 从virtio-blk设备读取内容到output数组
   blk.read_block(i, &mut output).expect("failed to read");
   // 检查virtio-blk设备读写的正确性
   assert_eq!(input, output);
//...
```

### rCore对接virtio-blk设备初始化

但 virtio_drivers 模块还没有与操作系统内核进行对接。我们还需在操作系统中封装 virtio-blk 设备，让操作系统内核能够识别并使用 virtio-blk 设备。首先分析一下操作系统需要建立的**表示 virtio_blk 设备的全局变量 `BLOCK_DEVICE` ：**

```rust
// os/src/drivers/block/virtio_blk.rs
pub struct VirtIOBlock {
   virtio_blk: UPIntrFreeCell<VirtIOBlk<'static, VirtioHal>>,
   condvars: BTreeMap<u16, Condvar>,
}
// os/easy-fs/src/block_dev.rs
pub trait BlockDevice: Send + Sync + Any {
   fn read_block(&self, block_id: usize, buf: &mut [u8]);
   fn write_block(&self, block_id: usize, buf: &[u8]);
   fn handle_irq(&self);
}
// os/src/boards/qemu.rs
pub type BlockDeviceImpl = crate::drivers::block::VirtIOBlock;
// os/src/drivers/block/mod.rs
lazy_static! {
   pub static ref BLOCK_DEVICE: Arc<dyn BlockDevice> = Arc::new(BlockDeviceImpl::new());
}
```

从上面的代码可以看到，操作系统中表示virtio_blk设备的全局变量 `BLOCK_DEVICE` 的类型是 `VirtIOBlock` ，封装了来自 virtio_drivers 模块的 `VirtIOBlk` 类型。这样，操作系统内核就可以通过 `BLOCK_DEVICE` 全局变量来访问virtio_blk设备了。

而 `VirtIOBlock` 中的 `condvars: BTreeMap<u16, Condvar>` 条件变量结构，是用于进程在等待 I/O 读或写操作完成前，通过条件变量让进程处于挂起状态。当 virtio_blk 设备完成 I/O 操作后，会通过中断唤醒等待的进程。

而操作系统对 virtio_blk 设备的初始化除了封装 `VirtIOBlk` 类型并调用 `VirtIOBlk::<VirtioHal>::new()` 外，还需要初始化 `condvars` 条件变量结构，而**每个条件变量对应着一个虚拟队列条目的编号，这意味着每次 I/O 请求都绑定了一个条件变量，让发出请求的线程/进程可以被挂起。**代码如下：

```rust
impl VirtIOBlock {
   pub fn new() -> Self {
      let virtio_blk = unsafe {
            UPIntrFreeCell::new(
               VirtIOBlk::<VirtioHal>::new(&mut *(VIRTIO0 as *mut VirtIOHeader)).unwrap(),
            )
      };
      let mut condvars = BTreeMap::new();
      let channels = virtio_blk.exclusive_access().virt_queue_size();
      for i in 0..channels {
            let condvar = Condvar::new();
            condvars.insert(i, condvar);
      }
      Self {
            virtio_blk,
            condvars,
      }
   }
}
```

* 在上述初始化代码中，我们先看到 `VIRTIO0` ，这是 Qemu 模拟的virtio_blk设备中I/O寄存器的物理内存地址， `VirtIOBlk` 需要这个地址来对 `VirtIOHeader` 数据结构所表示的virtio-blk I/O控制寄存器进行读写操作，从而完成对某个具体的virtio-blk设备的初始化过程。
* 而且我们还看到了 `VirtioHal` 结构，它实现 virtio-drivers 模块定义 `Hal` trait约定的方法 ，提供DMA内存分配和虚实地址映射操作，从而让virtio-drivers 模块中 `VirtIOBlk` 类型能够得到操作系统的服务。

```rust
// os/src/drivers/bus/virtio.rs
impl Hal for VirtioHal {
   fn dma_alloc(pages: usize) -> usize {
      //分配页帧 page-frames
      let pa: PhysAddr = ppn_base.into();
      pa.0
   }

   fn dma_dealloc(pa: usize, pages: usize) -> i32 {
      //释放页帧 page-frames
      0
   }

   fn phys_to_virt(addr: usize) -> usize {
      addr
   }

   fn virt_to_phys(vaddr: usize) -> usize {
      //把虚地址转为物理地址
   }
}
```

## 1.3 virtio设备的I/O操作

### virtio-drivers中的I/O处理

操作系统的 virtio-blk 驱动的主要功能是：**给操作系统中的文件系统内核模块提供读写磁盘块的服务，并对进程管理有一定的影响，但不用直接给应用程序提供服务。**在操作系统与 `virtio-drivers crate` 中 virtio-blk 裸机驱动对接的过程中，需要注意的关键问题是**操作系统的 virtio-blk 驱动如何封装 virtio-blk 裸机驱动的基本功能，**完成如下服务：

1. **读磁盘块，**挂起发起请求的进程/线程;
2. **写磁盘块，**挂起发起请求的进程/线程；
3. 对 virtio-blk 设备发出的**中断进行处理，唤醒**相关等待的进程/线程；

---

virtio-blk 驱动程序发起的 I/O 请求包含操作类型 (读或写)、起始扇区 (块设备的最小访问单位的一个扇区的长度512字节)、内存地址、访问长度；请求处理完成后返回的 I/O 响应仅包含结果状态 (成功或失败，读操作请求的读出扇区内容)。

系统产生的一个 I/O 请求在内存中的数据结构分为三个部分（这些信息分别放在三个buffer，所以需要三个描述符）：

* Header（请求头部，包含操作类型和起始扇区）；
* Data（数据区，包含地址和长度）；
* Status（结果状态）；

virtio-blk 设备使用 `VirtQueue` 数据结构来表示虚拟队列进行数据传输，此数据结构主要由三段连续内存组成：描述符表 `Descriptor[]` 、环形队列结构的 `AvailRing` 和 `UsedRing` 。驱动程序和 virtio-blk 设备都能访问到此数据结构。

---

> ***描述符表***

描述符表由固定长度 (16字节) 的描述符 `Descriptor` 组成，其个数等于环形队列长度，其中每个 `Descriptor` 的结构为：

```rust
struct Descriptor {
   addr: Volatile<u64>,
   len: Volatile<u32>,
   flags: Volatile<DescFlags>,
   next: Volatile<u16>,
}
```

`Descriptor` 包含四个域：

* `addr` 代表某段内存的起始地址，长度为 8 个字节；
* `len` 代表某段内存的长度，本身占用 4 个字节 (因此代表的内存段最大为 4 GB)；
* `flags` 代表内存段读写属性等，长度为 2 个字节；
* `next` 代表下一个内存段对应的 Descpriptor 在描述符表中的索引，因此通过 `next` 字段可以将一个请求对应的多个内存段连接成链表；

---

> ***可用环***

可用环 `AvailRing` 的结构为：

```rust
struct AvailRing {
   flags: Volatile<u16>,
   /// A driver MUST NOT decrement the idx.
   idx: Volatile<u16>,
   ring: [Volatile<u16>; 32], // actual size: queue_size
   used_event: Volatile<u16>, // unused
}
```

可用环由头部的 `flags` 和 `idx` 域及 `ring` 数组组成：

*  `flags` 与通知机制相关； 
* `idx` 代表最新放入 IO 请求的编号，从零开始单调递增，将其对队列长度取余即可得该 I/O 请求在可用环数组中的索引；
* `ring` 可用环数组元素用来存放 I/O 请求占用的首个描述符在描述符表中的索引，数组长度等于可用环的长度 (不开启 `event_idx` 特性)；

---

> ***已用环***

已用环 `UsedRing` 的结构为：

```rust
struct UsedRing {
   flags: Volatile<u16>,
   idx: Volatile<u16>,
   ring: [UsedElem; 32],       // actual size: queue_size
   avail_event: Volatile<u16>, // unused
}
```

已用环由头部的 `flags` 和 `idx` 域及 `ring` 数组组成： 

* `flags` 与通知机制相关； 
* `idx` 代表最新放入 I/O 响应的编号，从零开始单调递增，将其对队列长度取余即可得该 I/O 响应在已用环数组中的索引；
* `ring ` 已用环数组元素主要用来存放 I/O 响应占用的首个描述符在描述符表中的索引， 数组长度等于已用环的长度 (不开启event_idx特性)；

---

针对用户进程发出的I/O请求，经过系统调用，文件系统等一系列处理后，最终会形成对 virtio-blk 驱动程序的调用。对于写操作，具体实现如下：

```rust
//virtio-drivers/src/blk.rs
 pub fn write_block(&mut self, block_id: usize, buf: &[u8]) -> Result {
     assert_eq!(buf.len(), BLK_SIZE);
     let req = BlkReq {
         type_: ReqType::Out,
         reserved: 0,
         sector: block_id as u64,
     };
     let mut resp = BlkResp::default();
     self.queue.add(&[req.as_buf(), buf], &[resp.as_buf_mut()])?;
     self.header.notify(0);
     while !self.queue.can_pop() {
         spin_loop();
     }
     self.queue.pop_used()?;
     match resp.status {
         RespStatus::Ok => Ok(()),
         _ => Err(Error::IoError),
     }
 }
```

基本流程如下：

1. 一个完整的 virtio-blk 的 I/O 写请求由三部分组成，这三部分需要三个描述符来表示，包括：
   * 表示I/O写请求信息的结构 `BlkReq` ；
   * 要传输的数据块 `buf`；
   * 表示设备响应信息的结构 `BlkResp` ；
2. （驱动程序处理）接着调用 `VirtQueue.add` 函数，从描述符表中申请三个空闲描述符，每项指向一个内存块，填写上述三部分的信息，并将三个描述符连接成一个描述符链表；
3. （驱动程序处理）接着调用 `VirtQueue.notify` 函数，写MMIO模式的 `queue_notify` 寄存器，即向 virtio-blk 设备发出通知；
4. （设备处理）virtio-blk 设备收到通知后，通过比较 `last_avail` (初始为0) 和 `AvailRing` 中的 `idx` 判断是否有新的请求待处理 (如果 `last_vail` 小于 `AvailRing` 中的 `idx` ，则表示有新请求)。如果有，则 `last_avail` 加1，并以 `last_avail` 为索引从描述符表中找到这个 I/O 请求对应的描述符链来获知完整的请求信息，并完成存储块的 I/O 写操作；
5. （设备处理）设备完成 I/O 写操作后 (包括更新包含 `BlkResp` 的 Descriptor)，将已完成 I/O 的描述符放入 `UsedRing` 对应的ring项中，并更新 `idx`，代表放入一个响应；如果设置了中断机制，还会产生中断来通知操作系统响应中断；
6. （驱动程序处理）驱动程序可用轮询机制查看设备是否有响应（持续调用 `VirtQueue.can_pop` 函数），通过比较内部的 `VirtQueue.last_used_idx` 和 `VirtQueue.used.idx` 判断是否有新的响应。如果有，则取出响应(并更新 `last_used_idx` )，将完成响应对应的三项Descriptor回收，最后将结果返回给用户进程。当然，也可通过中断机制来响应。

> I/O 读请求的处理过程与I/O写请求的处理过程几乎一样，仅仅是 `BlkReq` 的内容不同，写操作中的 `req.type_` 是 `ReqType::Out`，而读操作中的 `req.type_` 是 `ReqType::In` 。具体可以看看 `virtio-drivers/src/blk.rs` 文件中的 `VirtIOBlk.read_block` 函数的实现。

这种基于轮询的 I/O 访问方式效率比较差，为此，我们需要实现基于中断的 I/O 访问方式。为此提供了一个支持中断的 `write_block_nb` 方法：

```rust
pub unsafe fn write_block_nb(
     &mut self,
     block_id: usize,
     buf: &[u8],
     resp: &mut BlkResp,
 ) -> Result<u16> {
     assert_eq!(buf.len(), BLK_SIZE);
     let req = BlkReq {
         type_: ReqType::Out,
         reserved: 0,
         sector: block_id as u64,
     };
     let token = self.queue.add(&[req.as_buf(), buf], &[resp.as_buf_mut()])?;
     self.header.notify(0);
     Ok(token)
}

// Acknowledge interrupt.
pub fn ack_interrupt(&mut self) -> bool {
     self.header.ack_interrupt()
}
```

与不支持中断的 `write_block` 函数比起来， `write_block_nb` 函数更简单了，在发出I/O请求后，就直接返回了。 `read_block_nb` 函数的处理流程与此一致。

> 注意，响应中断的 `ack_interrupt` 函数只是完成了非常基本的 virtio 设备的中断响应操作。在 virtio-drivers 中实现的virtio设备驱动是看不到进程、条件变量等操作系统的各种关键要素，只有与操作系统内核对接，才能完整实现基于中断的I/O访问方式。

### rCore对接virtio-blk设备I/O处理

操作系统中的文件系统模块与操作系统中的块设备驱动程序 `VirtIOBlock` 直接交互，而操作系统中的块设备驱动程序 `VirtIOBlock` 封装了virtio-drivers中实现的 virtio_blk 设备驱动。在文件系统的介绍中，我们并没有深入分析virtio_blk设备。这里我们将介绍操作系统对接virtio_blk设备驱动并完成基于中断机制的 I/O 处理过程。

接下来需要扩展文件系统对块设备驱动的 I/O 访问要求，这体现在 `BlockDevice` trait 的新定义中增加了 `handle_irq` 方法，而操作系统的virtio_blk 设备驱动程序中的 `VirtIOBlock` 实现了这个方法，并且实现了既支持轮询方式，也支持中断方式的块读写操作。

```rust
// easy-fs/src/block_dev.rs
pub trait BlockDevice: Send + Sync + Any {
   fn read_block(&self, block_id: usize, buf: &mut [u8]);
   fn write_block(&self, block_id: usize, buf: &[u8]);
   // 更新的部分：增加对块设备中断的处理
   fn handle_irq(&self);
}
// os/src/drivers/block/virtio_blk.rs
impl BlockDevice for VirtIOBlock {
   fn handle_irq(&self) {
      self.virtio_blk.exclusive_session(|blk| {
            while let Ok(token) = blk.pop_used() {
                  // 唤醒等待该块设备I/O完成的线程/进程
               self.condvars.get(&token).unwrap().signal();
            }
      });
   }

   fn read_block(&self, block_id: usize, buf: &mut [u8]) {
      // 获取轮询或中断的配置标记
      let nb = *DEV_NON_BLOCKING_ACCESS.exclusive_access();
      if nb { // 如果是中断方式
            let mut resp = BlkResp::default();
            let task_cx_ptr = self.virtio_blk.exclusive_session(|blk| {
               // 基于中断方式的块读请求
               let token = unsafe { blk.read_block_nb(block_id, buf, &mut resp).unwrap() };
               // 将当前线程/进程加入条件变量的等待队列
               self.condvars.get(&token).unwrap().wait_no_sched()
            });
            // 切换线程/进程
            schedule(task_cx_ptr);
            assert_eq!(
               resp.status(),
               RespStatus::Ok,
               "Error when reading VirtIOBlk"
            );
      } else { // 如果是轮询方式，则进行轮询式的块读请求
            self.virtio_blk
               .exclusive_access()
               .read_block(block_id, buf)
               .expect("Error when reading VirtIOBlk");
      }
   }
```

`write_block` 写操作与 `read_block` 读操作的处理过程一致，这里不再赘述。

---

然后需要对操作系统整体的中断处理过程进行调整，以**支持对基于中断方式的块读写操作：**

```rust
// os/src/trap/mode.rs
//在用户态接收到外设中断
pub fn trap_handler() -> ! {
   //...
   crate::board::irq_handler();
//在内核态接收到外设中断
pub fn trap_from_kernel(_trap_cx: &TrapContext) {
   ...
   crate::board::irq_handler();
// os/src/boards/qemu.rs
pub fn irq_handler() {
   let mut plic = unsafe { PLIC::new(VIRT_PLIC) };
   // 获得外设中断号
   let intr_src_id = plic.claim(0, IntrTargetPriority::Supervisor);
   match intr_src_id {
      ...
      //处理virtio_blk设备产生的中断
      8 => BLOCK_DEVICE.handle_irq(),
   }
   // 完成中断响应
   plic.complete(0, IntrTargetPriority::Supervisor, intr_src_id);
}
```

`BLOCK_DEVICE.handle_irq()` 执行的就是 `VirtIOBlock` 实现的中断处理方法 `handle_irq()` ，从而让等待在块读写的进程/线程得以继续执行。

> 有了基于中断方式的块读写操作后，当某个线程/进程由于块读写操作无法继续执行时，操作系统可以切换到其它处于就绪态的线程/进程执行，从而让计算机系统的整体执行效率得到提升。



# 2 virtio_gpu设备驱动

本节主要介绍了：

* 与操作系统无关的基本 `virtio_gpu` 设备驱动程序的设计与实现；
* 如何在操作系统中封装 `virtio_gpu` 设备驱动程序，实现对丰富多彩的 GUI App 的支持；

让操作系统能够显示图形是一个有趣的目标。这可以通过在QEMU或带显示屏的开发板上写显示驱动程序来完成。这里我们主要介绍**如何驱动基于QEMU的virtio-gpu虚拟显示设备。**

> 大家不用担心这个驱动实现很困难，其实它主要完成的事情就是对显示内存进行写操作而已。我们看到的图形显示屏幕其实是由一个一个的像素点来组成的，显示驱动程序的主要目标就是把每个像素点用内存单元来表示，并把代表所有这些像素点的内存区域（也称显示内存，显存， frame buffer）“通知” 显示I/O控制器（也称图形适配器，`graphics adapter`），然后显示I/O控制器会根据内存内容渲染到图形显示屏上。

这里我们以Rust语言为例，给出 virtio-gpu 设备驱动程序的设计与实现。主要包括如下内容：

- `virtio-gpu` 设备的关键数据结构
- 初始化 `virtio-gpu` 设备
- 操作系统对接 `virtio-gpu` 设备初始化
- `virtio-gpu` 设备的I/O操作
- 操作系统对接 `virtio-gpu` 设备I/O操作

## 2.1 virtio-gpu设备的关键数据结构

```rust
// virtio-drivers/src/gpu.rs
 pub struct VirtIOGpu<'a, H: Hal> {
     header: &'static mut VirtIOHeader,
     /// 显示区域的分辨率
     rect: Rect,
     /// 显示内存frame buffer
     frame_buffer_dma: Option<DMA<H>>,
     /// 光标图像内存cursor image buffer.
     cursor_buffer_dma: Option<DMA<H>>,
     /// Queue for sending control commands.
     control_queue: VirtQueue<'a, H>,
     /// Queue for sending cursor commands.
     cursor_queue: VirtQueue<'a, H>,
     /// Queue buffer
     queue_buf_dma: DMA<H>,
     /// Send buffer for queue.
     queue_buf_send: &'a mut [u8],
     /// Recv buffer for queue.
     queue_buf_recv: &'a mut [u8],
 }
```

解释一下 `VirtIOGpu` 的各个字段：

* `header` 成员对应的 `VirtIOHeader` 数据结构是virtio设备的共有属性，包括版本号、设备id、设备特征等信息，其内存布局和成员变量的含义与本章前述的 [virt-mmio设备的寄存器内存布局](https://rcore-os.cn/rCore-Tutorial-Book-v3/chapter9/2device-driver-2.html#term-virtio-mmio-regs) 是一致的。而 [VirtQueue数据结构的内存布局](https://rcore-os.cn/rCore-Tutorial-Book-v3/chapter9/2device-driver-3.html#term-virtqueue-struct) 和 [virtqueue的含义](https://rcore-os.cn/rCore-Tutorial-Book-v3/chapter9/2device-driver-2.html#term-virtqueue) 与本章前述内容一致。与 [具体操作系统相关的服务函数接口Hal](https://rcore-os.cn/rCore-Tutorial-Book-v3/chapter9/2device-driver-3.html#term-virtio-hal) 在上一节已经介绍过，这里不再赘述。
* 显示内存区域 `frame_buffer_dma` 是一块要由操作系统或运行时分配的显示内存，当把表示像素点的值就写入这个区域后，virtio-gpu设备会在Qemu虚拟的显示器上显示出图形。
* 光标图像内存区域 `cursor_buffer_dma` 用于存放光标图像的数据，当光标图像数据更新后，virtio-gpu设备会在Qemu虚拟的显示器上显示出光标图像。
* 以上两块区域与 `queue_buf_dma` 区域都是用于与I/O设备进行数据传输的 [DMA内存](https://rcore-os.cn/rCore-Tutorial-Book-v3/chapter9/1io-interface.html#term-dma-tech)，都由操作系统进行分配等管理。所以在 `virtio_drivers` 中建立了对应的 `DMA` 结构，用于操作系统管理这些 DMA 内存。

---

DMA 相关代码如下：

```rust
// virtio-drivers/src/gpu.rs
 pub struct DMA<H: Hal> {
     paddr: usize,  // DMA内存起始物理地址
     pages: usize,  // DMA内存所占物理页数量
     //...
 }
 impl<H: Hal> DMA<H> {
     pub fn new(pages: usize) -> Result<Self> {
         //操作系统分配 pages*页大小的DMA内存
         let paddr = H::dma_alloc(pages);
         //...
     }
     // DMA内存的物理地址
     pub fn paddr(&self) -> usize {
         self.paddr
     }
     // DMA内存的虚拟地址
     pub fn vaddr(&self) -> usize {
         H::phys_to_virt(self.paddr)
     }
     // DMA内存的物理页帧号
     pub fn pfn(&self) -> u32 {
         (self.paddr >> 12) as u32
     }
     // 把DMA内存转为便于Rust处理的可变一维数组切片
     pub unsafe fn as_buf(&self) -> &'static mut [u8] {
         core::slice::from_raw_parts_mut(self.vaddr() as _, PAGE_SIZE * self.pages as usize)
     //...
 impl<H: Hal> Drop for DMA<H> {
     // 操作系统释放DMA内存
     fn drop(&mut self) {
         let err = H::dma_dealloc(self.paddr as usize, self.pages as usize);
         //...
```

---

virtio-gpu 驱动程序与 virtio-gpu 设备之间**通过两个 virtqueue 来进行交互访问：**

* `control_queue` 用于驱动程序发送显示相关控制命令（如设置显示内存的起始地址等）给 virtio-gpu 设备；
*  `cursor_queue` 用于驱动程序给给 virtio-gpu 设备发送显示鼠标更新的相关控制命令；

对于 DMA 内存：

* `queue_buf_dma` 是存放控制命令和返回结果的内存；
*  `queue_buf_send` 和 `queue_buf_recv` 是 `queue_buf_dma` DMA内存的可变一维数组切片形式，分别用于虚拟队列的接收与发送。

## 2.2 virtio-driver中的OS无关操作

### 初始化virtio-gpu设备

在 `virtio-drivers` crate 的 `examples/riscv/src/main.rs` 文件中的 `virtio_probe` 函数识别出virtio-gpu设备后，会调用 `virtio_gpu(header)` 函数来完成对virtio-gpu设备的初始化过程。virtio-gpu设备初始化的工作主要是：**查询显示设备的信息（如分辨率等），并将该信息用于初始显示扫描（scanout）设置。**

下面的命令可以看到虚拟 GPU 的创建和识别过程：

```shell
# 在virtio-drivers仓库的example/riscv目录下执行如下命令
make run
## 通过 qemu-system-riscv64 命令启动 Qemu 模拟器，创建 virtio-gpu 设备
qemu-system-riscv64 \
     -device virtio-gpu-device ...
## 可以看到设备驱动查找到的virtio-gpu设备色信息
...
[ INFO] Detected virtio MMIO device with vendor id 0x554D4551, device type GPU, version Modern
[ INFO] Device features EDID | RING_INDIRECT_DESC | RING_EVENT_IDX | VERSION_1
[ INFO] events_read: 0x0, num_scanouts: 0x1
[ INFO] GPU resolution is 1280x800
[ INFO] => RespDisplayInfo { header: CtrlHeader { hdr_type: OkDisplayInfo, flags: 0, fence_id: 0
```

并看到Qemu输出的图形显示：

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202312011107824.png" alt="image-20231201110655079" style="zoom:25%;" />

接下来我们看看**virtio-gpu设备初始化**的具体过程如下：

```rust
 // virtio-drivers/examples/riscv/src/main.rs
 fn virtio_gpu(header: &'static mut VirtIOHeader) {
     let mut gpu = VirtIOGpu::<HalImpl>::new(header).expect("failed to create gpu driver");
     let (width, height) = gpu.resolution().expect("failed to get resolution");
     info!("GPU resolution is {}x{}", width, height);
     let fb = gpu.setup_framebuffer().expect("failed to get fb");
     //...
```

在 `virtio_gpu` 函数调用创建了 `VirtIOGpu::<HalImpl>::new(header)` 函数：

1. 获得关于virtio-gpu设备的重要信息，显示分辨率 `1280x800` ；
2. 建立virtio虚拟队列，并基于这些信息来创建表示virtio-gpu的 `gpu` 结构；
3. 然后会进一步调用 `gpu.setup_framebuffer()` 函数来建立和配置显示内存缓冲区，打通设备驱动与virtio-gpu设备间的显示数据传输通道。

---

#### VirtIOGpu结构初始化

`VirtIOGpu::<HalImpl>::new(header)` 函数主要完成了virtio-gpu设备的初始化工作：

```rust
// virtio-drivers/src/gpu.rs
impl VirtIOGpu<'_, H> {
pub fn new(header: &'static mut VirtIOHeader) -> Result<Self> {
     header.begin_init(|features| {
         let features = Features::from_bits_truncate(features);
         let supported_features = Features::empty();
         (features & supported_features).bits()
     });

     // read configuration space
     let config = unsafe { &mut *(header.config_space() as *mut Config) };

     let control_queue = VirtQueue::new(header, QUEUE_TRANSMIT, 2)?;
     let cursor_queue = VirtQueue::new(header, QUEUE_CURSOR, 2)?;

     let queue_buf_dma = DMA::new(2)?;
     let queue_buf_send = unsafe { &mut queue_buf_dma.as_buf()[..PAGE_SIZE] };
     let queue_buf_recv = unsafe { &mut queue_buf_dma.as_buf()[PAGE_SIZE..] };

     header.finish_init();

     Ok(VirtIOGpu {
         header,
         frame_buffer_dma: None,
         rect: Rect::default(),
         control_queue,
         cursor_queue,
         queue_buf_dma,
         queue_buf_send,
         queue_buf_recv,
     })
 }
```

* 首先是 `header.begin_init` 函数完成了对virtio设备的共性初始化的常规步骤的前六步；
* 第七步在这里被忽略；
* 第八步读取virtio-gpu设备的配置空间（config space）信息；
* 紧接着是创建两个虚拟队列：控制命令队列、光标管理队列。并为控制命令队列分配两个 `page` （8KB）的内存空间用于放置虚拟队列中的控制命令和返回结果；
* 最后的第九步，调用 `header.finish_init` 函数，将virtio-gpu设备设置为活跃可用状态；

---

#### 设置显存区域frame_buffer

虽然virtio-gpu设备在驱动中的数据结构初始化完毕，但它目前还不能进行显示。为了能够进行正常的显示，我们还需建立显存区域 `frame buffer`，并绑定在virtio-gpu设备上。这主要是通过 `VirtIOGpu.setup_framebuffer` 函数来完成的。代码如下：

```rust
// virtio-drivers/src/gpu.rs
pub fn setup_framebuffer(&mut self) -> Result<&mut [u8]> {
     // get display info
     let display_info: RespDisplayInfo =
         self.request(CtrlHeader::with_type(Command::GetDisplayInfo))?;
     display_info.header.check_type(Command::OkDisplayInfo)?;
     self.rect = display_info.rect;

     // create resource 2d
     let rsp: CtrlHeader = self.request(ResourceCreate2D {
         header: CtrlHeader::with_type(Command::ResourceCreate2d),
         resource_id: RESOURCE_ID,
         format: Format::B8G8R8A8UNORM,
         width: display_info.rect.width,
         height: display_info.rect.height,
     })?;
     rsp.check_type(Command::OkNodata)?;

     // alloc continuous pages for the frame buffer
     let size = display_info.rect.width * display_info.rect.height * 4;
     let frame_buffer_dma = DMA::new(pages(size as usize))?;

     // resource_attach_backing
     let rsp: CtrlHeader = self.request(ResourceAttachBacking {
         header: CtrlHeader::with_type(Command::ResourceAttachBacking),
         resource_id: RESOURCE_ID,
         nr_entries: 1,
         addr: frame_buffer_dma.paddr() as u64,
         length: size,
         padding: 0,
     })?;
     rsp.check_type(Command::OkNodata)?;

     // map frame buffer to screen
     let rsp: CtrlHeader = self.request(SetScanout {
         header: CtrlHeader::with_type(Command::SetScanout),
         rect: display_info.rect,
         scanout_id: 0,
         resource_id: RESOURCE_ID,
     })?;
     rsp.check_type(Command::OkNodata)?;

     let buf = unsafe { frame_buffer_dma.as_buf() };
     self.frame_buffer_dma = Some(frame_buffer_dma);
     Ok(buf)
 }
```

上面的函数主要完成的工作有如下几个步骤，其实就是**驱动程序给virtio-gpu设备发控制命令，建立好显存区域：**

1. 发出 `GetDisplayInfo` 命令，获得virtio-gpu设备的显示分辨率;
2. 发出 `ResourceCreate2D` 命令，让设备以分辨率大小（ `width * height` ），像素信息（ `Red/Green/Blue/Alpha` 各占1字节大小，即一个像素占4字节），来配置设备显示资源；
3. 分配 `width * height * 4` 字节的连续物理内存空间作为显存， 发出 `ResourceAttachBacking` 命令，让设备把显存附加到设备显示资源上；
4. 发出 `SetScanout` 命令，把设备显示资源链接到显示扫描输出上，这样才能让显存的像素信息显示出来；

---

到这一步，才算是把virtio-gpu设备初始化完成了。做完这一步后，virtio-gpu设备和设备驱动之间的虚拟队列接口就打通了，显示缓冲区也建立好了，就可以进行显存数据读写了。

### virtio-gpu设备的I/O操作

对初始化好的virtio-gpu设备进行图形显示其实很简单，主要就是两个步骤：

1. 把要显示的像素数据写入到显存中；
2. 发出刷新命令，让virtio-gpu在Qemu模拟的显示区上显示图形。

下面简单代码完成了对虚拟GPU的图形显示：

```rust
// virtio-drivers/src/gpu.rs
fn virtio_gpu(header: &'static mut VirtIOHeader) {
     ...
     //把像素数据写入显存
     for y in 0..height {    //height=800
         for x in 0..width { //width=1280
             let idx = (y * width + x) * 4;
             fb[idx] = x as u8;
             fb[idx + 1] = y as u8;
             fb[idx + 2] = (x + y) as u8;
         }
     }
     // 发出刷新命令
     gpu.flush().expect("failed to flush");
```

> 这里需要注意的是对virtio-gpu进行刷新操作比较耗时，所以我们尽量先把显示的图形像素值都写入显存中，再发出刷新命令，减少刷新命令的执行次数。

## 2.3 rCore对接virtio-gpu设备

### rCore对接virtio-gpu设备初始化

虽然virtio-gpu设备驱动程序已经完成了，但是还需要操作系统对接virtio-gpu设备，才能真正的把virtio-gpu设备驱动程序和操作系统对接起来。virtio-gpu设备在rCore中的初始化过程主要包括：

1. 调用 virtio-drivers/gpu.rs 中提供 `VirtIOGpu::new()` 方法，初始化virtio_gpu设备；
2. 建立显存缓冲区的可变一维数组引用，便于后续写显存来显示图形；
3. 建立显示窗口中的光标图形；
4. 设定表示 virtio_gpu 设备的全局变量。

---

#### GpuDevice实现

先看看操作系统需要建立的表示 virtio_gpu 设备的全局变量 `GPU_DEVICE` ：

```rust
 // os/src/drivers/gpu/mod.rs
 pub trait GpuDevice: Send + Sync + Any {
     fn update_cursor(&self); //更新光标，目前暂时没用
     fn get_framebuffer(&self) -> &mut [u8];
     fn flush(&self);
 }
 pub struct VirtIOGpuWrapper {
     gpu: UPIntrFreeCell<VirtIOGpu<'static, VirtioHal>>,
     fb: &'static [u8],
 }
 lazy_static::lazy_static!(
     pub static ref GPU_DEVICE: Arc<dyn GpuDevice> = Arc::new(VirtIOGpuWrapper::new());
 );
```

从上面的代码可以看到，操作系统中表示表示 virtio_gpu 设备的全局变量 `GPU_DEVICE` 的类型是 `VirtIOGpuWrapper` ,封装了来自 virtio_derivers 模块的 `VirtIOGpu` 类型，以及一维字节数组引用表示的显存缓冲区 `fb` 。这样，操作系统内核就可以通过 `GPU_DEVICE` 全局变量来访问 gpu_blk 设备了。而操作系统对 virtio_blk 设备的初始化就是调用 `VirtIOGpuWrapper::<VirtioHal>::new()` 。

当用户态应用程序要进行图形显示时，至少需要得到操作系统的两个基本图形显示服务：

* 其一是得到**显存在用户态可访问的的内存地址，**这样应用程序可以在用户态把表示图形的像素值写入显存中；
* 其二是**给virtio-gpu设备发出 `flush` 刷新指令，**这样virtio-gpu设备能够更新显示器中的图形显示内容。

为此，操作系统在 `VirtIOGpuWrapper` 结构类型中需要实现 `GpuDevice` trait，该 trait 需要实现两个函数来支持应用程序所需要的基本显示服务：

```rust
 impl GpuDevice for VirtIOGpuWrapper {
     // 通知virtio-gpu设备更新图形显示内容
     fn flush(&self) {
         self.gpu.exclusive_access().flush().unwrap();
     }
     // 得到显存的基于内核态虚地址的一维字节数组引用
     fn get_framebuffer(&self) -> &mut [u8] {
         unsafe {
             let ptr = self.fb.as_ptr() as *const _ as *mut u8;
             core::slice::from_raw_parts_mut(ptr, self.fb.len())
         }
     }
 //...
```

---

#### rCore初始化GPU设备

接下来，看一下操作系统对virtio-gpu设备的初始化过程：

```rust
 // os/src/drivers/gpu/mod.rs
 impl VirtIOGpuWrapper {
     pub fn new() -> Self {
         unsafe {
             // 1. 执行virtio-drivers的gpu.rs中virto-gpu基本初始化
             let mut virtio =
                 VirtIOGpu::<VirtioHal>::new(&mut *(VIRTIO7 as *mut VirtIOHeader)).unwrap();
             // 2. 设置virtio-gpu设备的显存，初始化显存的一维字节数组引用
             let fbuffer = virtio.setup_framebuffer().unwrap();
             let len = fbuffer.len();
             let ptr = fbuffer.as_mut_ptr();
             let fb = core::slice::from_raw_parts_mut(ptr, len);
             // 3. 初始化光标图像的像素值
             let bmp = Bmp::<Rgb888>::from_slice(BMP_DATA).unwrap();
             let raw = bmp.as_raw();
             let mut b = Vec::new();
             for i in raw.image_data().chunks(3) {
                 let mut v = i.to_vec();
                 b.append(&mut v);
                 if i == [255, 255, 255] {
                     b.push(0x0)
                 } else {
                     b.push(0xff)
                 }
             }
             // 4. 设置virtio-gpu设备的光标图像
             virtio.setup_cursor(b.as_slice(), 50, 50, 50, 50).unwrap();
             // 5. 返回VirtIOGpuWrapper结构类型
             Self {
                 gpu: UPIntrFreeCell::new(virtio),
                 fb,
             }
     //...
```

在上述初始化过程中，我们先看到 `VIRTIO7` ，这是 Qemu 模拟的 virtio_gpu 设备中I/O寄存器的物理内存地址， `VirtIOGpu` 需要这个地址来对 `VirtIOHeader` 数据结构所表示的virtio-gpu I/O控制寄存器进行读写操作，从而完成对某个具体的virtio-gpu设备的初始化过程。整个初始化过程的步骤如下：

1. 执行virtio-drivers的 `gpu.rs` 中virto-gpu基本初始化
2. 设置virtio-gpu设备的显存，初始化显存的一维字节数组引用
3. （可选）初始化光标图像的像素值
4. （可选）设置virtio-gpu设备的光标图像
5. 返回 `VirtIOGpuWrapper` 结构类型

上述步骤的第一步 [“virto-gpu基本初始化”](https://rcore-os.cn/rCore-Tutorial-Book-v3/chapter9/2device-driver-4.html#term-virtio-driver-gpu-new) 和第二步 [设置显存](https://rcore-os.cn/rCore-Tutorial-Book-v3/chapter9/2device-driver-4.html#term-virtio-driver-gpu-setupfb) 是核心内容，都由 virtio-drivers 中与具体操作系统无关的 virtio-gpu 裸机驱动实现，极大降低本章从操作系统的代码复杂性。

至此，我们已经完成了操作系统对 virtio-gpu 设备的初始化过程，接下来，我们看一下操作系统对 virtio-gpu 设备的I/O处理过程。

### rCore对接virtio-gpu设备I/O处理

操作系统的 virtio-gpu 驱动的主要功能是**给操作系统提供支持，让运行在用户态应用能够显示图形。**为此，应用程序需要做到：

* 知道可读写的显存在哪里，并能把更新的像素值写入显存；
* 另外还需要能够通知 virtio-gpu 设备更新显示内容；

可以看出，这主要与操作系统的进程管理和虚存管理有直接的关系。

---

在操作系统与 `virtio-drivers` crate 中 virtio-gpu 裸机驱动对接的过程中，需要注意的关键问题是操作系统的 virtio-gpu 驱动如何封装 virtio-gpu 裸机驱动的基本功能，完成如下服务：

* 根据 virtio-gpu 裸机驱动提供的显存信息，建立应用程序访问的用户态显存地址空间；
* 实现相关系统调用：
  * 把用户态显存地址空间的基址和范围发给应用程序；
  * 实现系统调用，把更新显存的命令发给virtio-gpu设备。

> 这里我们还是做了一些简化，即应用程序预先知道了 virtio-gpu 的显示分辨率为 `1280x800` ，采用的是 `R/G/B/Alpha` 像素显示，即一个像素点占用 4 个字节。这样整个显存大小为 `1280x800x4=4096000` 字节，即大约 `4000KB`，近 `4MB`。

#### 用户态显存服务的系统调用

我们先看看图形应用程序所需要的**两个系统调用：**

```rust
 // os/src/syscall/mod.rs
 const SYSCALL_FRAMEBUFFER: usize = 2000;
 const SYSCALL_FRAMEBUFFER_FLUSH: usize = 2001;
 // os/src/syscall/gui.rs
 // 显存的用户态起始虚拟地址
 const FB_VADDR: usize = 0x10000000;
 pub fn sys_framebuffer() -> isize {
     // 获得显存的起始物理页帧和结束物理页帧
     let gpu = GPU_DEVICE.clone();
     let fb = gpu.get_framebuffer();
     let len = fb.len();
     let fb_ppn = PhysAddr::from(fb.as_ptr() as usize).floor();
     let fb_end_ppn = PhysAddr::from(fb.as_ptr() as usize + len).ceil();
     // 获取当前进程的地址空间结构 mem_set
     let current_process = current_process();
     let mut inner = current_process.inner_exclusive_access();
     let mem_set = &mut inner.memory_set;
     // 把显存的物理页帧映射到起始地址为FB_VADDR的用户态虚拟地址空间
     mem_set.push_noalloc(
         MapArea::new(
             (FB_VADDR as usize).into(),
             (FB_VADDR + len as usize).into(),
             MapType::Framed,
             MapPermission::R | MapPermission::W | MapPermission::U,
         ),
         PPNRange::new(fb_ppn, fb_end_ppn),
     );
     // 返回起始地址为FB_VADDR
     FB_VADDR as isize
 }
 // 要求virtio-gpu设备刷新图形显示
 pub fn sys_framebuffer_flush() -> isize {
     let gpu = GPU_DEVICE.clone();
     gpu.flush();
     0
 }
```

有了这两个系统调用，就可以很容易建立图形应用程序了。下面这个应用程序，可以在 Qemu 模拟的屏幕上显示一个彩色的矩形。

```rust
// user/src/bin/gui_simple.rs
pub const VIRTGPU_XRES: usize = 1280; // 显示分辨率的宽度
pub const VIRTGPU_YRES: usize = 800;  // 显示分辨率的高度
pub fn main() -> i32 {
     // 访问sys_framebuffer系统调用，获得显存基址
     let fb_ptr = framebuffer() as *mut u8;
     // 把显存转换为一维字节数组
     let fb = unsafe {core::slice::from_raw_parts_mut(fb_ptr as *mut u8, VIRTGPU_XRES*VIRTGPU_YRES*4 as usize)};
     // 更新显存的像素值
     for y in 0..800 {
         for x in 0..1280 {
             let idx = (y * 1280 + x) * 4;
             fb[idx] = x as u8;
             fb[idx + 1] = y as u8;
             fb[idx + 2] = (x + y) as u8;
         }
     }
     // 访问sys_framebuffer_flush系统调用，要求virtio-gpu设备刷新图形显示
     framebuffer_flush();
     0
}
```

---

#### 创建用户态显存虚拟地址空间

到目前为止，看到的操作系统支持工作还是比较简单的，但其实我们还没分析如何给应用程序提供显存虚拟地址空间的。以前章节的操作系统支持应用程序的 [用户态地址空间](https://rcore-os.cn/rCore-Tutorial-Book-v3/chapter4/5kernel-app-spaces.html#term-vm-app-addr-space) ，都是在创建应用程序对应进程的初始化过程中建立，涉及不少工作，具体包括：

- 分配空闲 [物理页帧](https://rcore-os.cn/rCore-Tutorial-Book-v3/chapter4/4sv39-implementation-2.html#term-manage-phys-frame)
- 建立 [进程地址空间(Address Space)](https://rcore-os.cn/rCore-Tutorial-Book-v3/chapter4/5kernel-app-spaces.html#term-vm-memory-set) 中的 [逻辑段（MemArea）](https://rcore-os.cn/rCore-Tutorial-Book-v3/chapter4/5kernel-app-spaces.html#term-vm-map-area)
- 建立映射物理页帧和虚拟页的 [页表](https://rcore-os.cn/rCore-Tutorial-Book-v3/chapter4/4sv39-implementation-2.html#term-create-pagetable)

目前这些工作不能直接支持建立用户态显存地址空间。主要原因在于，用户态显存的物理页帧分配和物理虚地址页表映射，都是由 virtio-gpu 裸机设备驱动程序在设备初始化时完成。在图形应用进程的创建过程中，不需要再分配显存的物理页帧了，只需建立显存的用户态虚拟地址空间。

**为了支持操作系统把用户态显存地址空间的基址发给应用程序，需要对操作系统的虚拟内存管理进行一定的扩展，** 即实现 `sys_framebuffer` 系统调用中访问的 `mem_set.push_noalloc` 新函数和其它相关函数。

```rust
// os/src/mm/memory_set.rs
impl MemorySet {
  pub fn push_noalloc(&mut self, mut map_area: MapArea, ppn_range: PPNRange) {
     map_area.map_noalloc(&mut self.page_table, ppn_range);
     self.areas.push(map_area);
  }
impl MapArea {
  pub fn map_noalloc(&mut self, page_table: &mut PageTable,ppn_range:PPNRange) {
     for (vpn,ppn) in core::iter::zip(self.vpn_range,ppn_range) {
         self.data_frames.insert(vpn, FrameTracker::new_noalloc(ppn));
         let pte_flags = PTEFlags::from_bits(self.map_perm.bits).unwrap();
         page_table.map(vpn, ppn, pte_flags);
     }
  }
// os/src/mm/frame_allocator.rs
pub struct FrameTracker {
  pub ppn: PhysPageNum,
  pub nodrop: bool,
}
impl FrameTracker {
  pub fn new_nowrite(ppn: PhysPageNum) -> Self {
     Self { ppn, nodrop: true }
  }
impl Drop for FrameTracker {
     fn drop(&mut self) {
         if self.nodrop {
             return;
         }
         frame_dealloc(self.ppn);
     }
}
```

这样，就可以实现把某一块已分配的物理页帧映射到进程的用户态虚拟地址空间，并且在进程退出释放地址空间的物理页帧时，不会把显存的物理页帧给释放掉。

## 2.4 图形化应用程序设计

### 移植 embedded-graphics 嵌入式图形库

`embedded-graphics` 嵌入式图形库给出了很详细的移植说明， 主要是实现 `embedded_graphics_core::draw_target::DrawTarget` trait中的函数接口 `fn draw_iter<I>(&mut self, pixels: I)` 。为此需要为图形应用建立一个能够表示显存、像素点特征和显示区域的数据结构 `Display` 和创建函数 `new()` ：

```rust
pub struct Display {
    pub size: Size,
    pub point: Point,
    pub fb: &'static mut [u8],
}
impl Display {
    pub fn new(size: Size, point: Point) -> Self {
        let fb_ptr = framebuffer() as *mut u8;
        println!(
            "Hello world from user mode program! 0x{:X} , len {}",
            fb_ptr as usize, VIRTGPU_LEN
        );
        let fb =
            unsafe { core::slice::from_raw_parts_mut(fb_ptr as *mut u8, VIRTGPU_LEN as usize) };
        Self { size, point, fb }
    }
}
```

在这个 `Display` 结构的基础上，我们就可以实现 `DrawTarget` trait 要求的函数：

```rust
impl OriginDimensions for Display {
    fn size(&self) -> Size {
        self.size
    }
}

impl DrawTarget for Display {
    type Color = Rgb888;
    type Error = core::convert::Infallible;

    fn draw_iter<I>(&mut self, pixels: I) -> Result<(), Self::Error>
    where
        I: IntoIterator<Item = embedded_graphics::Pixel<Self::Color>>,
    {
        pixels.into_iter().for_each(|px| {
            let idx = ((self.point.y + px.0.y) * VIRTGPU_XRES as i32 + self.point.x + px.0.x)
                as usize
                * 4;
            if idx + 2 >= self.fb.len() {
                return;
            }
            self.fb[idx] = px.1.b();
            self.fb[idx + 1] = px.1.g();
            self.fb[idx + 2] = px.1.r();
        });
        framebuffer_flush();
        Ok(())
    }
}
```

上述的 `draw_iter()` 函数实现了对一个由像素元素组成的显示区域的绘制迭代器，将迭代器中的像素元素绘制到 `Display` 结构中的显存中，并调用 `framebuffer_flush()` 函数将显存中的内容刷新到屏幕上。这样， `embedded-graphics` 嵌入式图形库在rCore的移植任务就完成了。

### 实现贪吃蛇游戏图形应用

`embedded-snake-rs` 的具体实现大约有200多行代码，提供了一系列的数据结构，主要的数据结构（包含相关方法实现）包括：

- `ScaledDisplay` ：封装了 `Dislpay` 并支持显示大小可缩放的方块
- `Food` ：会在随机位置产生并定期消失的”食物”方块
- `Snake` : “贪吃蛇” 方块，长度由一系列的方块组成，可以通过键盘控制方向，碰到食物会增长
- `SnakeGame` ：食物和贪吃蛇的游戏显示配置和游戏状态

有了上述事先准备的数据结构，我们就可以实现贪吃蛇游戏的主体逻辑了。

```rust
pub fn main() -> i32 {
    // 创建具有virtio-gpu设备显示内存虚地址的Display结构
    let mut disp = Display::new(Size::new(1280, 800), Point::new(0, 0));
    // 初始化游戏显示元素的配置：红色的蛇、黄色的食物，方格大小为20个像素点
    let mut game = SnakeGame::<20, Rgb888>::new(1280, 800, 20, 20, Rgb888::RED, Rgb888::YELLOW, 50);
    // 清屏
    let _ = disp.clear(Rgb888::BLACK).unwrap();
    // 启动游戏循环
    loop {
        if key_pressed() {
            let c = getchar();
            match c {
                LF => break,
                CR => break,
                // 调整蛇行进方向
                b'w' => game.set_direction(Direction::Up),
                b's' => game.set_direction(Direction::Down),
                b'a' => game.set_direction(Direction::Left),
                b'd' => game.set_direction(Direction::Right),
                _ => (),
            }
        }
        //绘制游戏界面
        let _ = disp.clear(Rgb888::BLACK).unwrap();
        game.draw(&mut disp);
        //暂停一小会
        sleep(10);
    }
    0
}
```

这里看到，为了判断通过串口输入的用户是否按键，我们扩展了一个系统调用 `sys_key_pressed` ：

```rust
// os/src/syscall/input.rs
pub fn sys_key_pressed()  -> isize {
    let res =!UART.read_buffer_is_empty();
    if res {
        1
    } else {
        0
    }
}
```

这样，我们结合串口和 `virtio-gpu` 两种外设，并充分利用已有的Rust库，设计实现了一个 `贪吃蛇` 小游戏（如下图所示）。至此，基于侏罗猎龙操作系统的图形应用开发任务就完成了。

<img src="https://cdn.jsdelivr.net/gh/MaskerDad/BlogImage@main/202312011128139.png" alt="image-20231201112805837" style="zoom: 25%;" />







