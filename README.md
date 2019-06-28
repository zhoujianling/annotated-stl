# Annotated STL

这是一个仓库，里面放了一些我注释过的 STL。STL 来自 9.1.0 版 gcc 里面的 libstdc++。



## STL内存管理



### 关于 Allocator

狭义上说，STL 容器需要一定内存空间来存储我们放入的元素。比如，每次我们往 vector 中 push_back 元素的时候，STL容器会自动为放入的元素分配堆里面的内存。一般而言，我们可以直接用 new 关键字来分配内存，但是 STL 提供了更灵活的机制，就是用 Allocator 来分配空间。

一般我们用 new 关键字在堆里面构造对象，实际上做了两件事：

1. 用 ::operator new 在堆里面分配对象所需空间（注意 ::operator new 和 new 是不一样的）
2. 调用构造函数，构造出对象



类似的，对于 vector<T> arr(1); 这样的声明，容器的 allocator 也需要两步来构造对象：

1. 在堆里分配对象所需的空间
2. 调用构造函数，把对象放在分配好的内存上

当 STL 容器析构的时候，对于容器内元素的析构，则需要：

1. 调用析构函数，清理资源
2. 释放掉为该对象分配的空间

所以一个 Allocator 的定义一般如下：

![1561641296219](https://jimmie00x0000.github.io/img/annotated-stl/1.png)

其中，allocate() 负责分配空间，construct() 负责调用对象的构造函数，destroy() 负责调用对象的析构函数，deallocate() 负责释放已分配的空间。



---

如果去看 vector 的定义，可以看到：

```c++
  template<typename _Tp, typename _Alloc = std::allocator<_Tp> >
    class vector : protected _Vector_base<_Tp, _Alloc>
    {/** ... **/ };
```

模板参数 _Alloc 就是 Allocator 的类型，其默认类型为定义在 <bits/allocator.h> 里的 allocator<T>。然而如果去看 allocator 的定义，发现其具体实现要追溯到其基类。

![1561642558300](https://jimmie00x0000.github.io/img/annotated-stl/2.png)



### new_allocator

gcc 中的默认 allocator 是 new_allocator，定义在 ext/new_allocator.h 里面。

new_allocator 使用 ::operator new 来分配空间，其底层就是最常用的 malloc()。

来看一下其 allocate() 方法的实现，就是通过调用 ::operator new (size_t ) 来完成空间分配的。

```c++
      pointer
      allocate(size_type __n, const void* = static_cast<const void*>(0))
      {
	if (__n > this->max_size())
	  std::__throw_bad_alloc();

#if __cpp_aligned_new
	if (alignof(_Tp) > __STDCPP_DEFAULT_NEW_ALIGNMENT__)
	  {
	    std::align_val_t __al = std::align_val_t(alignof(_Tp));
	    return static_cast<_Tp*>(::operator new(__n * sizeof(_Tp), __al));
	  }
#endif
	return static_cast<_Tp*>(::operator new(__n * sizeof(_Tp)));
      }
```

更具体的注释将直接在仓库源码中给出。

。

### pool_allocator

pool_allocator 定义在 <ext/pool_allocator.h> 里，实际类名为 \_\_pool_alloc， 其继承于  __pool_alloc_base 。
![A-3](https://jimmie00x0000.github.io/img/annotated-stl/3.png)

pool_allocator 采用如下机制分配内存：

1. 如果被全局强制要求用 new 分配，使用 :: operator new 为对象分配内存
2. 如果要求分配的空间大小超过阈值 s_max，则直接用 :: operator new 分配内存
3. 如果小于阈值 s_max，则通过数据结构 free_list（空闲链表）来分配内存



注：如果了解 malloc 的底层实现机制，会发现这里的 free_list 和 malloc 的分配器实现并无太大区别。



该分配器的 allocate() 实现如下：

```c++
  template<typename _Tp>
    _Tp*
    __pool_alloc<_Tp>::allocate(size_type __n, const void*)
    {
      pointer __ret = 0;
      if (__builtin_expect(__n != 0, true))
	{
	  if (__n > this->max_size())
	    std::__throw_bad_alloc();

	  const size_t __bytes = __n * sizeof(_Tp);

#if __cpp_aligned_new
	  if (alignof(_Tp) > __STDCPP_DEFAULT_NEW_ALIGNMENT__)
	    {
	      std::align_val_t __al = std::align_val_t(alignof(_Tp));
	      return static_cast<_Tp*>(::operator new(__bytes, __al));
	    }
#endif

	  // If there is a race through here, assume answer from getenv
	  // will resolve in same direction.  Inspired by techniques
	  // to efficiently support threading found in basic_string.h.
	  if (_S_force_new == 0)
	    {
	      if (std::getenv("GLIBCXX_FORCE_NEW"))
		__atomic_add_dispatch(&_S_force_new, 1);
	      else
		__atomic_add_dispatch(&_S_force_new, -1);
	    }
		
      // 如果要求分配的内存空间大于 s_max ，或者强制使用 new 分配
	  if (__bytes > size_t(_S_max_bytes) || _S_force_new > 0)
	    __ret = static_cast<_Tp*>(::operator new(__bytes));
	  else
      // 使用空闲链表进行分配
	    {
	      _Obj* volatile* __free_list = _M_get_free_list(__bytes);
	      
	      __scoped_lock sentry(_M_get_mutex());
	      _Obj* __restrict__ __result = *__free_list;
	      if (__builtin_expect(__result == 0, 0))
		__ret = static_cast<_Tp*>(_M_refill(_M_round_up(__bytes)));
	      else
		{
		  *__free_list = __result->_M_free_list_link;
		  __ret = reinterpret_cast<_Tp*>(__result);
		}
	      if (__ret == 0)
		std::__throw_bad_alloc();
	    }
	}
      return __ret;
    }
```



。

### bitmap_allocator



## 容器

### 变长数组 vector

## 算法

## 参考文献

* https://gcc.gnu.org/onlinedocs/libstdc++/manual/memory.html
* 

