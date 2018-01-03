---
layout: post
title: 无锁数据结构-tagged pointers and hazard pointer
category: 无锁数据结构
tags: 无锁 , hazard pointer , tagged pointer 
---

# 无锁数据结构的两大问题

实现无锁数据结构最困难的两个问题是ABA问题和内存回收问题。它们之间存在着一定的关联：一般内存回收问题的解决方案，可以作为解决ABA问题的一种只需很少开销或者根本不需额外开销的方法(**内存不回收,ABA中两个A地址不同**)，但也存在一些情况并不可行，如两个链表实现的栈，不断在两个栈间交换节点。

## 标签指针（Tagged pointers）

标签指针作为一种规范由IBM引入，旨在解决ABA问题。从某一方面来看，ABA问题的出现是由于不能区分前一个A与后一个A，标签指针通过标签（版本号）来解决。其每个指针由一组原子性的内存单元和标签（32比特的整数）组成。

> ```c++
> template <typename T>
> struct tagged_ptr {
>     T * ptr ;
>     unsigned int tag ;
>     tagged_ptr(): ptr(nullptr), tag(0) {}
>     tagged_ptr( T * p ): ptr(p), tag(0) {}
>     tagged_ptr( T * p, unsigned int n ): ptr(p), tag(n){}
>     T * operator->() const { return ptr; }
> };
> ```
> 标签作为一个版本号，随着标签指针上的每次CAS运算增加，并且只增不减。一旦需要从容器中非物理地移除某个元素，就应将其放入一个放置空闲元素的列表中。在空闲元素列表中，逻辑删除的元素完全有可能被再次调用。因为是无锁数据结构，一个线程删除X元素，另外一个线程依然可以持有标签指针的本地副本，并指向元素字段。因此需要一个针对每种T类型的空闲元素列表。多数情况下，将元素放入空闲列表中，意味着调用这个T类型数据的析构函数是非法的（考虑到并行访问，在析构函数运算的过程中，其它线程是可以读到此元素的数据）。

有了标签，A在CAS时虽然内存地址相同，但若A已被使用过，其标签不同。不会出现ABA问题。

当然其必然也有缺点：

其一：要实现标签指针需要平台支持，平台需要支持dwCAS（地址指针需一个word，标签至少需32位），由于现代的系统架构都有一套完整的64位指令集，对于32位系统，dwCAS需要64位，可以支持；但对于64位系统，dwCAS需要128位（至少96位），并不是所有架构都能支持。
> The scheme is implemented at platforms, which have an atomic CAS primitive over a double word (dwCAS). This requirement is fulfilled for 32-bit modern systems as dwCAS operates with 64-bit words while all modern architectures have a complete set of 64-bit instructions. But 128-bit (or at least 96-bit) dwCAS is required for 64-bit operation mode. It isn’t implemented in all architectures.

其二：空闲列表通常以无锁栈或无锁队列的方式实现，对性能也会有影响，但也正是因为使用无锁，导致其性能有提高（相对于有锁）。
其三: 对于每种类型提供单独的空闲列表,这样做太过奢侈难以被大众所接收，一些应用使用内存太过低效。例如，无锁队列通常包含10个元素，但可以扩展到百万，比如在一次阻塞后，空闲列表扩展至百万。
> Availability of a separate free-list for every data type can be an unattainable luxury for some applications as it can lead to inefficient memory use. For example, if a lock-free queue consists of 10 elements on the average but its size can increase up to 1 million, the free-list size after the spike can be about 1 million. Such behavior is often illegal.

### Example

详情可参考1

```c++
template <typename T> struct node {
    tagged_ptr next;
    T data;
} ;
template <typename T> class MSQueue {
   tagged_ptr<T> volatile m_Head;
   tagged_ptr<T> volatile m_Tail;
   FreeList m_FreeList;
public:
   MSQueue()
   {
     // Allocate dummy node
     // Head & Tail point to dummy node
     m_Head.ptr = m_Tail.ptr = new node();
   }
void enqueue( T const& value )
{
E1: node * pNode = m_FreeList.newNode();
E2: pNode–>data = value;
E3: pNode–>next.ptr = nullptr;
E4: for (;;) {
E5:   tagged_ptr<T> tail = m_Tail;
E6:   tagged_ptr<T> next = tail.ptr–>next;
E7:   if tail == Q–>Tail {
         // Does Tail point to the last element?
E8:      if next.ptr == nullptr {
            // Trying to add the element in the end of the list
E9:         if CAS(&tail.ptr–>next, next, tagged_ptr<T>(node, next.tag+1)) {
              // Success, leave the loop
E10:          break;
            }
E11:     } else {
            // Tail doesn’t point to the last element
            // Trying to relocate tail to the last element
E12:        CAS(&m_Tail, tail, tagged_ptr<T>(next.ptr, tail.tag+1));
         }
      }
    } // end loop
    // Trying to relocate tail to the inserted element
E13: CAS(&m_Tail, tail, tagged_ptr<T>(pNode, tail.tag+1));
 }
bool dequeue( T& dest ) {
D1:  for (;;) {
D2:    tagged_ptr<T> head = m_Head;
D3:    tagged_ptr<T> tail = m_Tail;
D4:    tagged_ptr<T> next = head–>next;
       // Head, tail and next consistent?
D5:    if ( head == m_Head ) {
          // Is queue empty or isn’t tail the last?
D6:       if ( head.ptr == tail.ptr ) {
            // Is the queue empty?
D7:         if (next.ptr == nullptr ) {
               // The queue is empty
D8:                  return false;
            }
            // Tail isn’t at the last element
            // Trying to  move tail forward
D9:         CAS(&m_Tail, tail, tagged_ptr<T>(next.ptr, tail.tag+1>));
D10:      } else { // Tail is in position
            // Read the value before CAS, as otherwise  
            // another dequeue can deallocate next
D11:        dest = next.ptr–>data;
            // Trying to move head forward
D12:        if (CAS(&m_Head, head, tagged_ptr<T>(next.ptr, head.tag+1))
D13:           break // Success, leave the loop
          }
       }
     } // end of loop
     // Deallocate the old dummy node
D14: m_FreeList.add(head.ptr);
D15: return true; // the result is in dest
  }
```

## 险象指针（Hazard pointer）

此规则由Michael创建（Hazard Pointers: Safe Memory Reclamation for Lock-Free Objects），用来保护无锁数据结构中元素的局部引用，也即延迟删除。
> it’s the most popular and studied scheme of delayed deletion.

此规则的实现仅依赖原子性读写，而未采用任何重量级的CAS同步原语。其实现原理大致如下：

1. 通过使用线程局部数据保存正在使用的共享内存（数据结构元素），以及要删除的共享内存，记为Plist，Dlist。
2. 每个线程都将自己正在访问其不希望被释放的内存对象保存在Plist中，使用完取出。
3. 当任何线程删除内存对象，把对象放入Dlist。
4. 当Dlist中元素数量达到阈值，扫描Dlist和Plist，真正释放在Dlist中有而所以Plist中没有的元素。

Note：**Plist 一写多读；Dlist单写单读。**

hazard pointer的使用是要结合具体的数据结构的，我们需要分析所要保护的数据结构的每一步操作，找出需要保护的内存对象并使用hazard pointer替换普通指针对危险的内存访问进行保护。

### Example

详情可参考3

```c++
template <typename T>
void Queue<T>::enqueue(const T &data)
{
  qnode *node = new qnode();
  node->data = data;
  node->next = NULL;
  // qnode *t = NULL;
  HazardPointer<qnode> t(hazard_mgr_);
  qnode *next = NULL;
 
  while (true) {
    if (!t.acquire(&tail_)) {
      continue;
    }
    next = t->next;
    if (next) {
      __sync_bool_compare_and_swap(&tail_, t, next);
      continue;
    }
    if (__sync_bool_compare_and_swap(&t->next, NULL, node)) {
      break;
    }
  }
  __sync_bool_compare_and_swap(&tail_, t, node);
}
 
template <typename T>
bool Queue<T>::dequeue(T &data)
{
  qnode *t = NULL;
  // qnode *h = NULL;
  HazardPointer<qnode> h(hazard_mgr_);
  // qnode *next = NULL;
  HazardPointer<qnode> next(hazard_mgr_);
 
  while (true) {
    if (!h.acquire(&head_)) {
      continue;
    }
    t = tail_;
    next.acquire(&h->next);
    asm volatile("" ::: "memory");
    if (head_ != h) {
      continue;
    }
    if (!next) {
      return false;
    }
    if (h == t) {
      __sync_bool_compare_and_swap(&tail_, t, next);
      continue;
    }
    data = next->data;
    if (__sync_bool_compare_and_swap(&head_, h, next)) {
      break;
    }
  }
 
  /* h->next = (qnode *)1; // bad address, It's a trap! */
  /* delete h; */
  hazard_mgr_.retireNode(h);
  return true;
}
```
## 参考资料

1. [无锁数据结构（机制篇）：内存管理规则](http://blog.jobbole.com/107955/) ([EN 原文](https://kukuruku.co/post/lock-free-data-structures-the-inside-memory-management-schemes/))
2. [风险指针(Hazard Pointers)——用于无锁对象的安全内存回收机制](https://www.cnblogs.com/coryxie/archive/2013/02/01/2951249.html)
3. [http://blog.kongfy.com/2017/02/hazard-pointer/](http://blog.kongfy.com/2017/02/hazard-pointer/)