---
layout: post
title:  C语言实现分割字符串
date:   2017-12-29 17:54:09 +0800
categories: Dev
tag: C/C++
---

* content
{:toc}

背景
----------
遇到一个将字符串分割场景.以前从没有用c语言实现,都是使用python的split()函数,
python处理起来很简单.

split()方法语法：

    str.split(str="", num=string.count(str)).
    • str -- 分隔符，默认为所有的空字符，包括空格、换行(\n)、制表符(\t)等。
    • num -- 分割次数。
    返回分割后的字符串列表。
    用例:
    #!/usr/bin/python
    str = "Line1-abcdef \nLine2-abc \nLine4-abcd";
    print str.split( );
    print str.split(' ', 1 );
    以上实例输出结果如下：
    ['Line1-abcdef', 'Line2-abc', 'Line4-abcd']
    ['Line1-abcdef', '\nLine2-abc \nLine4-abcd']

Python split()通过指定分隔符对字符串进行切片，如果参数num 有指定值，则仅分隔 num 个子字符串

C语言扩展
----------
那使用C语言如何实现？

网上搜索到有个strtok函数具有分割字符串功能，信息如下

    # man strtok
    char *strtok(char *str, const char *delim);
    The  strtok()  function  breaks a string into a sequence of zero or more nonempty tokens.  On the first call to strtok() the
    string to be parsed should be specified in str.  In each subsequent call that should parse the same string, str must be NULL.

    C语言实现python中的split函数:
    #include <stdio.h>
    #include <string.h>
     
    // 将str字符以spl分割,存于dst中，并返回子字符串数量
    int split(char dst[][80], char* str, const char* spl)
    {
        int n = 0;
        char *result = NULL;
        result = strtok(str, spl);
        while( result != NULL )
        {
            strcpy(dst[n++], result);
            result = strtok(NULL, spl);
        }
        return n;
    }
     
    int main()
    {
        char str[] = "what is your name?";
        char dst[10][80];
        int cnt = split(dst, str, " ");
        for (int i = 0; i < cnt; i++)
            puts(dst[i]);
        return 0;
    }

开始时strtok第一次调用str传入需分割的字符串,返回第一段分割出来的字符串"what"的地址(以'\0'结尾)，后续调用str为NULL，依次返回后续分割
字符串"is""your""name?"的地址.

到这里感觉有点奇怪，为什么后续str为NULL还能继续对"what is your name?"进行分割，它与第一次调用的关联在哪里?

strtok源码分析
----------
下载c语言标准库源码:

	git clone git://sourceware.org/git/glibc.git
	cd glibc
	git checkout --track -b local_glibc-2.26 origin/release/2.26/master

`Strtok.c`:

	char * strtok (char *s, const char *delim)
	{
	  static char *olds;//初始为0，每次调用strtok_r后设置成下一次带分割字符串地址
	  return __strtok_r (s, delim, &olds);
	}

`Strtok_r.c`:

	#ifndef _LIBC
	/* Get specification.  */
	# include "strtok_r.h"
	# define __strtok_r strtok_r //可直接使用strtok_r()函数分割字符串
	#endif

	char * __strtok_r (char *s, const char *delim, char **save_ptr)
	{
	  char *end;

	  if (s == NULL) //后续分割字符串时str设置为NULL，使用上次存储在static变量中的地址
		s = *save_ptr;

	  if (*s == '\0')  //待分割字符串已到末尾，结束并返回NULL
		{
		  *save_ptr = s;
		  return NULL;
		}

	  /* Scan leading delimiters.  */
	  s += strspn (s, delim); //跳过直到第一个非分隔符的地址或字符串结尾
	  if (*s == '\0') //待分割字符串已到末尾，结束并返回NULL
		{
		  *save_ptr = s;
		  return NULL;
		}

	  /* Find the end of the token.  */
	  end = s + strcspn (s, delim); //跳过直到第一个分隔符地址或字符串结尾
	  if (*end == '\0') //待分割字符串已到末尾，结束并返回最后一个分割完的字符串
		{
		  *save_ptr = end;
		  return s;
		}

	  /* Terminate the token and make *SAVE_PTR point past it.  */
	  *end = '\0'; //分割好的字符串末尾置为终止符
	  *save_ptr = end + 1; //保存下一次待分割字符串地址
	  return s; //返回本次分割完的字符串
	}

可以看到原来strtok()使用static变量存放下一次待分割字符串的地址,当传入str为NULL时使用该地址继续分割,
即后续strtok()调用是通过static变量与第一次strtok()关联的.

strtok与strtok_r
----------
手册里提到strtok()是非线程安全的:

	ATTRIBUTES
    For an explanation of the terms used in this section, see attributes(7).
    ┌───────────┬───────────────┬───────────────────────┐
    │Interface  │ Attribute     │ Value                 │
    ├───────────┼───────────────┼───────────────────────┤
    │strtok()   │ Thread safety │ MT-Unsafe race:strtok │
    ├───────────┼───────────────┼───────────────────────┤
    │strtok_r() │ Thread safety │ MT-Safe               │
    └───────────┴───────────────┴───────────────────────┘

因为根据其定义，strtok()必须使用内部静态变量来记录字符串中下一个需要解析的标记的当前位置。但是，由于指示这个位置的变量只有一个，那么，在同一个程序中出现多个解析不同字符串的strtok调用时，各自的字符串的解析就会互相干扰。

POSIX定义了一个线程安全的函数——`strtok_r`，以此来代替strtok

带有_r的函数主要来自于UNIX下面。 所有的带有_r和不带_r的函数的区别的是：带_r的函数是线程安全的，r的意思是reentrant，可重入的

另外要注意的是`strtok`和`strtok_r`均对字符串本身做了修改，将分隔符替换成'\0'了

strtok进阶:strsep
----------
搜索strtok字符串时，在`linux kernel(4.14.9 version)`源码文件string.c中看到文件开始的注释里提到:

     * * Fri Jun 25 1999, Ingo Oeser <ioe@informatik.tu-chemnitz.de>
      * -  Added strsep() which will replace strtok() soon (because strsep() is
       *    reentrant and should be faster). Use only strsep() in new code, please.

即新code建议使用strsep替换strtok,由于前者可重入且更快...1999年时就建议用strsep()了，结果在网上一搜分割字符串还都是strtok()...

查看strsep手册及用法

    #man strsep
    char *strsep(char **stringp, const char *delim);
	用法:
	#include <stdio.h>  
	#include <string.h>  
	  
	int main(void) {  
	    char source[] = "hello, world! welcome to china!";  
	    char delim[] = " ,!";  
	  
	    char *s = strdup(source);  
	    char *token;  
	    for(token = strsep(&s, delim); token != NULL; token = strsep(&s, delim)) {  
	        printf(token);  
	        printf("+");  
	    }  
	    printf("\n");  
	    return 0;  
	}  
	//output: hello++world++welcome+to+china++

对比strtok用法:

	#include <stdio.h>  
	#include <string.h>  
	  
	int main(void) {  
	    char s[] = "hello, world! welcome to china!";  
	    char delim[] = " ,!";  
	  
	    char *token;  
	    for(token = strtok(s, delim); token != NULL; token = strtok(NULL, delim)) {  
	        printf(token);  
	        printf("+");  
	    }  
	    printf("\n");  
	    return 0;  
	}  
	//output: hello+world+welcome+china+

二者第一个参数有差别: 由于strsep()执行完需要改变字符串s的地址以指向下一次分割开始,所以传递字符串s地址的指针

二者输出也有些差别，出现连续++的打印是因为中间输出了空字符串，在strsep()中碰到连续分隔符时会认为
分隔符之间有空字符串(这一点与python的split是一致的),详见[strsep源码](#strsep)

而strtok()会使用`s += strspn (s, delim);`跳过直到第一个非分隔符的地址或字符串结尾,而不会分割出空字符串.

strsep源码分析
----------
<span id="strsep">源码如下</span>

	char * __strsep (char **stringp, const char *delim)
	{
	  char *begin, *end;

	  begin = *stringp;
	  if (begin == NULL) //处理为NULL情况
		return NULL;

	  /* Find the end of the token.  */
	  end = begin + strcspn (begin, delim); //跳过直到第一个分隔符地址或字符串结尾

	  if (*end) //若为分隔符则将其设置为'\0'，并将下次开始分割的地址赋值为end+1
		{
		  /* Terminate the token and set *STRINGP past NUL character.  */
		  *end++ = '\0';
		  *stringp = end;
		}
	  else //若为字符串结尾则将下次开始分割的地址赋值为NULL,下次处理直接返回NULL
		/* No more delimiters; this is the last token.  */
		*stringp = NULL;

	  return begin; //返回分割字符串地址
	}

使用注意事项
----------
`strsep(),strtok(),strtok_r()`三者都需注意:

	函数会对传入的第一个参数即待分割字符串进行修改,即把分隔符替换成'\0',由于该原因所以参数不能指向只读字符串.

所以通常的做法是把待分割字符串拷贝一份来处理,如上面用法中strdup的使用.

strspn源码分析
----------
strtok函数中使用到strspn()来定位非分隔符地址.

    Strspn.c:
    /* Return the length of the maximum initial segment
    of S which contains only characters in ACCEPT.  */
    size_t STRSPN (const char *str, const char *accept)
    {
      if (accept[0] == '\0')
        return 0;
      if (__glibc_unlikely (accept[1] == '\0')) //分支预测优化
        {
          const char *a = str;
          for (; *str == *accept; str++);
          return str - a;
        }
    
    //测试了下分开的时间是不分开的2倍，不理解。
    //从代码本身注释看memset 64个字节可以直接在此处将memset函数当内联函数展开？
      /* Use multiple small memsets to enable inlining on most targets.  */
      unsigned char table[256];
      unsigned char *p = memset (table, 0, 64);
      memset (p + 64, 0, 64);
      memset (p + 128, 0, 64);
      memset (p + 192, 0, 64);
    
      unsigned char *s = (unsigned char*) accept;
      /* Different from strcspn it does not add the NULL on the table
         so can avoid check if str[i] is NULL, since table['\0'] will
         be 0 and thus stopping the loop check.  */
      do
        p[*s++] = 1; //将分隔符对应index处置1
      while (*s);
    
      s = (unsigned char*) str;
      if (!p[s[0]]) return 0;
      if (!p[s[1]]) return 1;
      if (!p[s[2]]) return 2;
      if (!p[s[3]]) return 3;
    
      //宏定义PTR_ALIGN_DOWN将向下找到与4字节对齐的地址返回
      //PTR_ALIGN_DOWN (0x3, 4); return 0x0
      //PTR_ALIGN_DOWN (0x4, 4); return 0x4
      //PTR_ALIGN_DOWN (0x5, 4); return 0x4
      s = (unsigned char *) PTR_ALIGN_DOWN (s, 4);
    
      unsigned int c0, c1, c2, c3;
      do {
          s += 4;
          c0 = p[s[0]];
          c1 = p[s[1]];
          c2 = p[s[2]];
          c3 = p[s[3]];
      } while ((c0 & c1 & c2 & c3) != 0);
    
      size_t count = s - (unsigned char *) str;
     //此处为何不直接return count+c0+c1+c2,估计涉及到编译优化吧
      return (c0 & c1) == 0 ? count + c0 : count + c2 + 2;
    }

计时统计
----------
统计strspn中耗时使用如下代码

    struct timeval t_val;
    gettimeofday(&t_val, NULL);

    //write your process here

    struct timeval t_val_end;
    gettimeofday(&t_val_end, NULL);
    struct timeval t_result;
    timersub(&t_val_end, &t_val, &t_result);
    double consume = t_result.tv_sec + (1.0 * t_result.tv_usec)/1000000;
    printf("end.elapsed time= %fs \n", consume);

strtok源码NetBSD及微软实现
----------
来源于某博客[C库源代码NetBSD及微软实现: strtok](http://www.cppblog.com/yinquan/archive/2009/06/01/86411.html)

NetBSD简单粗暴地遍历，而微软通过32bytes的空间将时间复杂度降至N，比glibc中256bytes空间利用率高

C语言实现split
----------
C语言分割字符串可使用strsep(),最好复制一份进行分割.

	#include<stdio.h>
	#include <string.h>  
	
	int split(char *str, const char *delim, char dst[][80]) {
	    char *s = strdup(str);  
	    char *token;  
	    int n = 0;
	    for(token = strsep(&s, delim); token != NULL; token = strsep(&s, delim)) {  
	        strcpy(dst[n++], token);
	    }  
	    return n;
	}
	
	int main(void) {  
	    char source[] = "hello, world! welcome to china!";  
	    char delim[] = " ,!";  
	    char dst[10][80];
	    int cnt = split(source, delim, dst);
	    for (int i = 0; i < cnt; i++)
	        printf("%s\n",dst[i]);
	    return 0;  
	}  

python的split()是返回分割字符串数组的，上面实现这个返回的动态数组比较困难,用固定的`dst[10][80]`...限制太多

还是C++的动态数组vector好用...

C++实现split
----------
使用substr()定位分隔符,vector的`push_back()`动态插入分割子字符串

	//字符串分割函数
	std::vector<std::string> split(std::string str,std::string pattern)
	{
	    std::string::size_type pos;
	    std::vector<std::string> result;
	    str+=pattern;//扩展字符串以方便操作
	    int size=str.size();
	
	    for(int i=0; i<size; i++)
	    {
	        pos=str.find(pattern,i);
	        if(pos<size)
	        {
	            std::string s=str.substr(i,pos-i);
	            result.push_back(s);
	            i=pos+pattern.size()-1;
	        }
	    }
	    return result;
	}

总结
----------
C语言实现split()可以使用strsep(),但还是不好用，能用python处理最好，应用C++也是不错的.

libc库底层的实现有很多优化的思想...拜一拜...

参考
----------

[python split使用文档](http://www.runoob.com/python/att-string-split.html)

[线程安全——strtok VS strtok_r](http://blog.csdn.net/ixidof/article/details/5154016)

[strtok和strsep函数详解](http://blog.csdn.net/yafeng_jiang/article/details/7109285)

[likely()/unlikely() macros in the Linux kernel - how do they work?benefit?](https://stackoverflow.com/questions/109710/likely-unlikely-macros-in-the-linux-kernel-how-do-they-work-whats-their)

[字符串分割(C++)](https://www.cnblogs.com/MikeZhang/archive/2012/03/24/MySplitFunCPP.html)
