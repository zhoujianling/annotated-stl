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
vector 是 STL 最常用的数据容器之一，用来顺序放置元素。其底层实现是一块连续空间，每次往分配的空间里放元素，如果刚好放满了，就对已分配空间扩容。

在 x86-64 架构下，如果我们在代码中计算 sizeof(任意vector)，我们会发生计算出来的大小为 24。原因很简单，数组是在堆中分配的，vector对象本身只维护三个指针（见下图的 \_Vector_impl_data 的三个成员变量），分别指向已分配空间开始位置、当前元素放到哪儿了、已分配空间的结束位置。

![1561641296219](https://jimmie00x0000.github.io/img/annotated-stl/5.png)

vector 的实际的实现在 <bits/stl_vector.h> 中，其继承于 _Vector_base, 类图如下：

![1561641296219](https://jimmie00x0000.github.io/img/annotated-stl/4.png)

调用 push_back 向 vector 添加元素，如果 _M_finish 指针小于 _M_end_of_storage ，则可以继续愉快地插入数据，否则调用 _M_relloc_insert 实现扩容并插入：

```c++
#if __cplusplus >= 201103L
  template<typename _Tp, typename _Alloc>
    template<typename... _Args>
      void
      vector<_Tp, _Alloc>::
      _M_realloc_insert(iterator __position, _Args&&... __args)
#else
  template<typename _Tp, typename _Alloc>
    void
    vector<_Tp, _Alloc>::
    _M_realloc_insert(iterator __position, const _Tp& __x)
#endif
    {
	// _M_check_len 一般情况下返回 size() * 2，即扩容后的大小
      const size_type __len =
	_M_check_len(size_type(1), "vector::_M_realloc_insert");
	// 记录旧指针位置
      pointer __old_start = this->_M_impl._M_start;
      pointer __old_finish = this->_M_impl._M_finish;
	// 之前放了多少个元素
      const size_type __elems_before = __position - begin();
	// 执行扩容
      pointer __new_start(this->_M_allocate(__len));
      pointer __new_finish(__new_start);
      __try
	{
	  // The order of the three operations is dictated by the C++11
	  // case, where the moves could alter a new element belonging
	  // to the existing vector.  This is an issue only for callers
	  // taking the element by lvalue ref (see last bullet of C++11
	  // [res.on.arguments]).
	  // 把要插入的新元素 移动/拷贝 到新存储分配的空间的对应位置上
	  // 此时旧的空间上的元素还没有拷过来
	  _Alloc_traits::construct(this->_M_impl,
				   __new_start + __elems_before,
#if __cplusplus >= 201103L
				   std::forward<_Args>(__args)...);
#else
				   __x);
#endif
	  __new_finish = pointer();

#if __cplusplus >= 201103L
	  if _GLIBCXX17_CONSTEXPR (_S_use_relocate())
	    {
	      __new_finish = _S_relocate(__old_start, __position.base(),
					 __new_start, _M_get_Tp_allocator());

	      ++__new_finish;

	      __new_finish = _S_relocate(__position.base(), __old_finish,
					 __new_finish, _M_get_Tp_allocator());
	    }
	  else
#endif
	    {
		// 因为新插入的元素可能在数组中的任何位置（insert也会导致扩容），所以分两次拷贝原空间里的数据
		// 先拷贝 [_old_start, _position) 到 [_new_start, )，
		// 再拷贝 [_position, _old_finish) 到 [_new_finish, ) 上去
	      __new_finish
		= std::__uninitialized_move_if_noexcept_a
		(__old_start, __position.base(),
		 __new_start, _M_get_Tp_allocator());

	      ++__new_finish;

		// __uninitialized_move_if_noexcept_a 在 <bits/stl_uninitialized> 中，
	      __new_finish
		= std::__uninitialized_move_if_noexcept_a
		(__position.base(), __old_finish,
		 __new_finish, _M_get_Tp_allocator());
	    }
	}
      __catch(...)
	{
	  if (!__new_finish)
	    _Alloc_traits::destroy(this->_M_impl,
				   __new_start + __elems_before);
	  else
	    std::_Destroy(__new_start, __new_finish, _M_get_Tp_allocator());
	  _M_deallocate(__new_start, __len);
	  __throw_exception_again;
	}
#if __cplusplus >= 201103L
      if _GLIBCXX17_CONSTEXPR (!_S_use_relocate())
#endif
	// 析构扩容前空间里的对象
	std::_Destroy(__old_start, __old_finish, _M_get_Tp_allocator());
      _GLIBCXX_ASAN_ANNOTATE_REINIT;
	// 释放扩容前使用的空间
      _M_deallocate(__old_start,
		    this->_M_impl._M_end_of_storage - __old_start);
	// 重新定位三个指针
      this->_M_impl._M_start = __new_start;
      this->_M_impl._M_finish = __new_finish;
      this->_M_impl._M_end_of_storage = __new_start + __len;
    }

```

更多的分析将在源码中给出。

### 双端队列 deque

deque 相比于 vector，可以实现在常数时间内向头部插入数据（push_front）。因为数组的头部插入效率是 O(n)，此时再使用内存上的连续数组便无法满足这样的需求。deque 使用**分段连续空间**来存储数据，其数据结构如下图所示：



在 x86-64架构下，如果我们计算 sizeof(deque>，会得到80字节的大小。原因见下面的类图，因为 _Deque_iterator 包含4个指针，大小为 4 * 8 = 32，一个 _Deque_impl 包含两个这样的迭代器，一个 size_t 类型的成员，和一个  _Tp ** 类型的指针，故大小为 32 * 2 + 8 + 8 = 80。类图如下：
![1561642558300](https://jimmie00x0000.github.io/img/annotated-stl/6.png)



### map
### unordered_map



## 算法

### 排序 sort



## 参考文献

* https://gcc.gnu.org/onlinedocs/libstdc++/manual/memory.html
* https://sploitfun.wordpress.com/2015/02/10/understanding-glibc-malloc/
* https://gcc.gnu.org/wiki/Visibility
* https://blog.csdn.net/fengbingchun/article/details/78898623
* 

