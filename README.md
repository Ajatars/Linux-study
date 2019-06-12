# Linux-study 
linux字符设备
## 字符设备基础 （课设你猜在哪)
字符设备:只能一个字节一个字节进行读写操作的设备,不能随机读取设备的某一数据，读取数据要按照先后数据。字符设备是面向流的设备，常见的字符设备有鼠标、键盘、串口、控制台和LED等。</br>
一般字符设备都会放在/dev目录
## 字符设备驱动与用户空间访问设备的程序三者关系
驱动程序主要工作:</br>
1.初始化,添加，删除 ```struct cdev```结构体，</br>
2.申请和释放设备号，</br>
3.填充```struct file_operations```结构中断的操作函数,</br>
4.实现```struct file_operations```结构体中的read(),write()和ioctl()等函数是驱动设计的主体工作.</br></br>
**linux内核代码中**
- 使用struct cdev结构体来抽象一个字符设备；</br>
+ 通过一个dev_t类型的设备号（分为主（major）、次设备号（minor））一确定字符设备唯一性；</br>
- 通过struct file_operations类型的操作方法集来定义字符设备提供个VFS的接口函数。 </br>
![关系](https://images2015.cnblogs.com/blog/891355/201612/891355-20161214225430558-441684263.png)

## 字符设备模型
![](https://images2015.cnblogs.com/blog/891355/201612/891355-20161213170514370-1604026374.png)

### 1.linux内核，使用struct cdev 描述一个字符设备
```
<include/linux/cdev.h>                    //头文件

struct cdev {   
　　struct kobject kobj;                  //内嵌的内核对象.  
　　struct module *owner;                 //该字符设备所在的内核模块（所有者）的对象指针，一般为THIS_MODULE主要用于模块计数  
　　const struct file_operations *ops;    //该结构描述了字符设备所能实现的操作集（打开、关闭、读/写、...），是极为关键的一个结构体
　　struct list_head list;                //用来将已经向内核注册的所有字符设备形成链表
　　dev_t dev;                            //字符设备的设备号，由主设备号和次设备号构成（如果是一次申请多个设备号，此设备号为第一个）
　　unsigned int count;                   //隶属于同一主设备号的次设备号的个数
　　...
};
```
***操作接口***
动态申请（构造）cdev内存（设备对象）```struct cdev *cdev_alloc(void);```
初始化cdev的成员，并建立cdev和file_operations之间关联起来 ```void cdev_init(struct cdev *p, const struct file_operations *p);　　```
注册cdev设备对象```int cdev_add(struct cdev *p, dev_t dev, unsigned count);```
将cdev对象从系统中移除（注销 ）```void cdev_del(struct cdev *p);```
释放cdev内存 ```void cdev_put(struct cdev *p);```
</br></br>
### 设备号申请/释放
一个字符设备或块设备都有一个主设备号和一个次设备号。主设备号用来标识与设备文件相连的驱动程序，用来反映设备类型。次设备号被驱动程序用来辨别操作的是哪个设备，用来区分同类型的设备。</br>
</br>
***设备号用dev_t来描述***
```typedef u_long dev_t;　　// 在32位机中是4个字节，高12位表示主设备号，低20位表示次设备号。```
内核也为我们提供了几个方便操作的宏实现dev_t： 
```
#define MAJOR(dev)    ((unsigned int) ((dev) >> MINORBITS))　　// 从设备号中提取主设备号
#define MINOR(dev)    ((unsigned int) ((dev) & MINORMASK))　　// 从设备号中提取次设备号
#define MKDEV(ma,mi)    (((ma) << MINORBITS) | (mi))</span>　　// 将主、次设备号拼凑为设备号
/* 只是拼凑设备号,并未注册到系统中，若要使用需要竞态申请 */

```
***头文件*** ```linux/fs.h``` 
 静态申请设备号 ```int register_chrdev_region(dev_t from, unsigned count, const char *name);```
动态分配设备号 ```int alloc_chrdev_region(dev_t *dev, unsigned baseminor, unsigned count, const char *name);```
释放设备号 ```void unregister_chrdev_region(dev_t from, unsigned count);```

### struct cdev 中的 file_operations *fops成员
Linux下一切皆是“文件”，字符设备也是这样，file_operations结构体中的成员函数是字符设备程序设计的主题内容，这些函数实际会在用户层程序进行Linux的open()、close()、write()、read()等系统调用时最终被调用。</br>
"文件"的操作接口结构：
```
struct file_operations {
　　struct module *owner;　　
　　　　/* 模块拥有者，一般为 THIS——MODULE */
　　ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);　　
　　　　/* 从设备中读取数据，成功时返回读取的字节数，出错返回负值（绝对值是错误码） */
　　ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);　　　
　　　　/* 向设备发送数据，成功时该函数返回写入字节数。若为被实现，用户调层用write()时系统将返回 -EINVAL*/
　　int (*mmap) (struct file *, struct vm_area_struct *);　　
　　　　/* 将设备内存映射内核空间进程内存中，若未实现，用户层调用 mmap()系统将返回 -ENODEV */
　　long (*unlocked_ioctl)(struct file *filp, unsigned int cmd, unsigned long arg);　　
　　　　/* 提供设备相关控制命令（读写设备参数、状态，控制设备进行读写...）的实现，当调用成功时返回一个非负值 */
　　int (*open) (struct inode *, struct file *);　　
　　　　/* 打开设备 */
　　int (*release) (struct inode *, struct file *);　　
　　　　/* 关闭设备 */
　　int (*flush) (struct file *, fl_owner_t id);　　
　　　　/* 刷新设备 */
　　loff_t (*llseek) (struct file *, loff_t, int);　　
　　　　/* 用来修改文件读写位置，并将新位置返回，出错时返回一个负值 */
　　int (*fasync) (int, struct file *, int);　　
　　　　/* 通知设备 FASYNC 标志发生变化 */
　　unsigned int (*poll) (struct file *, struct poll_table_struct *);　　
　　　　/* POLL机制，用于询问设备是否可以被非阻塞地立即读写。当询问的条件未被触发时，用户空间进行select()和poll()系统调用将引起进程阻塞 */
　　...
};
```
## 简单字符设备实例 (linux课设实验 我放最后了 来打我呀 QAQ)
前提是下载VirtualBox安装一个低版本的linux系统 打开命令行</br>
使用mkdir 创建一个文件 存放我们.c文件 我这里用一个VPS来演示。</br>

![5d00899379a4011466.jpg.jpg](https://i.loli.net/2019/06/12/5d00899379a4011466.jpg)
![1.jpg](https://i.loli.net/2019/06/12/5d0089937a23a63207.jpg)
1.使用Vim MemDev.c 命令创建驱动程序  (进入后按 i 才能输入 然后直接粘贴 这是vim编辑器语法)</br>
```
#include <linux/module.h>
#include <linux/types.h>
#include <linux/fs.h>
#include <linux/errno.h>
#include <linux/init.h>
#include <linux/cdev.h>
#include <asm/uaccess.h>
#include <linux/slab.h>

/* We suppose this is the two device's registers */
int dev1_registers[5];
int dev2_registers[5];

struct cdev cdev; 
dev_t devno;

/*文件打开函数*/
int mem_open(struct inode *inode, struct file *filp)
{
    /*获取次设备号*/
    int num = MINOR(inode->i_rdev);
    
    if (num == 0)
        filp->private_data = dev1_registers;
    else if(num == 1)
        filp->private_data = dev2_registers;
    else
        return -ENODEV;  //无效的次设备号
    
    return 0; 
}

/*文件释放函数*/
int mem_release(struct inode *inode, struct file *filp)
{
    return 0;
}

/*读函数*/
static ssize_t mem_read(struct file *filp, char __user *buf, size_t size, loff_t *ppos)
{
    unsigned long p = *ppos;
    unsigned int count = size;
    int ret = 0;
    int *register_addr = filp->private_data; /*获取设备的寄存器基地址*/

    /*判断读位置是否有效*/
    if (p >= 5 * sizeof(int))
        return 0;
    if (count > 5 * sizeof(int) - p)
        count = 5 * sizeof(int) - p;

  /*读数据到用户空间*/
    if (copy_to_user(buf, register_addr + p, count))
    {
        ret = -EFAULT;
    }
    else
    {
        *ppos += count;
        ret = count;
    }
    
    return ret;
}

/*写函数*/
static ssize_t mem_write(struct file *filp, const char __user *buf, size_t size, loff_t *ppos)
{
    unsigned long p =  *ppos;
    unsigned int count = size;
    int ret = 0;
    int *register_addr = filp->private_data; /*获取设备的寄存器地址*/
  
    /*分析和获取有效的写长度*/
    if (p >= 5*sizeof(int))
        return 0;
    
    if (count > 5 * sizeof(int) - p)
        count = 5 * sizeof(int) - p;
    
    /*从用户空间写入数据*/
    if (copy_from_user(register_addr + p, buf, count))
        ret = -EFAULT;
    else
    {
        *ppos += count;
        ret = count;
    }
    
    return ret;
}

/* seek文件定位函数 */
static loff_t mem_llseek(struct file *filp, loff_t offset, int whence)
{ 
    loff_t newpos;

    switch(whence) {
        case SEEK_SET: 
            newpos = offset;
            break;

        case SEEK_CUR: 
            newpos = filp->f_pos + offset;
            break;

        case SEEK_END: 
            newpos = 5 * sizeof(int) - 1 + offset;
            break;
        
        default: 
            return -EINVAL;
    }
    
    if ((newpos < 0) || (newpos > 5 * sizeof(int)))
        return -EINVAL;
        
    filp->f_pos = newpos;
    
    return newpos;
}

/*文件操作结构体*/
static const struct file_operations mem_fops =
{
    .open = mem_open,    
    .read = mem_read,
    .write = mem_write,
    .llseek = mem_llseek,
    .release = mem_release,
};

/*设备驱动模块加载函数*/
static int memdev_init(void)
{
    /*初始化cdev结构*/
    cdev_init(&cdev, &mem_fops);
  
    /* 注册字符设备 */
    alloc_chrdev_region(&devno, 0, 2, "memdev");
    
    cdev_add(&cdev, devno, 2);
}

/*模块卸载函数*/
static void memdev_exit(void)
{
    cdev_del(&cdev);   /*注销设备*/
    unregister_chrdev_region(devno, 2); /*释放设备号*/
}

MODULE_LICENSE("GPL");

module_init(memdev_init);
module_exit(memdev_exit);

```
![2.jpg](https://i.loli.net/2019/06/12/5d0089939aae027252.jpg)</br>
2.测试代码MemWrite.c 也是vim MemWrite.c (回车) i 后粘贴进去</br>
```
#include <stdio.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>

int main()
{
    int fd = 0;
    int src0[] = {0, 1, 2, 3, 4};
    int src1[] = {10, 11, 12, 13, 14};
    
    /*打开设备文件*/
    fd = open("/dev/memdev0", O_RDWR);
    
    /*写入数据*/
    write(fd, src0, sizeof(src0));
    
    /*关闭设备*/
    close(fd);
    
    fd = open("/dev/memdev1", O_RDWR);
    
    /*写入数据*/
    write(fd, src1, sizeof(src1));
    
    /*关闭设备*/
    close(fd);
    
    return 0;    
}
```
3.测试代码MemRead.c 同样</br>
```
#include <stdio.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>

int main()
{
    int fd = 0;
    int dst = 0;
    
    /*打开设备文件*/
    fd = open("/dev/memdev0", O_RDWR);
    
    lseek(fd, 2, SEEK_SET);
    
    /*写入数据*/
    read(fd, &dst, sizeof(int));
    
    printf("dst0 is %d\n", dst);
    
    /*关闭设备*/
    close(fd);
    
    /*打开设备文件*/
    fd = open("/dev/memdev1", O_RDWR);
    
    lseek(fd, 3, SEEK_SET);
    /*写入数据*/
    read(fd, &dst, sizeof(int));
    
    printf("dst1 is %d\n", dst);
    
    /*关闭设备*/
    close(fd);
    
    return 0;    
}
```

4.这个是重点了 使用make编译我们的驱动文件</br>
vim Makefile  i  后粘贴</br>
```
ifneq ($(KERNELRELEASE),)
    obj-m := MemDev.o#obj-m 指编译成外部模块
else
    KERNELDIR := /lib/modules/$(shell uname -r)/build #定义一个变量,指向内核目录
    PWD := $(shell pwd)
modules:
    $(MAKE) -C $(KERNELDIR) M=$(PWD) modules
endif
clean:
    $(MAKE) -C $(KERNELDIR) M=$(PWD) clean
```
这里注意下 命令前面要用 tab键(红色前面删了空格 改成TAB）</br>
不能为红色像这样:</br>
![3.jpg](https://i.loli.net/2019/06/12/5d008993a12c176922.jpg)</br>

正确是这样:</br>
![4.jpg](https://i.loli.net/2019/06/12/5d008993a1ad179662.jpg)</br>
5. 生成ko驱动模块文件 </br>
命令行直接输入 ``` make``` </br>
![5.jpg](https://i.loli.net/2019/06/12/5d008993b3f3390012.jpg)
insmod 命令加载：```insmod MemDev.ko ```</br>
![6.jpg](https://i.loli.net/2019/06/12/5d008993a649a38761.jpg)
cat /proc/devices 查看</br>
![8.jpg](https://i.loli.net/2019/06/12/5d008993a68f085787.jpg)
然后</br>
```
gcc MemRead.c -o MemRead
gcc MemWrite.c -o  MemWrite
mknod /dev/memdev0 c 主设备号 0  （我这里主设备号是245)
``` 
再然后
```
./MemRead
```
没写入时为0</br>
![9.jpg](https://i.loli.net/2019/06/12/5d008993a6cb592340.jpg)
```
./MemWrite
```
写入后</br>
![10.jpg](https://i.loli.net/2019/06/12/5d008ab0ab83049264.jpg)
