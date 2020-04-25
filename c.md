---
title: c++杂项
tags: 基础
categories: C
date: 2019-01-04 20:07:14
---

## 未归类
1. 类型转换
 - static_cast<type>()  
    - 基本类型转换: 同整数类型截断/不同类型值解释
    - 同继承体系同进行转换:
    一般来说子类指针或者栈对象向父类转换,前者属于多态,后者不是,也就是说使用显示的static_cast<\>转换可以增加编译期检查从而确保安全性
    - 任意类型指针与void*转换
    
 - dynamic_cast
    - 向上类型转换和static_cast相同
    - 向下类型转换会做编译器检查,如果出现问题,则返回null
 - const_cast 解除const
 - reinterpret_cast 
   - 将直接按照位模式解释
2. std::swap() 内存转移基于移动构造
{%codeblock std::swap lang:cpp%}
template<typename _Tp>
    inline void
    swap(_Tp& __a, _Tp& __b)
#if __cplusplus >= 201103L
    noexcept(__and_<is_nothrow_move_constructible<_Tp>,
	            is_nothrow_move_assignable<_Tp>>::value)
#endif
    {
      // concept requirements
      __glibcxx_function_requires(_SGIAssignableConcept<_Tp>)

      _Tp __tmp = _GLIBCXX_MOVE(__a); //这个宏就是std::move
      __a = _GLIBCXX_MOVE(__b);
      __b = _GLIBCXX_MOVE(__tmp);
    }
{%endcodeblock%}
该函数使用实际上将根据a,b的移动构造进行权限转移,进一步的证实我认为的拷贝进行拷贝值行为,移动进行权限转移行为.
3. std::move  实际是一个强制类型转换
{%codeblock std::move lang:cpp%}
template<typename _Tp>
    constexpr typename std::remove_reference<_Tp>::type&&
    move(_Tp&& __t) noexcept
    { return static_cast<typename std::remove_reference<_Tp>::type&&>(__t); }
{%endcodeblock%}
实际上就是将`_t`类型转为`&&`右值
4. 关于模板使用typename
 如上`constexpr typename std::remove_reference<_Tp>::type&&` 表示一个返回值
 ```c++
 template<typename _Tp>
    struct remove_reference
    { typedef _Tp   type; };

  template<typename _Tp>
    struct remove_reference<_Tp&>
    { typedef _Tp   type; };

  template<typename _Tp>
    struct remove_reference<_Tp&&>
    { typedef _Tp   type; };
 ```
 `::`符号之后可以跟变量,类型名,typename就是为了告知编译器这是一个类型不是变量
 
 5. std::uninitialized_copy(a,b,c) 将b-a范围中的数据复制到c-->c+(b-a)区间,返回的指针是c+(b-a)的位置
    std::uninitialized_copy(std::make_move_iterator(a),std::make_move_iterator(b),c)
    //可以触发移动语义
 ```c++
  template<bool _TrivialValueTypes>
    struct __uninitialized_copy
    {
      template<typename _InputIterator, typename _ForwardIterator>
        static _ForwardIterator
        __uninit_copy(_InputIterator __first, _InputIterator __last,
		      _ForwardIterator __result)
        {
	  _ForwardIterator __cur = __result;
	  __try
	    {
	      for (; __first != __last; ++__first, ++__cur)
		std::_Construct(std::__addressof(*__cur), *__first);  //实际上还是使用的allocated,这样样子实现的就是拷贝语义,是不能进行转移语义的
	      return __cur;
	    }
	  __catch(...)
	    {
	      std::_Destroy(__result, __cur);
	      __throw_exception_again;
	    }
	}
    };
    
  /**
   *  @brief Copies the range [first,last) into result.
   *  @param  __first  An input iterator.
   *  @param  __last   An input iterator.
   *  @param  __result An output iterator.
   *  @return   __result + (__first - __last)
   *
   *  Like copy(), but does not require an initialized output range.
  */
  template<typename _InputIterator, typename _ForwardIterator>
    inline _ForwardIterator
    uninitialized_copy(_InputIterator __first, _InputIterator __last,
		       _ForwardIterator __result)
    {
      typedef typename iterator_traits<_InputIterator>::value_type
	_ValueType1;
      typedef typename iterator_traits<_ForwardIterator>::value_type
	_ValueType2;
#if __cplusplus < 201103L
      const bool __assignable = true;
#else
      // trivial types can have deleted assignment
      typedef typename iterator_traits<_InputIterator>::reference _RefType1;
      typedef typename iterator_traits<_ForwardIterator>::reference _RefType2;
      const bool __assignable = is_assignable<_RefType2, _RefType1>::value;
#endif

      return std::__uninitialized_copy<__is_trivial(_ValueType1)
				       && __is_trivial(_ValueType2)
				       && __assignable>::
	__uninit_copy(__first, __last, __result);  //调用__uninitialized_copy::__uninit_copy函数进行构造
    }
 ```