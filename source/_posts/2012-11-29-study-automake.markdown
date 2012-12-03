---
layout: post
title: "Step-By-Step Example Of Autotools For Beginner"
date: 2012-12-03 13:02
comments: true
categories: [linux-tools, autotools]
---

This is a simple autotools example for beginner. You can easily follow it. After that you should be able to use autotools for your own project without complex features. This example shows how to use autoconf/automake, how to edit configure.ac/Makefile.am and how to disable default CFLAGS='-g -O2' for debug etc. OK, let's do it!

autoconf version: 2.68  
automake version: 1.11.3  
Ubuntu 12.04

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
{% codeblock %}
airead@airead:/tmp/summary-1.0.0$ ls
doc  man  script  src
airead@airead:/tmp/summary-1.0.0$ autoscan
airead@airead:/tmp/summary-1.0.0$ ls
autoscan.log  configure.scan  doc  man  script  src
airead@airead:/tmp/summary-1.0.0$ 
airead@airead:/tmp/summary-1.0.0$ mv configure.scan configure.ac
airead@airead:/tmp/summary-1.0.0$ ls
autoscan.log  configure.ac  doc  man  script  src
{% endcodeblock %}
autoscan produces two files **autoscan.log** and **configure.scan**. Then we rename configure.scan to configure.ac which will be used by **autoconf**.

The configure is not perfect, so we should edit it.
{% codeblock lang:sh %}
mv
AC_INIT([FULL-PACKAGE-NAME], [VERSION], [BUG-REPORT-ADDRESS])
to
AC_INIT([summary], [1.0.0], [fgh1987168@gmail.com]) #or your E-mail
{% endcodeblock %}

### Generating configure, Use **autoconf**
{% codeblock %}
airead@airead:/tmp/summary-1.0.0$ ls
autoscan.log  configure.ac  doc  man  script  src
airead@airead:/tmp/summary-1.0.0$ autoconf
airead@airead:/tmp/summary-1.0.0$ ls
autom4te.cache  configure     doc  script
autoscan.log    configure.ac  man  src
{% endcodeblock  %}
autoconf produces shell script **configure** used by user and autom4te.cache which you can safely remove it. We already have **configure**, but where is the **Makefile**?

## Automake
### The Relationship Between Makefile.am, Makefile.in And Makefile
Makefile.am is a programmer-defined file and is used by automake to generate the Makefile.in file. The ./configure script will use the Makefile.in to generate a Makefile.

### Create Makefile.am
We now prepare the Makefile.am file for each of these directories. Automake will step into each of them and produce the corresponding Makefile.in file.

`airead@airead:/tmp/summary-1.0.0$ vim Makefile.am`
{% codeblock lang:make%}
AUTOMAKE_OPTIONS = foreign
SUBDIRS = doc man script src
{% endcodeblock %}
the first line sets automake mode. "foreign" means not GNU, and is common for avoiding boring messages about files organized differently from what gnu expects.  
The second line shows a list of subdirectories to descend for further work.

`airead@airead:/tmp/summary-1.0.0$ vim man/Makefile.am`
{% codeblock lang:make %}
man_MANS = summary.8
{% endcodeblock %}
In general, the uppercase, suffix part like "_PROGRAMS" is called primary and tells partially what to perform on the argument; the lowecase, prefix (it's not given a name) tells the directory where to install. So **man_MANS** installs manuals at $(PREFIX)/share/man.

`airead@airead:/tmp/summary-1.0.0$ vim script/Makefile.am`
{% codeblock lang:make %}
bin_SCRIPTS = summary.sh
{% endcodeblock %}
installs scripts in $(PREFIX)/bin

`airead@airead:/tmp/summary-1.0.0$ vim doc/Makefile.am`
{% codeblock lang:make %}
docdir = $(datadir)/doc/@PACKAGE@
doc_DATA = OVERVIEW
{% endcodeblock %}
if "abc" is wanted for prefix, "abcdir" is to be specified. E.g. the code above expands to $(PREFIX)share/doc/summary ("@PACKAGE@" will be expanded by autoconf when producing the final Makefile, see below). $(datadir) is known by all configure scripts it generates, $(PREFIX)/share.

Now let's create src/Makefile.am, it's a little complex.
`airead@airead:/tmp/summary-1.0.0$ vim src/Makefile.am`
{% codeblock lang:make %}
SUBDIRS=lib person
{% endcodeblock %}
it shows a list of subdirectories to descend for further work. Continue to create...

`airead@airead:/tmp/summary-1.0.0$ vim src/lib/Makefile.am`
{% codeblock lang:make %}
noinst_LIBRARIES=libutils.a
libutils_a_SOURCES=lib.c lib.h
{% endcodeblock %}
The special prefix **noinst\_** indicates that the objects in question should be built but not installed at all. Each _LIBRARIES variable is a list of the libraries to be built. So **noinst_LIBRARIES** build libutils.a but not install. **\_SOURCES** list sources of libutils.a.   
Extra objects can be added to a library using the **library_LIBADD** variable.

`airead@airead:/tmp/summary-1.0.0$ vim src/person/Makefile.am`
{% codeblock lang:make %}
bin_PROGRAMS = summary
summary_SOURCES = main.c main.h do.c do.h
summary_LDADD = ../lib/libutils.a
{% endcodeblock %}
Variables that end with _PROGRAMS are special variables that list programs that the resulting Makefile should build. **bin_PROGRAMS** indicates that program summary should be built and install in $(PREFIX)/bin. its sources shows in **summary_LDADD**. **summary_LDADD** indicates that the program needs libutils.a when linking.

### Generating Makefile.in, Use **automake**
let's run **automake**:
{% codeblock %}
airead@airead:/tmp/summary-1.0.0$ automake
configure.ac: no proper invocation of AM_INIT_AUTOMAKE was found.
configure.ac: You should verify that configure.ac invokes AM_INIT_AUTOMAKE,
configure.ac: that aclocal.m4 is present in the top-level directory,
configure.ac: and that aclocal.m4 was recently regenerated (using aclocal).
automake: no `Makefile.am' found for any configure output
automake: Did you forget AC_CONFIG_FILES([Makefile]) in configure.ac?
{% endcodeblock %}
So we should edit configure.ac again.

## Integrating The Checking (Autoconf) Part And The Building (Automake) Part
Now we should reautoscan in summary for extra information.
`airead@airead:/tmp/summary-1.0.0$ diff configure.ac configure.scan`
{% codeblock lang:diff%}
5c5
< AC_INIT([summary], [1.0.0], [fgh1987168@gmail.com])
---
> AC_INIT([FULL-PACKAGE-NAME], [VERSION], [BUG-REPORT-ADDRESS])
9,10d8
< AM_INIT_AUTOMAKE(summary, 1.0.0)
< 
12a11,15
> AC_PROG_INSTALL
> AC_PROG_AWK
> AC_PROG_RANLIB
> AC_PROG_CPP
> AC_PROG_MKDIR_P
22a26,32
> AC_CONFIG_FILES([Makefile
>                  doc/Makefile
>                  man/Makefile
>                  script/Makefile
>                  src/Makefile
>                  src/lib/Makefile
>                  src/person/Makefile])
{% endcodeblock %}
we found that the configure.ac misses the AC_* and AC\_CONFIG\_FILES([*]), so add those to configure.ac. after that, run following command:
{% codeblock %}
airead@airead:/tmp/summary-1.0.0$ aclocal
airead@airead:/tmp/summary-1.0.0$ autoheader 
airead@airead:/tmp/summary-1.0.0$ autoconf
airead@airead:/tmp/summary-1.0.0$ automake --add-missing
configure.ac:9: installing `./install-sh'
configure.ac:9: installing `./missing'
src/lib/Makefile.am: installing `./depcomp'
airead@airead:/tmp/summary-1.0.0$ 
{% endcodeblock %}
Now we have one usable configure script.

## References
 - [Autotools Tutorial for Beginners](http://markuskimius.wikidot.com/programming:tut:autotools/)
 - [A tutorial for porting to autoconf & automake](http://mij.oltrelinux.com/devel/autoconf-automake/)
 - [automake manual](http://www.gnu.org/software/automake/manual/automake.html)