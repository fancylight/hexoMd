---
title: 标准库源码-io
tags: 源码
categories: C
date: 2018-11-12 19:34:35
---

## c++ io概述
{% asset_img io.png %}
补充:
1. 关于postypes.h 定义的类型
{% asset_img fpos.png%}
```c++
#ifdef _GLIBCXX_HAVE_INT64_T_LONG
  typedef long          streamoff;  //表示相对的文件/流位置（距 fpos 的偏移），足以表示任何文件大小
#elif defined(_GLIBCXX_HAVE_INT64_T_LONG_LONG)
  typedef long long     streamoff;
#elif defined(_GLIBCXX_HAVE_INT64_T) 
 typedef ptrdiff_t	streamsize //ptrdiff_t 表示指针间距离=(addr1-addr2)/typeSize  如 (int* -int*)/4
 //streamsize 表示 I/O 操作中传输的字符数，或 I/O 缓冲区的大小 
 template<typename _StateT>
    class fpos //表示流或文件中的绝对位置 (filePosition?)
    {}
 //以下表示泛型特化   
  typedef fpos<mbstate_t> streampos;
  /// File position for wchar_t streams.
  typedef fpos<mbstate_t> wstreampos;
#if __cplusplus >= 201103L
  /// File position for char16_t streams.
  typedef fpos<mbstate_t> u16streampos;
  /// File position for char32_t streams.
  typedef fpos<mbstate_t> u32streampos;
  
  typedef fpos<mbstate_t> streampos;  //fpos表示的就是流的位置
  /// File position for wchar_t streams.
  typedef fpos<mbstate_t> wstreampos;
```
### local
{%asset_img locale.png locale.h%}
local是所有io类中都含有的一个成员属性,它完成了关于本地化的相关操作
{%codeblock lang:cpp locale.class %}
  typedef int	category;

    // Forward decls and friends:
    class facet;
    class id;
    class _Impl;
    
    friend class facet;  //它完成了各式的本地化操作
    friend class _Impl;
    //这三个实现在tcc中
    template<typename _Facet>
      friend bool
      has_facet(const locale&) throw();
    template<typename _Facet>
      friend const _Facet&
      use_facet(const locale&);
    template<typename _Cache>
      friend struct __use_cache;
  //这实际上就是平面的类型    
    static const category none		= 0;
    static const category ctype		= 1L << 0;
    static const category numeric	= 1L << 1;
    static const category collate	= 1L << 2;
    static const category time		= 1L << 3;
    static const category monetary	= 1L << 4;
    static const category messages	= 1L << 5;
    static const category all		= (ctype | numeric | collate |
					   time  | monetary | messages);
private: //私有属性                       
    // The (shared) implementation
    _Impl*		_M_impl;

    // The "C" reference locale
    static _Impl*       _S_classic;

    // Current global locale
    static _Impl*	_S_global;

    // Names of underlying locale categories.
    // NB: locale::global() has to know how to modify all the
    // underlying categories, not just the ones required by the C++
    // standard.
    static const char* const* const _S_categories;
{%endcodeblock%}

{%codeblock lang:cpp impl.class%}
  public:
    // Friends.
    friend class locale;
    friend class locale::facet;

    template<typename _Facet>
      friend bool
      has_facet(const locale&) throw();

    template<typename _Facet>
      friend const _Facet&
      use_facet(const locale&);

    template<typename _Cache>
      friend struct __use_cache;

  private:
    // Data Members.
    //关于facet /id的声明文件中,并不能看到有用的信息,总之facet是基类,id用来表示facet,impl则是locale具体行为的类,它持有了facet和id成员
    
    _Atomic_word			_M_refcount;
    const facet**			_M_facets;
    size_t				_M_facets_size;
    const facet**			_M_caches;
    char**				_M_names;
    static const locale::id* const	_S_id_ctype[];
    static const locale::id* const	_S_id_numeric[];
    static const locale::id* const	_S_id_collate[];
    static const locale::id* const	_S_id_time[];
    static const locale::id* const	_S_id_monetary[];
    static const locale::id* const	_S_id_messages[];
    static const locale::id* const* const _S_facet_categories[];
    
{%endcodeblock%}
{%codeblock lang:cpp collate:facet%}
  /**
   *  @brief  Facet for localized string comparison.  //该平面是用来比较字符串的
   *
   *  This facet encapsulates the code to compare strings in a localized
   *  manner.
   *
   *  The collate template uses protected virtual functions to provide
   *  the actual results.  The public accessors forward the call to
   *  the virtual functions.  These virtual functions are hooks for
   *  developers to implement the behavior they require from the
   *  collate facet.  //该类虚函数可以由子类决定实现
  */
template<typename _CharT>
    class _GLIBCXX_NAMESPACE_CXX11 collate : public locale::facet
    {  
  public:
    typedef _CharT			char_type;
      typedef basic_string<_CharT>	string_type;
      //比如:
       int
      compare(const _CharT* __lo1, const _CharT* __hi1,
	      const _CharT* __lo2, const _CharT* __hi2) const
      { return this->do_compare(__lo1, __hi1, __lo2, __hi2); }
      //虚
            virtual int
      do_compare(const _CharT* __lo1, const _CharT* __hi1,
		 const _CharT* __lo2, const _CharT* __hi2) const; //此处没有写成=0,tcc中也没有实现,不能确定是否库中没有实现
{%endcodeblock%}
{%codeblock lang:cpp num_put: public locale::facet%}
//该平面定义在locale_facets.中,起到了对于整形进行格式化,os的buf中的工作
 /**
   *  @brief  Primary class template num_put.
   *  @ingroup locales
   *
   *  This facet encapsulates the code to convert a number to a string.  It is
   *  used by the ostream numeric insertion operators. //当os进行numberic输出操作时,将数字转换为字符串
   *
   *  The num_put template uses protected virtual functions to provide the
   *  actual results.  The public accessors forward the call to the virtual
   *  functions.  These virtual functions are hooks for developers to
   *  implement the behavior they require from the num_put facet.  
  */
   template<typename _CharT, typename _OutIter>
    class num_put : public locale::facet
    {
    public:
    typedef _CharT		char_type;
    typedef _OutIter    iter_type; //buf迭代器,见下文
    //典型的函数
    /**
       *  @brief  Numeric formatting.
       *
       *  Formats the boolean @a v and inserts it into a stream.  It does so
       *  by calling num_put::do_put().
       *
       *  If ios_base::boolalpha is set, writes ctype<CharT>::truename() or
       *  ctype<CharT>::falsename().  Otherwise formats @a v as an int.
       *
       *  @param  __s  Stream to write to.
       *  @param  __io  Source of locale and flags.
       *  @param  __fill  Char_type to use for filling.
       *  @param  __v  Value to format and insert.
       *  @return  Iterator after writing.
      */
     iter_type
      put(iter_type __s, ios_base& __io, char_type __fill, bool __v) const
      { return this->do_put(__s, __io, __fill, __v); } //该函数实现也可以在locale_facet.h中找到
      //tcc实现
       template<typename _CharT, typename _OutIter>
    _OutIter
    num_put<_CharT, _OutIter>::
    do_put(iter_type __s, ios_base& __io, char_type __fill, bool __v) const //对bool类型进行格式化
    {
      const ios_base::fmtflags __flags = __io.flags();
      if ((__flags & ios_base::boolalpha) == 0) //当不是boolalpha控制符时
        {
          const long __l = __v;
          __s = _M_insert_int(__s, __io, __fill, __l); //调用这个插入
        }
      else
        {
	  typedef __numpunct_cache<_CharT>              __cache_type; //声明在locale_facets.h中
	  __use_cache<__cache_type> __uc;
	  const locale& __loc = __io._M_getloc();
	  const __cache_type* __lc = __uc(__loc);       //__uc(__loc)这是一个函数操作符,返回__numpunct_cache指针

	  const _CharT* __name = __v ? __lc->_M_truename
	                             : __lc->_M_falsename;  //改变bool为string类型的"true"/"false"
	  int __len = __v ? __lc->_M_truename_size
	                  : __lc->_M_falsename_size;

	  const streamsize __w = __io.width();
	  if (__w > static_cast<streamsize>(__len))
	    {
	      const streamsize __plen = __w - __len;
	      _CharT* __ps
		= static_cast<_CharT*>(__builtin_alloca(sizeof(_CharT)
							* __plen));

	      char_traits<_CharT>::assign(__ps, __plen, __fill);
	      __io.width(0);

	      if ((__flags & ios_base::adjustfield) == ios_base::left)
		{
		  __s = std::__write(__s, __name, __len);   //明显此函数才是进行buf插入的操作
		  __s = std::__write(__s, __ps, __plen);
		}
	      else
		{
		  __s = std::__write(__s, __ps, __plen);
		  __s = std::__write(__s, __name, __len);
		}
	      return __s;
	    }
	  __io.width(0);
	  __s = std::__write(__s, __name, __len);
	}
      return __s;
    }
    
    //插入整形
    template<typename _CharT, typename _OutIter>
    template<typename _ValueT>
      _OutIter
      num_put<_CharT, _OutIter>::
      _M_insert_int(_OutIter __s, ios_base& __io, _CharT __fill,
		    _ValueT __v) const
      {
      //获取一些必要信息
	using __gnu_cxx::__add_unsigned;
	typedef typename __add_unsigned<_ValueT>::__type __unsigned_type;
	typedef __numpunct_cache<_CharT>	             __cache_type;
	__use_cache<__cache_type> __uc;
	const locale& __loc = __io._M_getloc();
	const __cache_type* __lc = __uc(__loc);
	const _CharT* __lit = __lc->_M_atoms_out;
	const ios_base::fmtflags __flags = __io.flags();

	// Long enough to hold hex, dec, and octal representations.  由于要将num转换为字符串,并且要保证不同进制,进行了空间分配
	const int __ilen = 5 * sizeof(_ValueT);
	_CharT* __cs = static_cast<_CharT*>(__builtin_alloca(sizeof(_CharT)
							     * __ilen)); //用来容纳任意整形改变成不同进制字符的char/wchar_t*

	// [22.2.2.2.2] Stage 1, numeric conversion to character.
	// Result is returned right-justified in the buffer.
	const ios_base::fmtflags __basefield = __flags & ios_base::basefield;
	const bool __dec = (__basefield != ios_base::oct
			    && __basefield != ios_base::hex);
	const __unsigned_type __u = ((__v > 0 || !__dec)
				     ? __unsigned_type(__v)
				     : -__unsigned_type(__v));  //你按照有无符号转换截断原则就知道了,v<0&&__dec时,负数转换为同余正数
 	int __len = __int_to_char(__cs + __ilen, __u, __lit, __flags, __dec);//这个函数就是做生成进制字符串操作
	__cs += __ilen - __len;

	// Add grouping, if necessary.
	if (__lc->_M_use_grouping)
	  {
	    // Grouping can add (almost) as many separators as the number
	    // of digits + space is reserved for numeric base or sign.
	    _CharT* __cs2 = static_cast<_CharT*>(__builtin_alloca(sizeof(_CharT)
								  * (__len + 1)
								  * 2));
	    _M_group_int(__lc->_M_grouping, __lc->_M_grouping_size,
			 __lc->_M_thousands_sep, __io, __cs2 + 2, __cs, __len);
	    __cs = __cs2 + 2;
	  }

	// Complete Stage 1, prepend numeric base or sign.
	if (__builtin_expect(__dec, true))
	  {
	    // Decimal.
	    if (__v >= 0)
	      {
		if (bool(__flags & ios_base::showpos)
		    && __gnu_cxx::__numeric_traits<_ValueT>::__is_signed)
		  *--__cs = __lit[__num_base::_S_oplus], ++__len;
	      }
	    else
	      *--__cs = __lit[__num_base::_S_ominus], ++__len;
	  }
	else if (bool(__flags & ios_base::showbase) && __v)
	  {
	    if (__basefield == ios_base::oct)
	      *--__cs = __lit[__num_base::_S_odigits], ++__len;
	    else
	      {
		// 'x' or 'X'
		const bool __uppercase = __flags & ios_base::uppercase;
		*--__cs = __lit[__num_base::_S_ox + __uppercase];
		// '0'
		*--__cs = __lit[__num_base::_S_odigits];
		__len += 2;
	      }
	  }

	// Pad.
	const streamsize __w = __io.width();
	if (__w > static_cast<streamsize>(__len))
	  {
	    _CharT* __cs3 = static_cast<_CharT*>(__builtin_alloca(sizeof(_CharT)
								  * __w));
	    _M_pad(__fill, __w, __io, __cs3, __cs, __len);
	    __cs = __cs3;
	  }
	__io.width(0);

	// [22.2.2.2.2] Stage 4.
	// Write resulting, fully-formatted string to output iterator.
	return std::__write(__s, __cs, __len);  //最终写回还是这个函数
      }
    
    
     //在clion MinGW环境下,ide不会显示这些虚函数的定义和子类,文件路径mingw530_32\i686-w64-mingw32\include\c++\bits
    //std::__write的调用,
    template<typename _CharT>
    inline
    ostreambuf_iterator<_CharT>
    __write(ostreambuf_iterator<_CharT> __s, const _CharT* __ws, int __len)
    {
      __s._M_put(__ws, __len);  //这个函数实际调用了buf中的sputn函数
      return __s;
    }

  // This is the unspecialized form of the template. 非特化版本
  template<typename _CharT, typename _OutIter>
    inline
    _OutIter
    __write(_OutIter __s, const _CharT* __ws, int __len)  //逻辑上来看就是插入到buf当前空闲位置,然后将迭代器移动
    {
      for (int __j = 0; __j < __len; __j++, ++__s)
	*__s = __ws[__j];
      return __s;
    }
 //与o向对应的i操作,其基本读操作都是_M_extract_int类似这样的函数  
 //这是一部分
 {
  bool __negative = false;
	if (!__testeof)
	  {
	    __c = *__beg;
	    __negative = __c == __lit[__num_base::_S_iminus];
	    if ((__negative || __c == __lit[__num_base::_S_iplus])
		&& !(__lc->_M_use_grouping && __c == __lc->_M_thousands_sep)
		&& !(__c == __lc->_M_decimal_point))
	      {
		if (++__beg != __end)  //beg这个迭代器的操作就是对内部buf的操作
		  __c = *__beg;
		else
		  __testeof = true;
	      }
	  }
 }

{%endcodeblock%}
### ios_bsae
定义:管理格式化标准以及相关异常
声明在 ios_base.h中 iosfwd.h声明了向前泛型声明和类型别名
{% asset_img ios_base.png%}
1. fmtflag 
//1.fmtflags
常量	解释  
dec	为整数 I/O 使用十进制底：见 std::dec  
oct	为整数 I/O 使用八进制底：见 std::oct  
hex	为整数 I/O 使用十六进制底：见 std::hex  
basefield	dec|oct|hex 。适用于掩码运算  
left	左校正（添加填充字符到右）：见 std::left 
right	右校正（添加填充字符到左）：见 std::right  
internal	内部校正（添加填充字符到内部选定点）：见 std::internal  
adjustfield	left|right|internal 。适用于掩码运算  
scientific	用科学记数法生成浮点类型，或若与 fixed 组合则用十六进制记法：见 std::scientific  
fixed	用定点记法生成浮点类型，或若与 scientific 组合则用十六进制记法：见 std::fixed  
floatfield	scientific|fixed 。适用于掩码运算  
boolalpha	以字母数字格式插入并释出 bool 类型：见 std::boolalpha  
showbase	生成为整数输出指示数字基底的前缀，货币 I/O 中要求现金指示器：见 std::showbase 
showpoint	无条件为浮点数输出生成小数点字符：见 std::showpoint  
showpos	为非负数值输出生成 + 字符，见 std::showpos  
skipws	在具体输入操作前跳过前导空白符：见 std::skipws  
unitbuf	在每次输出操作后冲入输出：见 std::unitbuf  
uppercase	在具体输出的输出操作中以大写等价替换小写字符：见 std::uppercase  
```c++
void myIoTest::ios_base_test::flagEnumTest() {
    //以hex输出
    cout.setf(ios_base::hex, ios::basefield);
    cout << 15 << "\n";
    //left | right :修改填充字符的默认定位。 left 与 right 应用到任何输出，而 internal 应用到整数、浮点和货币输出。在输入时无效果。
    cout << "Left fill:\n" << std::left << setfill('*') //os.setFill('*')
         << setw(12) << -1.23 << '\n'
         << setw(12) << hex << showbase << 42 << '\n'
         << setw(12) << put_money(123, true) << "\n\n";
    //boolalpha 将缓冲使用bool量解释
    cout << boolalpha << true << "\n" << false << "\n";
    istringstream istringstream1("true false");
    bool b1, b2;
    istringstream1 >> boolalpha >> b1 >> b2;
    cout << dec << noboolalpha << "true false is parse" << " " << b1 << " " << b2;
}
//1. os<< 一般来说 <<都是重写的,而且都不是成员操作符,也就是接受两边值,同时返回一个值
//2. setw=width()
```
2. std::ios_base::openmode  定义在ios_base.h中的一个enum
常量	解释
app	每次写入前寻位到流结尾  
binary	以二进制模式打开  
in	为读打开  
out	为写打开  
trunc	在打开时舍弃流的内容  
ate	打开后立即寻位到流结尾  
```c++
//例子:
void myIoTest::ios_base_test::openModeTest() {
    ifstream ifstream1;
    const char* path;
    ifstream1.open(path);  
}
//源码:
//fstream.h
class class basic_ifstream : public basic_istream<_CharT, _Traits> {
        typedef basic_filebuf<char_type, traits_type> __filebuf_type; //basic_filebuf也是定义在该文件中的一个类,和ifstream 是has a关系
  private:
        __filebuf_type _M_filebuf;
        
        void open(const char *__s, ios_base::openmode __mode = ios_base::in) {
            if (!_M_filebuf.open(__s, __mode | ios_base::in)) //实际上open都是basic_filebuf.open(),底层实现看不到
                this->setstate(ios_base::failbit);
            else
                this->clear();
        }
        
```
3. iostate 依旧定义在ios_bsae.h中的enum
常量	解释  
goodbit	无错误  
badbit	不可恢复的流错误  
failbit	输入/输出操作失败（格式化或提取错误）  
eofbit	关联的输出序列已抵达文件尾  
```c++
//eofbit
1. string 输入函数 std::getline ，若它以抵达流结尾，而非抵达指定的终止字符完成。
2. basic_istream::operator>> 的数值输入重载，若在 num_get::get 处理的阶段 2 ，读取下个字符时遇到流结尾。取决于分析
状态，可能或可能不同时设置 failbit ：例如 int n; istringstream buf("1"); buf >> n; 设置 eofbit ，但不设置 
failbit ：成功分析整数 1 并存储之于 n 。另一方面， bool b; istringstream buf("tr"); buf >> boolalpha >> b; 一同
设置 eofbit 和 failbit ：无足够的字符完成布尔 true 的分析。
3. operator>>std::basic_istream 的字符释出重载，若在释出字符数量上的限制（若存在）前抵达流结尾。
```

    可查阅[iostate](https://zh.cppreference.com/w/cpp/io/ios_base/iostate)
4. seekdir ios_base.h中enum
beg	流的开始
end	流的结尾
cur	流位置指示器的当前位置
5. 内部定义函数
```c++
//1.flags
fmtflags flags() const;(1)	
fmtflags flags( fmtflags flags );(2)
管理格式化标志。
1) 返回当前格式化设置。
2) 以给定者替换当前设置。
//2.setf
fmtflags setf( fmtflags flags );(1)	
fmtflags setf( fmtflags flags, fmtflags mask );(2)
设置格式化标志以指定设置。
1) 设置 flags 所标识的格式化标志。等效地进行下列操作： fl = fl | flags ，其中 fl 定义内部格式化标志的状态。
2) 清除 mask 下的格式化标志，并设置被清除的标志为 flags 所指定者。等效地进行下列操作： fl = (fl & ~mask) | (flags & mask) ，其中 fl 定义格式化标志的内部状态。
//源码:
  fmtflags
    setf(fmtflags __fmtfl)
    {
      fmtflags __old = _M_flags;
      _M_flags |= __fmtfl;  //添加 __fmtfl到现有fmt中
      return __old;
    }
    fmtflags
    setf(fmtflags __fmtfl, fmtflags __mask)
    {
      fmtflags __old = _M_flags;
      _M_flags &= ~__mask;   //清除fmtflags __mask
      _M_flags |= (__fmtfl & __mask);  //将__fmtfl添加
      return __old;
    }
//3.unsetf
void unsetf( fmtflags flags ); //清除对应的fmt
//4.precision
streamsize precision() const;(1)	
streamsize precision( streamsize new_precision );(2)	
管理 std::num_put::do_put 所进行的浮点输出精度（即生成多少数位）。
1) 返回当前精度。
2) 设置精度为给定值。
std::basic_ios::init 所建立的默认精度为 6 。
void myIoTest::ios_base_test::precisionTest() {
    const double d=2.234234324;
    cout<<setprecision(3)<<d;
    //setprecision 并非定义在ios_base.h,而是iomanip.h
 //源码: iomanip.h
 struct _Setprecision { int _M_n; }; //结构体  
  inline _Setprecision setprecision(int __n){  //内联
  return { __n }; 
  }
 template<typename _CharT, typename _Traits>
    inline basic_ostream<_CharT, _Traits>& 
    operator<<(basic_ostream<_CharT, _Traits>& __os, _Setprecision __f) //接受_Setprecision的操作符
    { 
      __os.precision(__f._M_n); 
      return __os; 
    }
  //这里理解一下inline的作用,貌似就是和宏定义相同,并非是通过函数调用,也就是说此处的汇编代码就是执行内联函数体直接展开,那么编译器也做了优化,并没有执行return rax.
//5.width
streamsize width() const;(1)	
streamsize width( streamsize new_width );(2)	
管理某些输出操作时生成的最小字符数，和某些输出操作时生成的最大字符数。
1) 返回当前域宽。
2) 设置域宽为给定值 仅仅下一次io函数生效
//setw
用于表达式 out << setw(n) 或 in >> setw(n) 时，设置流 out 或 in 的 width 参数准确为 n 。
参数
n	-	width 的新值
返回值
返回未指定类型对象，满足若 str 是 std::basic_ostream<CharT, Traits> 或 std::basic_istream<CharT, Traits> 类型流的名称，则表达式 str << setw(n) 或 str >> setw(n) 表现为如同执行下列代码：
str.width(n);
//此处执行方式和prcstion相同,都是iomanip定义的
}
//6.std::ios_base::register_callback
注册将为 imbue() 、 std::basic_ios::copyfmt() 和 ~ios_base() 调用的用户定义函数。每次都调用每个注册的回调：事件类型（ event 类型值）作为首参数传递，而且可用于区别调用方。
//当调用 imbue() 这几个函数时,才会触发回调
//https://zh.cppreference.com/w/cpp/io/ios_base/register_callback 此处有一段实例
//7. xalloc 返回能安全用作 pword() 和 iword() 下标的程序范围内独有的整数
//8.iword/pword
源码:
 // 27.4.2.5  Members for iword/pword storage
    struct _Words
    {
      void*	_M_pword;
      long	_M_iword;
      _Words() : _M_pword(0), _M_iword(0) { }
    };
    // Allocated storage.
    int			_M_word_size;
    _Words*		_M_word;
 //返回值可以用来查看,也可以用来修改
  long&
    iword(int __ix) //
    {
      _Words& __word = (__ix < _M_word_size)
			? _M_word[__ix] : _M_grow_words(__ix, true);
      return __word._M_iword;
    }
//9.
imbue 
设置本地环境 
(公开成员函数)
getloc 
返回当前本地环境 
(公开成员函数)
```
6. 补充
```c++
//内部回调结构
struct _Callback_list
    {
      // Data Members
      _Callback_list*		_M_next;  //下一个回调节点
      ios_base::event_callback	_M_fn;//该回调对应的类型 
      int			_M_index; //下标
      _Atomic_word		_M_refcount;  // 0 means one reference.

      _Callback_list(ios_base::event_callback __fn, int __index,
		     _Callback_list* __cb)
      : _M_next(__cb), _M_fn(__fn), _M_index(__index), _M_refcount(0) { }

      void
      _M_add_reference() { __gnu_cxx::__atomic_add_dispatch(&_M_refcount, 1); }

      // 0 => OK to delete.
      int
      _M_remove_reference() 
      {
        // Be race-detector-friendly.  For more info see bits/c++config.
        _GLIBCXX_SYNCHRONIZATION_HAPPENS_BEFORE(&_M_refcount);
        int __res = __gnu_cxx::__exchange_and_add_dispatch(&_M_refcount, -1);
        if (__res == 0)
          {
            _GLIBCXX_SYNCHRONIZATION_HAPPENS_AFTER(&_M_refcount);
          }
        return __res;
      }
    };
```
### basic_ios(ios):public ios_bsae
1. 特征_Traits
```c++
//iosfwd.h
`template<typename _CharT, typename _Traits = char_traits<_CharT> >
    class basic_ios`
//stringfwd.h
  struct char_traits;
  template<> struct char_traits<char>;
#ifdef _GLIBCXX_USE_WCHAR_T
  template<> struct char_traits<wchar_t>;
#endif
#if ((__cplusplus >= 201103L) \
     && defined(_GLIBCXX_USE_C99_STDINT_TR1))
  template<> struct char_traits<char16_t>;
  template<> struct char_traits<char32_t>;
  //也就是实际上使用那个特征模板由_CharT决定, 如char 和wchar_t
  
//char_traits.h 定义char类型 特化模板
template<> //截取关于char的特化一部分
    struct char_traits<char>
    {
      typedef char              char_type;
      typedef int               int_type;
      typedef streampos         pos_type;  // typedef fpos<mbstate_t> streampos;(postype.h中定义)
      typedef streamoff         off_type; 
      typedef mbstate_t         state_type; //int
 //后边都是关系的static函数,其他特化也是如此
      static void
      assign(char_type& __c1, const char_type& __c2) _GLIBCXX_NOEXCEPT
      { __c1 = __c2; }

      static _GLIBCXX_CONSTEXPR bool
      eq(const char_type& __c1, const char_type& __c2) _GLIBCXX_NOEXCEPT
      { return __c1 == __c2; }

      static _GLIBCXX_CONSTEXPR bool
      lt(const char_type& __c1, const char_type& __c2) _GLIBCXX_NOEXCEPT
      {
	// LWG 467.
	return (static_cast<unsigned char>(__c1)
		< static_cast<unsigned char>(__c2));
      }

      static int
      compare(const char_type* __s1, const char_type* __s2, size_t __n)
      {
	if (__n == 0)
	  return 0;
	return __builtin_memcmp(__s1, __s2, __n);
      }
```
2. 部分属性
```c++
template<typename _CharT, typename _Traits>
    class basic_ios : public ios_base
    {
      public:
      //结合不同的模板如_CharT=char,_Traits= struct char_traits<char>
      typedef _CharT                                 char_type; //char
      typedef typename _Traits::int_type             int_type;  //int
      typedef typename _Traits::pos_type             pos_type;//fpos<int>
      typedef typename _Traits::off_type             off_type; //streamoff(long long)
      typedef _Traits                                traits_type;

      //都是facet的子类,用于local
      typedef ctype<_CharT>                          __ctype_type; 
      typedef num_put<_CharT, ostreambuf_iterator<_CharT, _Traits> >
						     __num_put_type;
      typedef num_get<_CharT, istreambuf_iterator<_CharT, _Traits> >
						     __num_get_type;
      // Data members:
    protected:
      basic_ostream<_CharT, _Traits>*                _M_tie; //绑定的流指针
      mutable char_type                              _M_fill;//填充的字符
      mutable bool                                   _M_fill_init;//是否填充
      basic_streambuf<_CharT, _Traits>*              _M_streambuf;//内部缓冲

      // Cached use_facet<ctype>, which is based on the current locale info.
      const __ctype_type*                            _M_ctype;
      // For ostream.
      const __num_put_type*                          _M_num_put;  //对于输出流<<操作符属于num的进行格式化为 _CharT类型,并插入到buf中
      // For istream.
      const __num_get_type*                          _M_num_get;
```
3. 函数
```c++
//1. good eod fail bad operator! operator bool  都是检查状态
//源码
#if __cplusplus >= 201103L
      explicit operator bool() const //这里就是一个类型转换符,并且使用了explicit阻止了自动转换,要通过static<bool>cast()
      { return !this->fail(); }
------------      
     bool
      fail() const
      { return (this->rdstate() & (badbit | failbit)) != 0; }
//2.rdstate setstate clear 控制ios_base::_M_streambuf_state
//3.copyfmt 
basic_ios& copyfmt(const basic_ios& other);
//源码
template<typename _CharT, typename _Traits>
    basic_ios<_CharT, _Traits>&
    basic_ios<_CharT, _Traits>::copyfmt(const basic_ios& __rhs)
    {
      // _GLIBCXX_RESOLVE_LIB_DEFECTS
      // 292. effects of a.copyfmt (a)
      if (this != &__rhs)
	{
	  // Per 27.1.1, do not call imbue, yet must trash all caches
	  // associated with imbue()

	  // Alloc any new word array first, so if it fails we have "rollback".
      //erase_evnet事件传递,触发该流的erase_event回调函数
	  _Words* __words = (__rhs._M_word_size <= _S_local_word_size) ?
	                     _M_local_word : new _Words[__rhs._M_word_size];

	  // Bump refs before doing callbacks, for safety.
	  _Callback_list* __cb = __rhs._M_callbacks;
	  if (__cb)
	    __cb->_M_add_reference();
	  _M_call_callbacks(erase_event);
	  if (_M_word != _M_local_word)
	    {
	      delete [] _M_word;
	      _M_word = 0;
	    }
	  _M_dispose_callbacks();
      //拷贝other的所有信息
	  // NB: Don't want any added during above.
	  _M_callbacks = __cb;
	  for (int __i = 0; __i < __rhs._M_word_size; ++__i)
	    __words[__i] = __rhs._M_word[__i];
	  _M_word = __words;
	  _M_word_size = __rhs._M_word_size;

	  this->flags(__rhs.flags());
	  this->width(__rhs.width());
	  this->precision(__rhs.precision());
	  this->tie(__rhs.tie());
	  this->fill(__rhs.fill());
	  _M_ios_locale = __rhs.getloc();
	  _M_cache_locale(_M_ios_locale);
       //调用 copyfmt_even 事件回调函数
	  _M_call_callbacks(copyfmt_event);
       //拷贝异常状态 
	  // The next is required to be the last assignment.
	  this->exceptions(__rhs.exceptions());
	}
      return *this;
    }
 //4.fill
    char_type
      fill() const
      {
	if (!_M_fill_init)
	  {
	    _M_fill = this->widen(' ');
	    _M_fill_init = true;
	  }
	return _M_fill;
      }
      //实际上调用了_M_ctype的widen
         char_type
      widen(char __c) const
      { return __check_facet(_M_ctype).widen(__c); }
 //5.imube 实现
   template<typename _CharT, typename _Traits>
    locale
    basic_ios<_CharT, _Traits>::imbue(const locale& __loc)
    {
      locale __old(this->getloc());
      ios_base::imbue(__loc); //调用了父类实现的
      _M_cache_locale(__loc); 
      if (this->rdbuf() != 0)
	this->rdbuf()->pubimbue(__loc);//改变内部buf
      return __old;
    }
  //6.tie  
std::basic_ostream<CharT,Traits>* tie() const;
(1)	
std::basic_ostream<CharT,Traits>* tie( std::basic_ostream<CharT,Traits>* str );
(2)	
管理联系流。联系流是输出流，它与流缓冲（ rdbuf() ）所控制的输出序列同步，即在任何 *this 上的输入/输出操作前，在联系流上调用 flush() 。

1) 返回当前联系流。若无联系流，则返回空指针。
2) 设置当前联系流为 str 。返回操作前的联系流。若无联系流，则返回空指针。

int main()
{
    std::ofstream os("test.txt");
    std::ifstream is("test.txt");
    std::string value("0");
 
    os << "Hello";
    is >> value;
 
    std::cout << "Result before tie(): \"" << value << "\"\n";
    is.clear();
    is.tie(&os);
  
    is >> value;
 
    std::cout << "Result after tie(): \"" << value << "\"\n";
}
//将is.tie(os),当is>>时,os调用flush,此时文本中才有值
//Result before tie(): "0"
//Result after tie(): "Hello"

//narrow 
窄化字符 
widen 
拓宽字符 
init 
//模板tcc 实现,该函数会被子类构造器调用
  template<typename _CharT, typename _Traits>
    void
    basic_ios<_CharT, _Traits>::init(basic_streambuf<_CharT, _Traits>* __sb)
    {
      // NB: This may be called more than once on the same object.
      ios_base::_M_init();

      // Cache locale data and specific facets used by iostreams.
      _M_cache_locale(_M_ios_locale);

      // NB: The 27.4.4.1 Postconditions Table specifies requirements
      // after basic_ios::init() has been called. As part of this,
      // fill() must return widen(' ') any time after init() has been
      // called, which needs an imbued ctype facet of char_type to
      // return without throwing an exception. Unfortunately,
      // ctype<char_type> is not necessarily a required facet, so
      // streams with char_type != [char, wchar_t] will not have it by
      // default. Because of this, the correct value for _M_fill is
      // constructed on the first call of fill(). That way,
      // unformatted input and output with non-required basic_ios
      // instantiations is possible even without imbuing the expected
      // ctype<char_type> facet.
      _M_fill = _CharT();
      _M_fill_init = false;

      _M_tie = 0;
      _M_exception = goodbit;
      _M_streambuf = __sb;  //改变子类的缓冲
      _M_streambuf_state = __sb ? goodbit : badbit;
    }
初始化一个默认构造的std::basic_ios 
move  (C++11) 
从另一 std::basic_ios 移动，除了 rdbuf 
swap  (C++11) 
与另一 std::basic_ios 交换，除了 rdbuf 
set_rdbuf 
替换 rdbuf 而不清除其错误状态 
```
### basic_streambuf
1. 简单构成
{%asset_img streambuf0.png 底层数据%}
cppref中的一部分解释:
- buf维护了两个缓冲区,结构类似于vector这样的指针,对于输入in来说称为获取区,对于out来说称为放置区
- 图中的associated character sequence,关联字符序列，又称作源（对于输入）或池（对于输出）。它可以是通过 OS API 访问的实体（文件、 TCP 接头、串行端口、其他字符设备），或者可以是能转译成字符源或池的对象（ std::vector 、数组、字符串字面量）。也就是说源或者池分别指的外部io和api内部的对象.
- basic_streambuf 对象可支持输入（该情况下起始、下一位置和终止指针所描述的区域被称为获取区）、输出（放置区），或同时输入与输出。在最后一种情况下，跟踪六个指针，它们可能全部指向同一数组的元素，或指向二个单独数组的元素。
- 若放置区中下一位置指针小于终止指针，则写位置可用。下一位置指针可被解引用和赋值
- 若获取区中下一位置指针小于终止指针，则读位置可用。下一位置指针可被解引用和读取。
- 若获取区中下一位置指针大于起始指针，则回放位置可用，而下一位置指针可以被自减并赋值，以将字符放回到获取区。
- 受控制序列中的字符表示和编码可以异于关联序列中的字符表示，该情况下典型地用 std::codecvt 本地环境进行转换。常见的例子是通过 std::wfstream 对象访问 UTF-8 （或其他多字节编码）文件：受控制字符序列由 wchar_t 字符组成，但关联序列由字节组成。
- 具体的缓冲类，如 std::basic_filebuf 或 std::basic_stringbuf 导出自 std::basic_streambuf 。
{%codeblock lang:cpp basic_streambuf.h%}
//类型
      typedef _CharT 					char_type;
      typedef _Traits 					traits_type;
      typedef typename traits_type::int_type 		int_type;
      typedef typename traits_type::pos_type 		pos_type;
      typedef typename traits_type::off_type 		off_type;
//友元
     friend class basic_ios<char_type, traits_type>;
      friend class basic_istream<char_type, traits_type>;
      friend class basic_ostream<char_type, traits_type>;
      friend class istreambuf_iterator<char_type, traits_type>;
      friend class ostreambuf_iterator<char_type, traits_type>;
//友元函数,仅仅截取了一个
//该函数被定义在stringbuf.tcc 而不是basic_streambuf.tcc中
template<typename _CharT2, typename _Traits2, typename _Alloc>
        friend basic_istream<_CharT2, _Traits2>&
        getline(basic_istream<_CharT2, _Traits2>&,
		basic_string<_CharT2, _Traits2, _Alloc>&, _CharT2);        
//关键成员
//内部的io缓冲,分配方式和vector这种扩容数组很相似
  char_type* 		_M_in_beg;     ///< Start of get area.
      char_type* 		_M_in_cur;     ///< Current read area.
      char_type* 		_M_in_end;     ///< End of get area.
      char_type* 		_M_out_beg;    ///< Start of put area.
      char_type* 		_M_out_cur;    ///< Current put area.
      char_type* 		_M_out_end;    ///< End of put area.
      
      /// Current locale setting.
      locale 			_M_buf_locale;
      //公开的函数
      getloc() const //获取_M_buf_locale
      pubimbue(const locale& __loc) //设置_M_buf_locale
      //实际调用虚函数
       basic_streambuf* pubsetbuf(char_type* __s, streamsize __n){  //设置buf为__s
       return this->setbuf(__s, __n);
       }
//       ------------------------------
//寻位
        pos_type
      pubseekoff(off_type __off, ios_base::seekdir __way,    //改变buf位置为__way(ios_base中定义)
		 ios_base::openmode __mode = ios_base::in | ios_base::out) 
      { return this->seekoff(__off, __way, __mode); }
      
       pos_type
      pubseekpos(pos_type __sp,
		 ios_base::openmode __mode = ios_base::in | ios_base::out) //通过sp来改变位置 pos_type是fpos类型
      { return this->seekpos(__sp, __mode); }
      
      int
      pubsync() { return this->sync(); } //从名字上看是于流关系的io设备进行同步,刷新??
//----------------------------------
//获取区函数
       streamsize
      in_avail() //当前获取区剩余
      {
	const streamsize __ret = this->egptr() - this->gptr(); //end-cur==0?__ret:this->showmanyc();
	return __ret ? __ret : this->showmanyc(); 
      }
      
       int_type
      snextc() //读取下一个字符
      {
	int_type __ret = traits_type::eof();
	if (__builtin_expect(!traits_type::eq_int_type(this->sbumpc(),
						       __ret), true))
	  __ret = this->sgetc();
	return __ret;
      }
      //读取当前数据,并指针前进
      int_type
      sbumpc()  
      {
	int_type __ret;
	if (__builtin_expect(this->gptr() < this->egptr(), true)) //非空
	  {
	    __ret = traits_type::to_int_type(*this->gptr()); //读取并转为int_type
	    this->gbump(1); //前进
	  }
	else
	  __ret = this->uflow(); //空则调用uflow
	return __ret;
      }
      
       int_type //读取当前的数据
      sgetc()
      {
	int_type __ret;
	if (__builtin_expect(this->gptr() < this->egptr(), true))//非空
	  __ret = traits_type::to_int_type(*this->gptr()); //读取当前获取区值
	else
	  __ret = this->underflow(); //若当前指针指向end,则进行underflow操作,并返回读取后的数据
	return __ret;
      }
      //该函数子类可以实现
      virtual int_type
      uflow()
      {
	int_type __ret = traits_type::eof();
	const bool __testeof = traits_type::eq_int_type(this->underflow(), //调用应该实现的underflow,该函数应该到io设备中读取数据,放回缓冲区,并且返回此时扩容的缓冲第一个指针,也就是之前__builtin_expect(this->gptr() < this->egptr(), true)的情况时,current_prt指针指向了没有数据的位置,若此次有数据,那么说明io设备还能读取,否则返回的一定是eof;然后此时指针指向了扩容部分的第二个位置,也就是说cpp_zh的那张图是正确的,current位置维持着是将要读取的位置
							__ret);
	if (!__testeof)
	  {
	    __ret = traits_type::to_int_type(*this->gptr());
	    this->gbump(1);  //这两个句等同于sbumpc
	  }
	return __ret;
      }
      
     streamsize
      sgetn(char_type* __s, streamsize __n)
      { return this->xsgetn(__s, __n); }
      
      //定义在tcc中
  template<typename _CharT, typename _Traits>
    streamsize
    basic_streambuf<_CharT, _Traits>::
    xsgetn(char_type* __s, streamsize __n)
    {
      streamsize __ret = 0;
      while (__ret < __n)
	{
	  const streamsize __buf_len = this->egptr() - this->gptr();  
	  if (__buf_len)
	    {
	      const streamsize __remaining = __n - __ret;
	      const streamsize __len = std::min(__buf_len, __remaining);  //获取n和剩余最大值
	      traits_type::copy(__s, this->gptr(), __len);//拷贝
	      __ret += __len; //读取了多少
	      __s += __len;//移动_s指针
	      this->__safe_gbump(__len);//移动缓冲指针
	    }

	  if (__ret < __n)
	    {//说明缓冲区已经满了
	      const int_type __c = this->uflow();  //扩容,返回当前指针数据,并且缓冲区指针向前移动
	      if (!traits_type::eq_int_type(__c, traits_type::eof()))
		{
		  traits_type::assign(*__s++, traits_type::to_char_type(__c)); //因为使用了uflow,因此要把这个数据返回
		  ++__ret; //增加ret
		}
	      else
		break;
	    }
	}
      return __ret; //返回
    }
// ----------------------------------------    
// 回放
 int_type //将__c放到反位置
      sputbackc(char_type __c)
      {
	int_type __ret;
	const bool __testpos = this->eback() < this->gptr();
	if (__builtin_expect(!__testpos ||
			     !traits_type::eq(__c, this->gptr()[-1]), false)) gptr() > eback()||而字符 c 不等于 gptr() 左一位置的字符
	  __ret = this->pbackfail(traits_type::to_int_type(__c)); //虚函数 子类实现,若不能移动则返回eof
	else
	  {
      //移动,并读取,返回,相同不用插入
	    this->gbump(-1); 
	    __ret = traits_type::to_int_type(*this->gptr());
	  }
	return __ret;
      }
     //例子:
     static void testSputbackc(){
            std::stringstream s("abcdef"); // gptr() 指向 "abcdef" 中的 'a'
            std::cout << "Before putback, string holds " << s.str() << '\n';
            char c1 = s.get(); // c1 = 'a' ， gptr() 现在指向 "abcdef" 中的 'b'
            char c2 = s.rdbuf()->sputbackc('z'); // 同 s.putback('z')
            // gptr() 现在指向 "zbcdef" 中的 'z'
            std::cout << "After putback, string holds " << s.str() << '\n';
            char c3 = s.get(); // c3 = 'z' ， gptr() 现在指向 "zbcdef" 中的 'b'
            char c4 = s.get(); // c4 = 'b' ， gptr() 现在指向 "zbcdef" 中的 'c'
            std::cout << c1 << c2 << c3 << c4 << '\n';

            s.rdbuf()->sputbackc('b');  // gptr() 现在指向 "zbcdef" 中的 'b'
            s.rdbuf()->sputbackc('z');  // gptr() 现在指向 "zbcdef" 中的 'z'
            int eof = s.rdbuf()->sputbackc('x');  // 无内容能反获取： pbackfail() 失败
            if (eof == EOF)
                std::cout << "No room to putback after 'z'\n";
        }
        //输出:
        //Before putback, string holds abcdef
        //After putback, string holds zbcdef
        //azzb
        //No room to putback after 'z'
  
    int_type
      sungetc()//取回反位置,若不能移动则返回eof
      {
	int_type __ret;
	if (__builtin_expect(this->eback() < this->gptr(), true))
	  {
	    this->gbump(-1);
	    __ret = traits_type::to_int_type(*this->gptr());
	  }
	else
	  __ret = this->pbackfail();
	return __ret;
      }
      //例
      void myIoTest::bufTest::testSunget() {
    std::stringstream s("abcdef"); // gptr() 指向 'a'
    char c1 = s.get(); // c = 'a', gptr() 现在指向 'b'
    char c2 = s.rdbuf()->sungetc(); // 同 s.unget() ： gptr() 又指向 'a'
    char c3 = s.get(); // c3 = 'a' ， gptr() 现在指向 'b'
    char c4 = s.get(); // c4 = 'b' ， gptr() 现在指向 'c'
    std::cout << c1 << c2 << c3 << c4 << '\n';

    s.rdbuf()->sungetc();  // 回到 'b'
    s.rdbuf()->sungetc();  // 回到 'a'
    int eof = s.rdbuf()->sungetc();  // 无内容可反获取： pbackfail() 失败
    if (eof == EOF)
        std::cout << "Nothing to unget after 'a'\n";}
----------------------------------
//放置区
  int_type
      sputc(char_type __c)
      {
	int_type __ret;
	if (__builtin_expect(this->pptr() < this->epptr(), true))
	  {
	    *this->pptr() = __c;
	    this->pbump(1);
	    __ret = traits_type::to_int_type(__c);
	  }
	else
	  __ret = this->overflow(traits_type::to_int_type(__c));
	return __ret;
      }
      sputn()//放置数组,对比于sgetn,调用虚函数xsputn
//tcc中实现
 template<typename _CharT, typename _Traits>
    streamsize
    basic_streambuf<_CharT, _Traits>::
    xsputn(const char_type* __s, streamsize __n)
    {
      streamsize __ret = 0;
      while (__ret < __n)
	{
	  const streamsize __buf_len = this->epptr() - this->pptr();
	  if (__buf_len)
	    {
	      const streamsize __remaining = __n - __ret;
	      const streamsize __len = std::min(__buf_len, __remaining);
	      traits_type::copy(this->pptr(), __s, __len);
	      __ret += __len;
	      __s += __len;
	      this->__safe_pbump(__len);  //移动缓冲区
	    }

	  if (__ret < __n)
	    {
	      int_type __c = this->overflow(traits_type::to_int_type(*__s));  //调用虚函数,对设备进行写操作,子类实现
	      if (!traits_type::eq_int_type(__c, traits_type::eof()))
		{
		  ++__ret;
		  ++__s;
		}
	      else
		break;
	    }
	}
      return __ret;
    }
 //总结来看,buf就是起到缓冲的作用,和java不同? c++io自带缓冲,in-->underflow,out-->overflow,分别对输入/输出序列进行操作.   
 //结合下文的it,本文的buf,以及上文的facet,实际上对于字符的控制,如格式化,本地化,都是和iostream有着has a关系的facet完成,并且在完成格式化操作后,由facet的api调用迭代器,实际上就是调用对应iostream的buf, i:将设备读取过来的字节转换成对应的类型,流入用户内存中,根据条件判断是否应该在此次i操作对底层设置进行读访问;o:将用户内存中的各种类型进行正确的格式化后输出刀缓冲,根据各种判定条件触发是否将缓冲刷入o设备.
{%endcodeblock%}
2.密切相关的类迭代器
{%codeblock lang:cpp ostreambuf_iterator%}
//定义在steambuf_iterator.h中,并没有tcc
 template<typename _CharT, typename _Traits>
    class ostreambuf_iterator
    : public iterator<output_iterator_tag, void, void, void, void>
    {
    public:
      // Types:
      //@{
      /// Public typedefs
      typedef _CharT                           char_type;
      typedef _Traits                          traits_type;
      typedef basic_streambuf<_CharT, _Traits> streambuf_type;
      typedef basic_ostream<_CharT, _Traits>   ostream_type;
      //@}

      template<typename _CharT2>
	friend typename __gnu_cxx::__enable_if<__is_char<_CharT2>::__value,
		                    ostreambuf_iterator<_CharT2> >::__type
	copy(istreambuf_iterator<_CharT2>, istreambuf_iterator<_CharT2>,
	     ostreambuf_iterator<_CharT2>);

    private:
      streambuf_type*	_M_sbuf;
      bool		_M_failed;

    public:
      ///  Construct output iterator from ostream.
      ostreambuf_iterator(ostream_type& __s) _GLIBCXX_USE_NOEXCEPT
      : _M_sbuf(__s.rdbuf()), _M_failed(!_M_sbuf) { }  //这两个函数都可以进行隐式构造

      ///  Construct output iterator from streambuf.
      ostreambuf_iterator(streambuf_type* __s) _GLIBCXX_USE_NOEXCEPT
      : _M_sbuf(__s), _M_failed(!_M_sbuf) { }

      ///  Write character to streambuf.  Calls streambuf.sputc().
  //这个=符号才是真正做操作的函数,具体看std::__write(_OutIter __s, const _CharT* __ws, int __len),定义在locale_facet.h中   
      ostreambuf_iterator&
      operator=(_CharT __c)   
      {
	if (!_M_failed &&
	    _Traits::eq_int_type(_M_sbuf->sputc(__c), _Traits::eof()))//实际上调用了buf.sputc函数
	  _M_failed = true;
	return *this;
      }
 //这个看出来了,对于osbuf_iter,重写的++ 没做什么事情
      /// Return *this.
      ostreambuf_iterator&
      operator*()
      { return *this; }

      /// Return *this.
      ostreambuf_iterator&
      operator++(int)
      { return *this; }

      /// Return *this.
      ostreambuf_iterator&
      operator++()
      { return *this; }

      /// Return true if previous operator=() failed.
      bool
      failed() const _GLIBCXX_USE_NOEXCEPT
      { return _M_failed; }

      ostreambuf_iterator&
      _M_put(const _CharT* __ws, streamsize __len)
      {
	if (__builtin_expect(!_M_failed, true)
	    && __builtin_expect(this->_M_sbuf->sputn(__ws, __len) != __len,
				false))
	  _M_failed = true;
	return *this;
      }
    };
    
{%endcodeblock%}
### basic_ostream(ostream)
1. 基本
{%asset_img basic_ostream.png ostream构成%}
2. 代码
{%codeblock lang:cpp basic_ostream.h%}
class basic_ostream : virtual public basic_ios<_CharT, _Traits> //虚继承
  public:
  //type
      // Types (inherited from basic_ios):
      typedef _CharT			 		char_type;
      typedef typename _Traits::int_type 		int_type;
      typedef typename _Traits::pos_type 		pos_type;
      typedef typename _Traits::off_type 		off_type;
      typedef _Traits			 		traits_type;

      // Non-standard Types:
      typedef basic_streambuf<_CharT, _Traits> 		__streambuf_type;
      typedef basic_ios<_CharT, _Traits>		__ios_type;
      typedef basic_ostream<_CharT, _Traits>		__ostream_type;
      typedef num_put<_CharT, ostreambuf_iterator<_CharT, _Traits> >
      							__num_put_type;
      typedef ctype<_CharT>	      			__ctype_type;
      //调用basic_ios::int
            explicit
      basic_ostream(__streambuf_type* __sb)
      { this->init(__sb); }
      
      class sentry(哨兵);
      friend class sentry;
        template <typename _CharT, typename _Traits>
    class basic_ostream<_CharT, _Traits>::sentry
    {
      // Data Members.
      bool 				_M_ok;
      basic_ostream<_CharT, _Traits>& 	_M_os;

    public:
    /*tcc中实现
    *   if(_os.tie&&os.good)
    *     _os.ite().flush
    *       if (__os.good())
	  _M_ok = true;
        else
	 __os.setstate(ios_base::failbit);
    */
      explicit
      sentry(basic_ostream<_CharT, _Traits>& __os);

      /**
       *  @brief  Possibly flushes the stream.
       *
       *  If @c ios_base::unitbuf is set in @c os.flags(), and
       *  @c std::uncaught_exception() is true, the sentry destructor calls
       *  @c flush() on the output stream.
      */
      ~sentry()
      {
	// XXX MT
	if (bool(_M_os.flags() & ios_base::unitbuf) && !uncaught_exception())
	  {
	    // Can't call flush directly or else will get into recursive lock.
	    if (_M_os.rdbuf() && _M_os.rdbuf()->pubsync() == -1)
	      _M_os.setstate(ios_base::badbit);
	  }
      //重写bool
      #if __cplusplus >= 201103L
      explicit
#endif
      operator bool() const
      { return _M_ok; }
      }
 //<<输出操作符,直接输出的操作符都调用了这个模板函数,定义在tcc中
template<typename _CharT, typename _Traits>
    template<typename _ValueT>
      basic_ostream<_CharT, _Traits>&
      basic_ostream<_CharT, _Traits>::
      _M_insert(_ValueT __v)
      {
	sentry __cerb(*this);  //这里就明白了,sentry就是在流进行io操作时对流本身进行的一次检验
	if (__cerb)//sentry的bool转换,查看 sentry::_M_ok
	  {
	    ios_base::iostate __err = ios_base::goodbit;
	    __try
	      {
		const __num_put_type& __np = __check_facet(this->_M_num_put); //_M_num_put时facet的子类
		if (__np.put(*this, *this, this->fill(), __v).failed())  //这里让人很困惑的原因时,第一个参数是隐式构造成的,cnm
		  __err |= ios_base::badbit;
	      }
	    __catch(__cxxabiv1::__forced_unwind&)
	      {
		this->_M_setstate(ios_base::badbit);		
		__throw_exception_again;
	      }
	    __catch(...)
	      { this->_M_setstate(ios_base::badbit); }
	    if (__err)
	      this->setstate(__err);
	  }
	return *this;
      }


{%endcodeblock%}