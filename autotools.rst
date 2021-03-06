====================
 Autotools 实例分析
====================
:Author: 张森
:Email: sen.zhang@windriver.com

.. contents:: 目录
    :depth: 2

简介
~~~~

autotools 的主要目的是方便用户, 简化软件编译的步骤。用 autotools 搭建的软件都有
这样的“标准”编译步骤::

    $ ./configure
    $ make
    $ sudo make install

用户不需要自己检查系统配置, 软件的依赖或者安装路径，autotools 能保证系统满足软
件的运行条件。

这三步已经成为 Linux (以及其他 UNIX 系统) 上编译软件的标准命令, 其他编译系统 (
比如 cmake, scons, python 的 setup.py) 尽管有各自的好处, 但反而不容易被用户接受
。

下面以 xwininfo 为例, 分析一下 autotools 的使用。xwininfo 是一个很简单的软件,
只为用户提供两个文件::

    /usr/share/man/man1/xwininfo.1.gz
    /usr/bin/xwininfo

在 /usr/bin 目录下安装了一个可执行程序, 然后在 /usr/share 目录下安装了一个手册。

xwininfo 的编译系统
~~~~~~~~~~~~~~~~~~~

首先看一下 xwininfo 的源代码。这是项目 `git 仓库`_ 中的目录树::

    |-- COPYING
    |-- Makefile.am
    |-- README
    |-- autogen.sh
    |-- clientwin.c
    |-- clientwin.h
    |-- configure.ac
    |-- dsimple.c
    |-- dsimple.h
    |-- strnlen.c
    |-- strnlen.h
    |-- xwininfo.c
    `-- xwininfo.man

    0 directories, 13 files

.. _git 仓库: http://cgit.freedesktop.org/xorg/app/xwininfo/

从开发者的角度看，xwininfo 只有 13 个文件。可以分成三类。

代码
    clientwin.c  clientwin.h  dsimple.c  dsimple.h  strnlen.c  strnlen.h
    xwininfo.c
文档
    COPYING  README  xwininfo.man
编译系统
    autogen.sh  configure.ac  Makefile.am

除去代码和文档，有 autogen.sh, configure.ac 和 Makefile.am。它们组成了 xwininfo
的编译系统。

跟 autotools 相关的就是这三个文件, 但软件的用户不会直接接触这些文件。用户手里拿
到的，是一个压缩包。看压缩包 (xwininfo-1.1.1.tar.bz2) 的内容::

    |-- COPYING
    |-- Makefile.am
    |-- README
    |-- autogen.sh
    |-- clientwin.c
    |-- clientwin.h
    |-- configure.ac
    |-- dsimple.c
    |-- dsimple.h
    |-- strnlen.c
    |-- strnlen.h
    |-- xwininfo.c
    `-- xwininfo.man

    |-- ChangeLog
    |-- INSTALL
    |-- Makefile.in
    |-- aclocal.m4
    |-- config.guess
    |-- config.h.in
    |-- config.sub
    |-- configure
    |-- depcomp
    |-- install-sh
    |-- missing

    0 directories, 24 files

tar 包里多了十一个文件。这些文件才是给用户使用的, 比如 configure 脚本。
autotools 要做的所有工作就是生成这些文件; 用户不需要直接跟 autotools 打交道。

顺便看一下编译完成之后的目录树::

    xwininfo-1.1.1/
    |-- COPYING
    |-- ChangeLog
    |-- INSTALL
    |-- Makefile
    |-- Makefile.am
    |-- Makefile.in
    |-- README
    |-- aclocal.m4
    |-- autogen.sh
    |-- autom4te.cache
    |   |-- output.0
    |   |-- output.1
    |   |-- output.2
    |   |-- requests
    |   |-- traces.0
    |   |-- traces.1
    |   `-- traces.2
    |-- clientwin.c
    |-- clientwin.h
    |-- clientwin.o
    |-- config.guess
    |-- config.h
    |-- config.h.in
    |-- config.h.in~
    |-- config.log
    |-- config.status
    |-- config.sub
    |-- configure
    |-- configure.ac
    |-- depcomp
    |-- dsimple.c
    |-- dsimple.h
    |-- dsimple.o
    |-- install-sh
    |-- m4
    |-- missing
    |-- stamp-h1
    |-- strnlen.c
    |-- strnlen.h
    |-- strnlen.o
    |-- xwininfo
    |-- xwininfo.1
    |-- xwininfo.c
    |-- xwininfo.man
    `-- xwininfo.o

    2 directories, 43 files

其中除了编译过程中生成的临时文件 (比如 .o), 还有 configure 脚本产生的日志和临时
文件。

什么是 Autotools?
~~~~~~~~~~~~~~~~~

自由软件基金会 (Free Software Foundation) 为他们的 GNU 系统开发了 GNU build
system, 也被称作 Autotools。这是一套编程工具的集合, 目的是帮助提高源代码包在类
UNIX 系统上的移植性。也就是说, 它关注的是源代码的移植性, 而不是二进制程序的移植
性。

GNU build system (以下称作 autotools) 提供的工具包括:

- Autoconf - 生成 configure 文件 (configure.ac -> configure)
- Automake - 生成 Makefile 模板 (Makefile.am -> Makefile.in)
- Libtool - 生成共享库

.. figure:: images/autoconf.png

    autoconf 和 automake

上图解释了一个软件从 git 仓库到安装到用户系统上的过程。整个过程分成两个部分, 分
别由开发者和用户主导。

开发者使用 autotools (autoconf 和 automake) 生成必要的文件, 然后将软件源码包分
发给用户。用户拿到源码后, 通过几个标准的命令来编译这个软件:

configure
    由 autoconf 生成
make
    在用户系统上安装。Makefile 是由 configure 从 Makefile.in 生成的。

下面具体讲究 autotools 的各个组件。

Autoconf
========

Autoconf 是 autotools 套件中最早出现的 (1991 年)。它解决的问题包括：

- 检查系统上的库和头文件
- 确定软件的最终安装路径
- 正确选择软件的组件和功能

当然, 这些都是 configure 脚本的功能。而 autoconf 的作用是生成这个 configure 脚
本。

Autoconf 提供的可执行程序包括：

1. autoconf
#. autom4te
#. autoreconf
#. autoheader
#. autoscan

.. figure:: images/autoconf_ahdr_dataflow.png

    autoconf 和 autoheader 的数据流图

简单地说, 开发者编写好 configure.ac 后, 调用 autoconf 以及其他的程序, 生成
configure。当然, 具体的操作更复杂一点, 由多个工具相互协作完成。

上图中 aclocal.m4 (左下角) 的颜色很模糊, 因为它的角色确实不太确定。有时候
aclocal.m4 是开发者手写的, 有时候是生成的。下面会讲到这点。

autoconf
--------

autoconf 是一个简单的 .sh 脚本。主要功能是检查当前 shell 能否支持 M4 的处理。然
后在对命令行参数进行简单解析后, 把控制权转给 autom4te::

    $ tail -n6 /usr/bin/autoconf
    # Run autom4te with expansion.
    eval set x "$autom4te_options" \
      --language=autoconf --output=\"\$outfile\" "$traces" \"\$infile\"
    shift
    $verbose && $as_echo "$as_me: running $AUTOM4TE $*" >&2
    exec "$AUTOM4TE" "$@"

也就是说, configure 实际上是由 autom4te 完成的。autom4te 读做 “automate”。

autom4te
--------

而事实上, autom4te 又是对 m4 的一个封装。autom4te 的作用是提供一个缓存, 加快 m4
的处理速度。我们经常能看到这样一个缓存目录::

    $ ls autom4te.cache/
    output.0  output.1  output.2  requests  traces.0  traces.1  traces.2

所以从 configure.ac 到 configure 的转换, 本质上是由 m4 完成的。

autoreconf
----------

autoreconf 可以看作是对所有 autotools 的封装, 它能够自动选择调用合适的工具, 最
终生成 configure 脚本以及其他的文件。

autoheader
----------

autoheader 能够生成一个“头文件模板”, 一般叫做 config.h.in 。里边包含对项目组件
和各种特性的开关(也就是宏定义)::

    $ head config.h.in
    /* config.h.in.  Generated from configure.ac by autoheader.  */

    /* Define to 1 if you have the iconv() function */
    #undef HAVE_ICONV

    /* Define to 1 if you have the <inttypes.h> header file. */
    #undef HAVE_INTTYPES_H

    /* Define to 1 if you have the <memory.h> header file. */
    #undef HAVE_MEMORY_H

用户执行 configure 后, 会根据 config.h.in 生成 config.h, 其中的 #undef 根据用户
系统的实际情况被替换为了真实数值::

    $ head config.h
    /* config.h.  Generated from config.h.in by configure.  */
    /* config.h.in.  Generated from configure.ac by autoheader.  */

    /* Define to 1 if you have the iconv() function */
    #define HAVE_ICONV 1

    /* Define to 1 if you have the <inttypes.h> header file. */
    #define HAVE_INTTYPES_H 1

    /* Define to 1 if you have the <memory.h> header file. */

有一类文件我们称之为模板文件。它们以 .in 做为文件名后缀, 比如 config.h.in,
Makefile.in。模板文件可能是手写的, 也可能是生成的; 由 configure 处理成最终文件.

autoscan
--------
autoscan 能够扫描项目源代码, 自动生成 configure.ac。

Automake
========

在 automake 出现之前, 人们必须手写 Makefile。但是项目稍微有点规模后, Makefile
就很容易变得又长又臭, 很难维护。

大多数项目在结构上都是类似的。无论项目的代码文件里有什么, 都是组织在树状的目录
里, 安装功能放在不同的子目录中。并且一般都要支持这些常见的 make 操作::

    $ make
    $ make clean
    $ make check
    $ make dist
    ....

所以 automake 出现了 (1994 年), 它能够自动生成 Makefile, 从而简化 Makefile 的维
护。

Automake 提供两个可执行程序:

1. automake
#. aclocal

automake
--------

automake 能够从抽象的高层描述 (Makefile.am) 生成具体的 makefile 模板
(Makefile.in)::

    $ wc Makefile*
       60   266  1901 Makefile.am
      763  3087 25552 Makefile.in
      763  3207 28080 Makefile

可以看到 Makefile.am 很短。而自动生成的 Makefile.in 和 Makefile 行数相同。

前面已经提到了, Makefile.in 会由 configure 转换为最终的 Makefile。

从语法上讲, Makefile.am 也是标准的 makefile。所以如果 automake 生成的某条规则不
满足要求, 可以直接把我们想要的结果写在 Makefile.am 里, 这样 automake 会采取我们
指定的规则, 不会自动生成。

aclocal
-------

automake 实际是对 autoconf 的一个扩展, 也就是提供一系列 m4 宏定义给用户使用。用
户可以在 configure.ac 里调用这些宏。但是, autoconf 最初的设计并没有考虑到这么大
程度的扩展; 这带来了一个问题。

autoconf 最初提供的扩展机制是通过一个叫 aclocal.m4 的文件完成的。用户可以在里边
添加自定义的宏, autoconf 在处理 configure.ac 的时候会自动读取这个文件里的宏定义
。

显然, 如果要使用 automake, 用户必须创建 aclocal.m4, 然后通过 m4_include 把
automake 的宏都包含进来。这样 autoconf 就能处理 automake 宏了。

这个办法不算友好, 因为它把 M4 的一些概念暴露出来了, 而 autoconf 本身是要把 M4
封装起来的, 不想让用户直接接触 M4。

所以开发者设计了 aclocal 来解决这个问题。它能够自动生成 aclocal.m4 文件, 供
autoconf 使用。

这也就是为什么 aclocal.m4 在“autoconf 和 autoheader 的数据流图”中的颜色很模糊。

.. figure:: images/aclocal_dataflow.png

    aclocal 的数据流图

Libtool
=======

Libtool 的目的是简化共享库的开发。尽管各种 UNIX 系统是基本相似的, 但是它们在共
享库的处理上有着各种各样的差别。libtool 可以帮开发者避开这些陷阱。比如:

- 库的命名。libname.so, libname.a, libname.sl。有的系统甚至不支持共享库。
- 库的动态加载。有的系统提供 libdl.so (dlopen), 有的系统提供其他的机制, 有的系
  统不支持动态加载。

Libtool 包提供了这些程序以及头文件和库:

- libtool, 一个 shell 脚本
- libtoolize, 能够为工程生成特定的 libtool 脚本。这个脚本会在用户系统上, 由 make
  执行。
- libdl, 一个通用的共享库加载接口。
- ltdl.h, 头文件

.. figure:: images/automake_libtool_dataflow.png

    automake 和 libtool 数据流图

Automake 和 Libtool 都是对 autoconf 的扩展, 用几个简单的宏调用就能够使能
automake 和 libtool。

从用户角度看
~~~~~~~~~~~~

以上的内容都是从开发者的角度看的。下面分析一下在用户系统上发生的事情, 也就是软
件的编译过程。

执行 ./configure
================

用户拿到了一个 tarball。里边有这些东西:

- 源代码 (.c, .h)
- 一个 configure 脚本
- 一些模板文件, 包括 Makefile.in, config.h.in 等等。其他例子有 glib.pc.in,
  trashapplet-empty-progress.ui.in。

用户需要进行的操作很简单, 就是一条命令: ::

    ./configure --prefix=/usr --enable-foo=yes --enable-bar=no...

.. figure:: images/configure_dataflow.png

    configure 过程数据流图

1. configure 脚本会收集系统信息, 以及用户给的命令行选项。
#. configure 最终生成 config.status 脚本, 并且执行这个脚本。
#. config.status 把模板文件都处理为最终的文件。
#. 日志记录到 config.log 文件里。

执行 make
=========

Makefile 已经由 configure 生成, 用户只需要执行一个简单的 make 命令。

.. figure:: images/make_dataflow.png

    make 过程数据流图


autotools 实例
~~~~~~~~~~~~~~

下面以 xwininfo 为例, 具体的讲解 autotools 的使用。

编写 configure.ac
=================

使用 autotools 管理工程的第一步是编写 configure.ac。

我们先看一个最短的 confgure.ac 是什么样子的。

最短的 configure.ac
-------------------

::

    AC_INIT([Jupiter], [1.0])
    AC_OUTPUT

编写 configure.ac 的语言是 M4 。M4 是一种宏处理语言 (macro processor), 本质上就
是对宏定义的文本递归扩展。上例中是两个 M4 宏调用, 被扩展后, 就生成了最终的
configure 脚本。

M4 宏与 C 语言的预处理宏有很多相似之处。这很容易理解, 因为它们都是进行简单的文
本替换, 而且作者都是 Brian Kernighan 和 Dennis Ritchie。

这两个宏是由 autoconf 定义的 (/usr/share/autoconf/autoconf/general.m4)。宏的用
法与 C 语言类似。

- 参数可以用括号 () 传递。没有参数可以不写括号。

- 在使用 autoconf 时, 必要的时候要用方括号 [] (而不是常见的双引号或者圆括号等等
  ) 把参数括起来。

在一个 configure.ac 脚本中, 有两个宏是必须的:

- AC_INIT(package, version, [bug-report], [tarname], [url])
    初始化 autoconf 系统。

- AC_OUTPUT
    生成并调用 config.status。每个 configure.ac 都应该在最后调用此宏。在
    AC_OUTPUT 之后执行的动作不会对 configure 过程产生作用。有的项目会在最后写一
    条 echo 语句, 打印一些 configure 信息。

完整的 configure.ac 的内容一般是这样的: ::

    AC_INIT
     测试程序
     测试函数库
     测试头文件
     测试类型定义
     测试结构
     测试编译器特性
     测试库函数
     测试系统调用
    AC_OUTPUT

生成 configure
--------------

调用 autotools 工具链的推荐办法是 autoreconf。有的工程为了方便, 会有一个简单的
autogen.sh 脚本。这是 xwininfo 的 autogen.sh::

    #! /bin/sh

    srcdir=`dirname $0`
    test -z "$srcdir" && srcdir=.

    ORIGDIR=`pwd`
    cd $srcdir

    autoreconf -v --install || exit 1
    cd $ORIGDIR || exit $?

    $srcdir/configure --enable-maintainer-mode "$@"

可以看到 xwininfo 的 autogen.sh 只是对 autoreconf 的封装, 最后直接调用刚刚生成
的 configure。

一个小技巧: 有些 GNOME 模块会在 autogen.sh 里调用 gnome-autogen.sh。
gnome-autogen.sh 支持一个 NOCONFIGURE 环境变量, 如果它被设为 yes, 就不会直接执
行 configure::

    NOCONFIGURE=yes ./autogen.sh

只生成 configure (及其它文件) 而不执行。

调用 config.status
------------------

前边提到过真正负责把 .in 模板文件转换为普通文件的是 config.status。实际上我们也
可以直接调用 ./config.status。这也是 autoconf 的设计目的之一, 把检查系统和转换
模板文件分成两步, 分别由两个脚本完成, 这样就可以单独地执行模板文件的转换, 不需
要费时再次检查系统。

但是 config.status 的更大作用是给 make 使用: ::

    Makefile: $(srcdir)/Makefile.in $(top_builddir)/config.status
            @case '$?' in \
              *config.status*) \
                $(SHELL) ./config.status;; \
              *) \
                cd $(top_builddir) && $(SHELL) ./config.status $@ $(am__depfiles_maybe);; \
            esac;

当 Makefile.in 模板文件被改变后, 可以自动更新 Makefile。

xwininfo 的 configure.ac
------------------------

下面逐行分析 xwininfo 的 configure.ac。

dnl
...

::

    dnl  Copyright 2005 Red Hat, Inc.
    dnl
    dnl  Permission to use, copy, modify, distribute, and sell this software and its
    dnl  documentation for any purpose is hereby granted without fee, provided that
    dnl  the above copyright notice appear in all copies and that both that
    dnl  copyright notice and this permission notice appear in supporting

dnl 的意思是 discard to next line。相当于注释, 但是实际上这些行都被丢弃了, 不会
出现在最终的扩展结果(也就是 configure) 中。

初始化
......

::

    AC_PREREQ([2.60])
    AC_INIT([xwininfo], [1.1.1],
            [https://bugs.freedesktop.org/enter_bug.cgi?product=xorg], [xwininfo])

指定 autoconf 最低版本。初始化 Autoconf。

初始化 automake
...............

::

    AM_INIT_AUTOMAKE([foreign dist-bzip2])
    AM_MAINTAINER_MODE

初始化 Automake 。详细内容会在下一小节讲到。

检查外部依赖
............

::

    # Require X.Org macros 1.8 or later for MAN_SUBSTS set by XORG_MANPAGE_SECTIONS
    m4_ifndef([XORG_MACROS_VERSION],
              [m4_fatal([must install xorg-macros 1.8 or later before running autoconf/autogen])])
    XORG_MACROS_VERSION(1.8)

m4_ifndef 是 M4 的一个内置宏, 作用跟 CPP 的 #ifndef 类似。如果
XORG_MACROS_VERSION 不存在, 就调用 m4_fatal 打印错误信息后退出。

XORG_MACROS_VERSION 由 util-macros 提供, 检查 util-macros 的版本是否大于 1.8。

有时候我们会遇到未识别的宏的问题。比如, 如果 util-macros 没有安装, 那么当开发者
执行 autoconf 的时候, autoconf 找不到 XORG_MACROS_VERSION 的定义, 最后生成的
configure 脚本就会有一个未被扩展的 XORG_MACROS_VERSION 调用。这时候再执行
./configure, 就会出现::

    XORG_MACROS_VERSION: command not found

config.h 头文件
................

::

    AM_CONFIG_HEADER(config.h)

指定一个头文件 config.h。AC_OUTPUT 会创建 config.h，把一些 CPP #define 宏放入这
个文件，并且把模板文件里的 @DEFS@ 替换为 DEFS 实际值。

更多检查
........

::

    AC_CHECK_FUNCS([strlcat])

    AC_FUNC_STRNLEN
    if test "x$ac_cv_func_strnlen_working" = xyes; then
      AC_DEFINE(HAVE_STRNLEN, 1, [Define to 1 if you have a working strnlen function.])
    fi

    # Check for iconv in libc, then libiconv
    AC_SEARCH_LIBS([iconv], [iconv], [AC_DEFINE([HAVE_ICONV], 1,
            [Define to 1 if you have the iconv() function])])

对系统进行各种检查。configure 进行检查的方式是，生成简单的源代码文件，根据是否
编译成功来判断系统能不能支持某种特性。以上的片段检查 strlcat 和 strnlen 函数，
并且查找 iconv 库。

如果找到了 strnlen，configure 就会定义一个 HAVE_STRNLEN 宏，值设为1。config.h 里就会有: ::

    #define HAVE_STRNLEN 1

否则就是: ::

    /* #undef HAVE_STRNLEN */

AC_CHECK_FUNCS 原型: ::

    AC_CHECK_FUNCS (function..., [action-if-found], [action-if-not-found])

检查结果缓存在 ac_cv_func_*function* 变量里。

AC_SEARCH_LIBS 原型: ::

    AC_CHECK_LIB (library, function, [action-if-found], [action-if-not-found], [other-libraries])

检查结果缓存在 ac_cv_lib_*library*_*function* 变量里。

自定义选项
..........

::

    # Allow using xcb-icccm, but don't make it the default while the API is
    # still being changed.
    AC_MSG_CHECKING([whether to use xcb-icccm library])
    AC_ARG_WITH([xcb-icccm],
                [AS_HELP_STRING([--with-xcb-icccm],
                                [use xcb-icccm (default: no)])],
                [], [with_xcb_icccm=no])
    AC_MSG_RESULT([$with_xcb_icccm])
    if test "x$with_xcb_icccm" != xno ; then
            AC_DEFINE([USE_XCB_ICCCM], 1,
                      [Define to 1 to call xcb-icccm library functions instead of local replacements])
            xcb_icccm_pc="xcb-icccm"
    fi

这一段代码为 configure 脚本添加了一个自定义选项, --with-xcb-icccm=[yes|no]。如
果使能了此特性, 就把 USE_XCB_ICCCM 置为 1。同样，这个定义会出现在 config.h 中: ::

    ---- config.h ----

    /* Define to 1 to call xcb-icccm library functions instead of local
       replacements */
    /* #undef USE_XCB_ICCCM */

AC_ARG_WITH 原型: ::

    AC_ARG_WITH (package, help-string, [action-if-given], [action-if-not-given])

pkg-config
..........

::

    # Checks for pkg-config packages
    PKG_CHECK_MODULES(XWININFO, [xcb >= 1.6] xcb-shape ${xcb_icccm_pc})

    # Even when using xcb, xproto is still required for Xfuncproto.h
    # and libX11 headers for cursorfont.h
    PKG_CHECK_MODULES(XLIB, x11 [xproto >= 7.0.17])
    XWININFO_CFLAGS="${XWININFO_CFLAGS} ${XLIB_CFLAGS}"

这段代码使用了 pkgconfig 检查依赖。PKG_CHECK_MODULES 原型: ::

    PKG_CHECK_MODULES(VARIABLE-PREFIX, modules [,action-if-found, [action-if-not-found]])

pkgconfig 也提供了一个命令 pkg-config: ::

    $ pkg-config --print-errors 'xcb >= 1.6'
    $ $ echo $?
    0

如果没有安装 xcb: ::

    $ pkg-config --print-errors 'xcb >= 1.6'
    Package xcb was not found in the pkg-config search path.
    Perhaps you should add the directory containing `xcb.pc'
    to the PKG_CONFIG_PATH environment variable
    No package 'xcb' found

PKG_CHECK_MODULES 在检查完模块后，还会定义两个宏，MODULES_LIBS 和
MODULES_CFLAGS。上文最后一行就使用了前一行定义的 XLIB_CFLAGS。

LIBS 和 CFLAGS 也可以用 pkg-config 命令获得: ::

    $ pkg-config --libs 'xcb >= 1.6'
    -lxcb
    $ pkg-config --cflags 'xcb >= 1.6'

    # xcb 没有提供 cflags 选项。

    # 以 glib 为例:
    $ pkg-config --cflags glib-2.0
    -I/usr/include/glib-2.0 -I/usr/lib64/glib-2.0/include

pkg-config 本质上是查看 .pc 文件的内容。各发行版一般都把 .pc 放在
/usr/lib{64}/pkgconfig/ 下。比如: ::

    $ cat /usr/lib64/pkgconfig/xcb.pc
    prefix=/usr
    exec_prefix=/usr
    libdir=/usr/lib64
    includedir=/usr/include
    xcbproto_version=1.6

    Name: XCB
    Description: X-protocol C Binding
    Version: 1.7
    Requires.private: xau >= 0.99.2
    Libs: -L${libdir} -lxcb
    Libs.private:
    Cflags: -I${includedir}

pkg-config 除了检查默认的 /usr/libdir/pkgconfig/，还会检查 PKG_CONFIG_PATH 所指
定的位置。

输出
....

::

    AC_OUTPUT([Makefile])

调用 AC_OUTPUT，生成并执行 config.status。

这是过时的调用方式。现在的 AC_OUTPUT 是不需要参数的。所以上句等同于: ::

    AC_CONFIG_FILES([Makefile])
    AC_OUTPUT

AC_CONFIG_FILES (file..., [cmds], [init-cmds]) 能够指定从 file.in 生成 file。

自动生成 Makefile
=================

使能 automake
-------------
前面提到过 automake 在形式上只是对 autoconf 的扩展, 所以要使能 automake, 要在
configure.ac 里修改。其实只要添加一行 AM_INIT_AUTOMAKE: ::

    AM_INIT_AUTOMAKE([foreign dist-bzip2])

Makefile.am 分析
----------------

Makefile.am 里通常是一些简单的赋值语句, 比如: ::

    SUBDIRS = src

    bin_PROGRAMS = jupiter
    jupiter_SOURCES = main.c

Makefile.am 在形式上是规范的 makefile, 所以上边这个 Makefile.am 仅仅是定义了三
个变量。但是对于 automake 来说，这些变量是有特殊含义的。通过 automake 的处理，
这三行普通的赋值就能变成一个非常完备的 makefile。

上例中首先定义了一个 SUBDIRS 变量。SUBDIRS 的值通常是项目中的几个子目录，比如上
例中的 src。automake 会按照 SUBDIRS 指定的顺序递归地处理这些目录。

上例然后定义了一个 bin_PROGRAMS 和一个 jupiter_SOURCES。它们分别称作 product
list variable (PLV) 和 product source variable (PSV)。

顾名思义，PLV 是一组 product 的列表，而 PSV 定义了每个 product 的“原料”。

product list variable
.....................

PLV 的形式是: ::

    [modifier-list]prefix_PRIMARY = product1 product2 ... productN

先看赋值运算符左边。modifier-list 不是必需的，后面会提到有哪些 modifier。所以一
个 PLV 的名字一般就是 prefix_PRIMARY。prefix 表示安装位置，PRIMARY 表示 product
的类型。

赋值运算符的右边是 PLV 的值，是以空白字符分隔的 product 的名字。

以 bin_PROGRAMS 为例，bin 表示这些 product 要安装到 bindir，(通常)也就是
/usr/bin；PROGRAMS 表示这些 product 都是二进制程序，automake 会按照类型生成合适
的编译命令。这个 PLV 的值只包含一个 product, jupiter。

prefix
''''''

有一些预定义的 prefix，它们都对应于 GCS (GNU Coding Standards) 中的定义，比如
bindir，libdir， sysconfdir。去掉结尾的 ‘dir’，就是能在 Makefile.am 中使用的
prefix (比如 "bin", "lib", "sysconf")。

还有一些 prefix 是跟安装位置无关的，比如 noinst，表示不需要安装；check，表示只
有在 make check 的时候才需要生成。

还可以自定义 prefix: ::

    xmldir = $(datadir)/xml
    xml_DATA = file1.xml file2.xml file3.xml ...

EXTRA
'''''

这里单独提一下 EXTRA 这个 prefix。EXTRA 也是跟安装位置无关的。它用于列举根据
configure 结果可能编译，也可能不编译的对象。

那为什么不能在 Makefile.am 里省略这些可选程序？

Automake 的作用是生成 makefile 规则，所以 automake 必须知道所有对象的存在。
Automake 不关心某个规则是否会被执行。

例子：::

     EXTRA_PROGRAMS = mt rmt
     sbin_PROGRAMS = $(MORE_PROGRAMS)

MORE_PROGRAMS 由 configure 定义，它只能包含 mt 和 rmt 中的一个或多个。如果
MORE_PROGRAMS 包含了其他的值，那显然会出错，因为 makefile 不会有相应的规则。

PRIMARY
'''''''

PLV 名字的最后一部分是 PRIMARY。PRIMARY 可以认为是 product 的种类。比如
PROGRAMS, LIBRARIES, PYTHON, JAVA, SCRIPTS, DATA, DEADERS, MANS, TEXINFOS。这
都是 automake 预定义的 PRIMARY。

也可以自定义 PRIMARY。这里不涉及这个话题。

每个 PRIMARY 都对应自己的编译规则。比如 LIBRARIES, automake 会生成规则，使用合
适的编译器和链接器，生成合适的库文件。

product source variable
.......................

PLV 列出了开发者需要的 product。而 product 与源文件的关系由相应的 product
source variable 指定。

PSV 的形式是: ::

    [modifier-list]product_SOURCES = file1 file2 ... fileN

比如 xwininfo 的 Makefile.am 里有: ::

    bin_PROGRAMS = xwininfo

    xwininfo_SOURCES =	\
            clientwin.c \
            clientwin.h \
            dsimple.c \
            dsimple.h \
            xwininfo.c

PLV and PSV modifiers
.....................

PLV 和 PSV 的名字可以含有一个 modifier-list。modifier 会改变随后的
prefix_PRIMARY 的行为。

这里只简单介绍 dist 和nodist 这两个 modifier。它们表示一个文件是否应该被分发 (
也就是当 make dist 的时候是否应该放入 tar 包)。比如: ::

    dist_myprog_SOURCES = file1.c file2.c
    nodist_myprog_SOURCES = file3.c file4.c

xwininfo 的 Makefile.am
-----------------------

::

    bin_PROGRAMS = xwininfo

    AM_CFLAGS = $(CWARNFLAGS) $(XWININFO_CFLAGS)
    xwininfo_LDADD = $(XWININFO_LIBS) $(LIBOBJS)

    xwininfo_SOURCES =	\
            clientwin.c \
            clientwin.h \
            dsimple.c \
            dsimple.h \
            xwininfo.c

    -------------------------

    appman_PRE = \
            xwininfo.man


    appmandir = $(APP_MAN_DIR)

    appman_DATA = $(appman_PRE:man=@APP_MAN_SUFFIX@)

    -------------------------

    EXTRA_DIST = $(appman_PRE) autogen.sh strnlen.h
    MAINTAINERCLEANFILES = ChangeLog INSTALL
    CLEANFILES = $(appman_DATA)

    .PHONY: ChangeLog INSTALL

    INSTALL:
            $(INSTALL_CMD)

    ChangeLog:
            $(CHANGELOG_CMD)

    dist-hook: ChangeLog INSTALL

    SUFFIXES = .$(APP_MAN_SUFFIX) .man

    # String replacements in MAN_SUBSTS now come from xorg-macros.m4 via configure
    .man.$(APP_MAN_SUFFIX):
            $(AM_V_GEN)$(SED) $(MAN_SUBSTS) < $< > $@

前面提到过 xwininfo 最终安装了两个文件。可以看到 Makefile.am 分别有每个文件的相
应规则。

用 Libtool 构建共享库
=====================

TODO

常见问题 debug
~~~~~~~~~~~~~~

- configure 出错

检查 config.log。

- Perhaps you should add the directory containing 'xcb.pc'

检查 pc 文件是否 ok。

- automake 生成的规则不符合要求

在 Makefile.in (Makefile.am) 里直接定义想要的规则。(未找到例子)

- 需要 enable/disable 某个特性，但是没有提供 --enable/disable 选项

重载 ac_cv_### 值。

ac_cv_###=### configure ....

参考书籍
~~~~~~~~

很长一段时间内, 市面上只有一本关于 Autotools 的书, 也就是传说中的“山羊书”,
(The Goat Book), `GNU Autoconf, Automake, and Libtool`_ 。整本书都可以在网
上看到, 但是这本书写得不算易懂, 而且出版于 2000 年, 许多内容已经过时了。

.. _GNU Autoconf, Automake, and Libtool: http://sources.redhat.com/autobook/

2010 年 Oreilly 出版了 `Autotools - A Practioner's Guide to GNU Autoconf,
Automake, and Libtool`_ 。这本书的写作风格更加易懂, 也更能反映 autotools 最新的
发展。本文就是基于这本书写的。

.. _Autotools - A Practioner's Guide to GNU Autoconf, Automake, and Libtool: http://oreilly.com/catalog/9781593272067
