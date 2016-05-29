---
layout: post
categories: [Programming]
tags: [Linux, Parallel programming]
---
# Linux RCU Q&A #  
---

###  Q1: Linux RCU是什么 ###    
A: Read-Copy-Update, 是Linux内核里的一种"锁"机制, 可以保证并发读的时候基本无加锁的等待, 不同于rwlock, RCU在写入的时候也可以读  

###  Q2: RCU是如何执行写入操作的 ###  
A: RCU在写入的时候, 采用以下步骤(需要修改的内容在结构体A内):   
1. 新建一个结构体, 将原结构体A拷贝到新结构体内(copy),   
2. 根据需要修改结构体的内容(update),   
3. 然后将原来指向该结构体的指针指向新的结构体. 此时原结构体A出现两份   
4. 待所有读操作完成后, 删除原结构体, 这样结构体A只有一份最新的  
以链表为例:  
1. 初始状态: A--->B--->C  
2. copy: B' = malloc(sizeof(B)); B' <= B  
3. update: B'->x = value; 此时存在A, B, C和B', A--->B--->C, B'--->C, 这里借用了赋值操作的原子性  
4. 修改指针, 得到: A--->B'--->C, B--->C  
5. 待所有对于B的读操作完成后, free(B), 此时变为A--->B'--->C  
由此可见写操作的损耗是比较大的, 涉及到内存分配, 内存拷贝等操作

###  Q3: RCU是如何执行读操作的 ###  
A: 仍然以前面的链表为例:  
1. 在步骤4之前发生的读操作, 获取到的是B, 在读操作期间, B不会被释放, 读操作可以正常完成  
2. 在步骤4之后发生的读操作, 获取到的是新的B', 同样可以正常完成操作  
3. 只有在步骤4之前发生的读操作完成后, B才会被释放, 此时是安全的  
由此可见读操作完全可以并发, 原则上是不需要加锁的, RCU唯一要判断的是"步骤4之前发生的读操作完成"这一条件

###  Q4: 那么, 如何判断读操作已完成? ###    
A: 内核提供了两个rcu api: rcu_read_lock和rcu_read_unlock, 两者之间的内容就是对于rcu保护对象的读操作  
前面链表例子的步骤5会释放结构体B, 如果此时对于B还有引用, 后续会导致异常, 因此这一步操作必须确认读操作已经调用rcu_read_unlock.   
从代码可以看出rcu_read_lock是关闭内核抢占, 而rcu_read_unlock是打开内核抢占, 因此RCU确保了读操作期间内核是禁用抢占的(对于当前这个CPU核), 那么写操作在释放原结构体之前, 可以尝试去抢占每个核, 如果每个核都能抢占到, 说明每个核的内核抢占都打开着, 也说明所有的核都没有正在读的操作.  
当然在轮询每个核的时候, 已经轮询过的核, 有可能有新的读操作又进来, 这是没有关系的, 从前面链表操作可以看出, 步骤4已经修改了链表, B'已经占据了B的位置, 新的读操作不会访问到B  
RCU提出的是一种机制, 轮询只是一种实现方式, Linux内核提供的相应接口为synchronize_rcu, 这个api会阻塞写操作, 直到完成前面的检测. 如果写操作不允许阻塞, 如果不想阻塞, 可以调用call_rcu接口, 这种情况不会阻塞, 但是会注册一个回调函数, 当判断读操作全部完成时, 会调用回调函数, 完成释放指针之类的操作.  
多核之间RCU读写流程如下:  

|	CPU 0		|		   CPU 1		 |		CPU 2  |  
| ----------------|:-----------------------:|--------------:|  
|rcu_read_lock()  |						 |			   |  
|				 | enters synchronize_rcu()|			   |  
|				 |						 |rcu_read_lock()|  
|rcu_read_unlock()|						 |			   |  
|				 | exits synchronize_rcu() |			   |   
|				 |						 |rcu_read_lock()|   

###  Q5: 并发写操作是否要加锁? ###  
A: 从内核文档看, 写操作在修改原指针的时候是需要加锁的, 用spinlock即可, 主要是为了防止多个CPU核在更新结构体时造成的冲突, 同样以链表为例, 如果不加锁, 在A--->B修改为A--->B'之前, 多个写操作可能都得到的是B对象, 同时可能导致修改不一致的冲突出现. 所以内核里推荐的读写操作如下:

```
struct el {
  struct list_head list;
  long key;
  spinlock_t mutex;
  int data;
  /* Other data fields */
};
spinlock_t listmutex;
struct el head;
int search(long key, int *result)
{
  struct list_head *lp;
  struct el *p;
  rcu_read_lock();
  list_for_each_entry_rcu(p, head, lp) {
	if (p->key == key) {
	  *result = p->data;
	  rcu_read_unlock();
	  return 1;
	}
  }
  rcu_read_unlock();
  return 0;
}
int delete(long key)
{
  struct el *p;
  spin_lock(&listmutex);
  list_for_each_entry(p, head, lp) {
	if (p->key == key) {
	  list_del_rcu(&p->list);
	  spin_unlock(&listmutex);
	  synchronize_rcu();
	  kfree(p);
	  return 1;
	}
  }
  spin_unlock(&listmutex);
  return 0;
}
```


###  Q5: 还有什么其他API? ###  
A: RCU还提供另外两个基本的API: rcu_assign_pointer和rcu_dereference.  
rcu_assign_pointer完成的是链表操作4中修改指针的操作, 将A--->B--->C改为A--->B'--->C:  

```
rcu_assign_pointer(A->next, typeof(B') B');
``` 

rcu_dereference在读操作获取指针地址时使用:  

```
	p = rcu_dereference(A->next);	
	return p->data;  
```  

对于一般的CPU如x86, arm, 这些操作实际上就是简单的赋值  

###  Q6: RCU有什么好处? 在哪用到? ###  
A: 并发读操作, 加锁开销可以忽略不计(实际上, 并没有真正的读加锁过程), 但是写操作开销比较大, 因此特别适合读多写很少的场景. 最近在读openvswitch的代码, 流表操作大量用到RCU的功能. 

###  Q7: 更详细的文档? ###  
A: 内核文档/Documentation/RCU/whatisRCU.txt里就有详细的说明, Is Parallel Programming Hard一书中同样用大篇幅介绍了RCU

	  
	 


