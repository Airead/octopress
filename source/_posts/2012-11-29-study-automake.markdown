---
layout: post
title: "Step-By-Step Example Of Autotools For Beginner"
date: 2012-12-03 13:02
comments: true
categories: [linux-tools, autotools]
---

This is a simple autotools example for beginner. You can easily follow it. After that you should be able to use autotools for your own project without complex features. This example shows how to use autoconf/automake, how to edit configure.ac/Makefile.am and how to disable default CFLAGS='-g -O2' for debug etc. OK, let's do it!

<code>
autoconf version: 2.68  
automake version: 1.11.3  
Ubuntu 12.04
</code>

<!--more-->
## Create Sources
If you want to use autotools in your project, first of all you should have an existing one. following is an example I use. 
{% codeblock %}
    summary-1.0.0
    ├── doc             #project documents
    │   └── OVERVIEW
    ├── man             #manuals
    │   └── summary.8
    ├── script          #useful scripts
    │   └── summary.sh
    └── src             #source code
        ├── lib         #general functions
        │   ├── lib.c
        │   └── lib.h
        └── person      #one program
            ├── do.c
            ├── do.h
            ├── main.c
            └── main.h
{% endcodeblock  %}
you can [DOWNLOAD](/downloads/code/autotools/summary-1.0.0.tar.gz) this project. Of course, you can create your own project.

## Autoconf
### Generating configure.ac In summary-1.0.0, Use **autoscan**
<code>
airead@airead:/tmp/summary-1.0.0$ ls  
doc  man  script  src  
airead@airead:/tmp/summary-1.0.0$ autoscan  
airead@airead:/tmp/summary-1.0.0$ ls  
autoscan.log  configure.scan  doc  man  script  src  
airead@airead:/tmp/summary-1.0.0$   
airead@airead:/tmp/summary-1.0.0$ mv configure.scan configure.ac  
airead@airead:/tmp/summary-1.0.0$ ls  
autoscan.log  configure.ac  doc  man  script  src  
</code>  
autoscan produces two files autoscan.log and configure.scan. Then we rename **configure.scan** to **configure.ac** which will be used by **autoconf**.

The configure is not perfect, so we should edit it.
{% codeblock lang:sh %}
mv
AC_INIT([FULL-PACKAGE-NAME], [VERSION], [BUG-REPORT-ADDRESS])
to
AC_INIT([summary], [1.0.0], [fgh1987168@gmail.com]) #or your E-mail
{% endcodeblock %}

### Generating configure, Use **autoconf**
<code>
airead@airead:/tmp/summary-1.0.0$ ls  
autoscan.log  configure.ac  doc  man  script  src  
airead@airead:/tmp/summary-1.0.0$ autoconf  
airead@airead:/tmp/summary-1.0.0$ ls  
autom4te.cache  configure     doc  script  
autoscan.log    configure.ac  man  src  
</code>  
autoconf produces shell script **configure** used by user and autom4te.cache which you can safely remove it. We already have **configure**, but where is the **Makefile**?

## Automake
### The Relationship Between Makefile.am, Makefile.in And Makefile
Makefile.am is a programmer-defined file and is used by automake to generate the Makefile.in file. The ./configure script will use the Makefile.in to generate a Makefile.

### Create Makefile.am
We now prepare the Makefile.am file for each of directories. Automake will step into each of them and produce the corresponding Makefile.in file.

`airead@airead:/tmp/summary-1.0.0$ vim Makefile.am`
{% codeblock lang:sh %}
AUTOMAKE_OPTIONS = foreign
SUBDIRS = doc man script src
{% endcodeblock %}
the first line sets automake mode. "foreign" means not GNU, and is common for avoiding boring messages about files organized differently from what gnu expects.  
The second line shows a list of subdirectories to descend for further work.

`airead@airead:/tmp/summary-1.0.0$ vim man/Makefile.am`
{% codeblock lang:sh %}
man_MANS = summary.8
{% endcodeblock %}
In general, the uppercase, suffix part like "_PROGRAMS" is called primary and tells partially what to perform on the argument; the lowecase, prefix (it's not given a name) tells the directory where to install. So **man_MANS** installs manuals at $(PREFIX)/share/man.

`airead@airead:/tmp/summary-1.0.0$ vim script/Makefile.am`
{% codeblock lang:sh %}
bin_SCRIPTS = summary.sh
{% endcodeblock %}
installs scripts in $(PREFIX)/bin

`airead@airead:/tmp/summary-1.0.0$ vim doc/Makefile.am`
{% codeblock lang:sh %}
docdir = $(datadir)/doc/@PACKAGE@
doc_DATA = OVERVIEW
{% endcodeblock %}
if "abc" is wanted for prefix, "abcdir" is to be specified. E.g. the code above expands to $(PREFIX)/share/doc/summary ("@PACKAGE@" will be expanded by autoconf when producing the final Makefile, see below). $(datadir) is known by all configure scripts it generates, $(PREFIX)/share.

Now let's create src/Makefile.am, it's a little complex.
`airead@airead:/tmp/summary-1.0.0$ vim src/Makefile.am`
{% codeblock lang:sh %}
SUBDIRS=lib person
{% endcodeblock %}
it shows a list of subdirectories to descend for further work. Continue to create...

`airead@airead:/tmp/summary-1.0.0$ vim src/lib/Makefile.am`
{% codeblock lang:sh %}
noinst_LIBRARIES=libutils.a
libutils_a_SOURCES=lib.c lib.h
{% endcodeblock %}
The special prefix **noinst\_** indicates that the objects in question should be built but not installed at all. Each _LIBRARIES variable is a list of the libraries to be built. So **noinst_LIBRARIES** build libutils.a but not install. **\_SOURCES** list sources of libutils.a.   
Extra objects can be added to a library using the **library_LIBADD** variable.

`airead@airead:/tmp/summary-1.0.0$ vim src/person/Makefile.am`
{% codeblock lang:sh %}
AM_CFLAGS = -I../lib

bin_PROGRAMS = summary
summary_SOURCES = main.c main.h do.c do.h
summary_LDADD = ../lib/libutils.a
{% endcodeblock %}
Variables that end with _PROGRAMS are special variables that list programs that the resulting Makefile should build. **bin_PROGRAMS** indicates that program summary should be built and install in $(PREFIX)/bin. its sources shows in **summary_LDADD**. **summary_LDADD** indicates that the program needs libutils.a when linking.

### Generating Makefile.in, Use **automake**
let's run **automake**:
<code>
airead@airead:/tmp/summary-1.0.0$ automake  
configure.ac: no proper invocation of AM_INIT_AUTOMAKE was found.  
configure.ac: You should verify that configure.ac invokes AM_INIT_AUTOMAKE,  
configure.ac: that aclocal.m4 is present in the top-level directory,  
configure.ac: and that aclocal.m4 was recently regenerated (using aclocal).  
automake: no `Makefile.am' found for any configure output  
automake: Did you forget AC_CONFIG_FILES([Makefile]) in configure.ac?  
</code>  
So we should edit configure.ac again. add following after AC_INIT() in configure.ac.

{% codeblock %}
AM_INIT_AUTOMAKE(summary, 1.0.0)
{% endcodeblock %}

## Integrating The Checking (Autoconf) Part And The Building (Automake) Part
Now there are some new Makefile.am in each of directories, so we should reautoscan for extra information in the project.  
`airead@airead:/tmp/summary-1.0.0$ autoscan`
`airead@airead:/tmp/summary-1.0.0$ diff configure.ac configure.scan`
{% codeblock lang:diff%}
5c5
< AC_INIT([summary], [1.0.0], [fgh1987168@gmail.com])
---
> AC_INIT([FULL-PACKAGE-NAME], [VERSION], [BUG-REPORT-ADDRESS])
9,10d8
< AM_INIT_AUTOMAKE(summary, 1.0.0)
< 
22a21,27
> AC_CONFIG_FILES([Makefile
>                  doc/Makefile
>                  man/Makefile
>                  script/Makefile
>                  src/Makefile
>                  src/lib/Makefile
>                  src/person/Makefile])
{% endcodeblock %}
we found that the configure.ac misses the AC\_CONFIG\_FILES([*]), so put those above AC_OUTPUT in configure.ac. after that, run following command:

<code>
airead@airead:/tmp/summary-1.0.0$ aclocal  
airead@airead:/tmp/summary-1.0.0$ autoheader   
airead@airead:/tmp/summary-1.0.0$ autoconf  
</code>  
`aclocal` program creates the file 'aclocal.m4' by combining stock installed macros, user defined macros and the contents of 'acinclude.m4' to define all of the macros required by 'configure.ac' in a single file.  
`autoheader` runs m4 over 'configure.ac', but with key macros defined differently than when autoconf is executed, such that suitable cpp definitions are output to 'config.h.in'.  
`autoconf` expands the m4 macros in 'configure.ac', perhaps using macro definitions from 'aclocal.m4', to generate the configure script.  

Then we call automake to generate Makefile.in.
<code>
airead@airead:/tmp/summary-1.0.0$ automake --add-missing  
configure.ac:9: installing \`./install-sh'  
configure.ac:9: installing \`./missing'  
src/lib/Makefile.am:1: library used but \`RANLIB' is undefined  
src/lib/Makefile.am:1:   The usual way to define \`RANLIB' is to add \`AC_PROG_RANLIB'  
src/lib/Makefile.am:1:   to \`configure.ac' and run \`autoconf' again.  
src/lib/Makefile.am: installing \`./depcomp'  
</code>

Read the error information and fix it by puting `AC_PROG_RANLIB` after `AC_INIT_AUTOMAKE()` in configure.ac. run `automake` again.  
`airead@airead:/tmp/summary-1.0.0$ automake`

Now we have one usable configure script.

## Build Project
`airead@airead:/tmp/summary-1.0.0$ ./configure`  
configure do two things:

1. scan for dependencies on the basis of the AC_* macros instructed in configure.ac. If there's something wrong/missing in the system, an opportune error message will be dumped.  
2. for each Makefile requested in AC_OUTPUT(), translate the Makefile.in template for generating the final Makefile. The main makefile will provide the most common targets like install, clean, distclean, uninstall et al.

If configure succeeds, all the Makefile files are available. Run make.
`airead@airead:/tmp/summary-1.0.0$ make`  
If no accident, make will products src/lib/libutils.a and src/person/summary which we can execute. Run following if you like.  
`airead@airead:/tmp/summary-1.0.0$ make install`  

## Change Default CFLAGS For Debug
autoconf default CFLAGS is '-g -O2', it is inconvenient to debug program. How to change this?  
Macro `AC_PROG_CC` in configure.ac determines a C compiler to use. If using the GNU C compiler, set shell variable GCC to 'yes'. If output variable CFLAGS was not already set, set it to '-g -O2' for the GNU C compiler ('-O2' on systems where GCC does not accept '-g'), or '-g' for other compilers.  
Put `CFLAGS="-g -O0"` above `AC_PROG_CC`, then  
`airead@airead:/tmp/summary-1.0.0$ autoreconf`  
It works well.  
**note:** There is no space on either side of `=`.

## References
 - [Autotools Tutorial for Beginners](http://markuskimius.wikidot.com/programming:tut:autotools/)
 - [A tutorial for porting to autoconf & automake](http://mij.oltrelinux.com/devel/autoconf-automake/)
 - [automake manual](http://www.gnu.org/software/automake/manual/automake.html)
 - [autobook](http://www.sourceware.org/autobook/autobook/autobook.html#SEC_Top)
