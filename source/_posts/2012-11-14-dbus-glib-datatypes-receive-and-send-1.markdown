---
layout: post
title: DBus glib 各数据类型接收与发送详解—C语言（1）
date: 2012-03-26 14:02
comments: true
categories: dbus-glib
---
## 动机
说到 **DBus** 用过的人大概都能明白其工作的流程。典型的使用流程是，向 DBus 服务进程发送数据，然后接收其返回的数据。简单的说，就像调用函数一样，向服务进程发送数据就相当于函数的参数，其返回的数据就相当于函数返回的结果。虽然明白了流程，但想要使用 **C语言** 通过已有的 DBus 服务进行操作，仍然是一项不太容易的工作（对像我这样的菜鸟 ^_^ ），因为数据的类型真是太多了, 使用 **Python** 会简单一点。简单点的有 **Boolean**, **Byte**, **Int32**, **Int64**, **String**, **ObjectPath**, **Signature** 等; 复杂一点的有 **Array**, **Struct**, **Dict** 等。如果不能弄清楚它们之间的联系，那么将是一件非常头痛的事。为了使我研究的结果不被淡忘，于是有了这篇文章。

<!--more-->
## 前置知识
 - 能够熟练使用 C语言；
 - 了解 DBus 各数据类型的表示, 参考 [D-Bus Specification](http://dbus.freedesktop.org/doc/dbus-specification.html)
 - 对 DBus-glib 有基本的了解，能够与 DBus 服务进程进行简单的交互。
 - 简单使用 d-feet, 参考 [D-Bus 实例讲解](http://blog.csdn.net/fmddlmyy/article/details/3585730) 
 - 大概对 Python 有些了解（只是为了说明我的分析思路，如果你只想找 C 的解决方法，那完全可以不了解）；
 - 简单了解 python dbus

对了，编译的时候要加上 dbus-glib 库，在本篇的最后会给出一个 Makefile 文件，把它放到要编译的文件的目录下，直接 make 应该就可以了，感觉说的不清楚，不过懂的话应该是懂的(-_-b)

## Python DBus 的简单演示
### Python DBus 服务进程
使用 Python 编写 DBus 服务进程是比较舒心的一件事。那么废话不多说，先来一个 "1+1=2" 的例子
{% include_code dbus/oneonetwo_service.py %} 

简单说明一下关键点：

 -  **@dbus.service.method('airead.fan.Example', in_signature='ii', out_signature='i')** 声明了一个 DBus 服务进程的一个方法，其中 **airead.fan.Example** 是接口， **in_signature='ii'** 说明该方法需要两个输入参数且者为 Int32 类型， **out_signature='i'** 说明该方法会输出一个参数且为 Int32 类型。
 -  **def IntArrayPrint(self, num1, num2)** 定义了接收到2个参数后的处理函数
 -  **name = dbus.service.BusName("airead.fan.Example", session_bus)** 取得 DBus 的 **well-known Bus name** 。
 -  **object = Example(session_bus, '/airead/fan/Example')** 将定义的 **class Example** 注册到 DBus 上。

### 调用 DBus 服务进程的方法
{% include_code dbus/oneonetwo_client.py %} 
看注释基本就可以了。

给 .py 添加可执行权限，先运行 service ,再运行 client 看结果，记得开两个shell。

## 所有基本数据类型演示
简单说一下我的思路，因为 D-Bus, glib 和 DBus-glib binding 中数据的类型真的是太多了，而我又没有系统的研究过它们三者的任何一个，所以各种数据类型的传递都是一点一点来试验的。因为 Python 使用起来简单，能够保证程序的正确性，我都是先用 Python 编写满足条件的 D-Bus 服务进程，再用 Python 编写该服务进程的测试用例，最后才开始使用C语言来发送和接收各种数据类型。所以后面就不对 Python 进行解释，直接分析 C 代码。
### 基本数据类型服务进程 (py)
{% include_code dbus/all_basic_data_deliver_service.py %}

### Python DBus 测试代码
{% include_code dbus/all_basic_data_deliver_client.py %}
我还写了一个比较料的 shell 脚本用来全面的进行测试。 
{% include_code dbus/all_basic_data_deliver_test_py.sh %}

### 使用 C 实现基本数据类型的传递
使用 C 来进行基本数据类型的传递还是比较简单的。大概可以分为两类：传递 **实体**（也就是没有用指针表示） 与 传递 **指针**（也就是使用指针表示）。说得也不是很清楚，举个例子，就像你定义一个字符是用 char c = 'b'; 定义一个字符串是用 char str[] = "AireadFan" 的区别一样。

Boolean, byte, int, uint 等属于 **实体**; String, ObjectPath, Signature 属于 **指针** 。下面是具体代码，最后会给出整个测试代码及 shell 脚本。

#### Boolean
Boolean: glib->gboolean, G_TYPE_BOOLEAN; D-Bus->'b';

解释一下，就是说 Boolean 类型，在 D-Bus glib binding 中使用 gboolean 声名，在使用类似 dbus_g_proxy_call() 函数传递参数时使用 G_TYPE_BOOLEAN, 在服务进程或 XML 声名时使用 'b'。 注意：以后将不再进行说明！

那么来看一下 Boolean 是怎么传递的吧。

{% codeblock lang:c %}
int send_recv_boolean(DBusGProxy *proxy, char *method, char *value)
{
    gboolean bool, ret;
    GError *error = NULL;

    if (!strcmp(value, "False")) {
        bool = FALSE;
    } else {
        bool = TRUE;
    }
    if (!dbus_g_proxy_call(proxy, method, &error,
                           G_TYPE_BOOLEAN, bool,
                           G_TYPE_INVALID,
                           G_TYPE_BOOLEAN, &ret,
                           G_TYPE_INVALID)) {
        g_printerr("call %s failed: %s\n", method, error->message);
        g_error_free(error);
        error = NULL;
        return -1;
    }
    printf("receive %d\n", ret);

    return 0;
}
{% endcodeblock %}


###* Byte
Byte: glib->guchar, G_TYPE_UCHAR, dbus->'y'
{% codeblock lang:c %}

int send_recv_byte(DBusGProxy *proxy, char *method, char *value)
{
    guchar byte, ret;
    GError *error = NULL;

    byte = value[0];

    if (!dbus_g_proxy_call(proxy, method, &error,
                           G_TYPE_UCHAR, byte,
                           G_TYPE_INVALID,
                           G_TYPE_UCHAR, &ret,
                           G_TYPE_INVALID)) {
        g_printerr("call %s failed: %s\n", method, error->message);
        g_error_free(error);
        error = NULL;
        return -1;
    }
    printf("receive %c\n", ret);
    
    return 0;
}
{% endcodeblock %}
###* Double
Double: glib->gdouble, G_TYPE_DOUBLE, dbus->'d'
{% codeblock lang:c %}
int send_recv_double(DBusGProxy *proxy, char *method, char *value)
{
    gdouble d, ret;
    GError *error = NULL;

    //double strtod(const char *nptr, char **endptr);
    d = strtod(value, NULL);

    if (!dbus_g_proxy_call(proxy, method, &error,
                           G_TYPE_DOUBLE, d,
                           G_TYPE_INVALID,
                           G_TYPE_DOUBLE, &ret,
                           G_TYPE_INVALID)) {
        g_printerr("call %s failed: %s\n", method, error->message);
        g_error_free(error);
        error = NULL;
        return -1;
    }
    printf("receive %f\n", ret);
    
    return 0;
}

{% endcodeblock %}
###* Int
Int32: glib->gint32, G_TYPE_INT, dbus->'i'

这里要说明的是: int16, int32, int64, uint16, uint32, uint64 之间几乎都是一样的，困难不大。
{% codeblock lang:c %}

int send_recv_int32(DBusGProxy *proxy, char *method, char *value)
{
    gint32 int32, ret;
    GError *error = NULL;

    int32 = strtol(value, NULL, 10);

    if (!dbus_g_proxy_call(proxy, method, &error,
                           G_TYPE_INT, int32,
                           G_TYPE_INVALID,
                           G_TYPE_INT, &ret,
                           G_TYPE_INVALID)) {
        g_printerr("call %s failed: %s\n", method, error->message);
        g_error_free(error);
        error = NULL;
        return -1;
    }
    printf("receive %d\n", ret);
    
    return 0;
}
{% endcodeblock %}
###* String
String: glib->gchar *, G_TYPE_STRING, dbus->'s'
{% codeblock lang:c %}

int send_recv_string(DBusGProxy *proxy, char *method, char *value)
{
    gchar *str, *ret;
    GError *error = NULL;

    str = value;

    if (!dbus_g_proxy_call(proxy, method, &error,
                           G_TYPE_STRING, str,
                           G_TYPE_INVALID,
                           G_TYPE_STRING, &ret,
                           G_TYPE_INVALID)) {
        g_printerr("call %s failed: %s\n", method, error->message);
        g_error_free(error);
        error = NULL;
        return -1;
    }
    printf("receive %s\n", ret);
    
    return 0;
}

{% endcodeblock %}
###* ObjectPath
ObjectPath: glib->DBusGObjectPath *, DBUS_TYPE_G_OBJECT_PATH, dbus->'o'
{% codeblock lang:c %}
int send_recv_objectpath(DBusGProxy *proxy, char *method, char *value)
{
    //typedef gchar DBusGObjectPath;
    const DBusGObjectPath *path, *ret;
    GError *error = NULL;

    path = value;

    if (!dbus_g_proxy_call(proxy, method, &error,
                           DBUS_TYPE_G_OBJECT_PATH, path,
                           G_TYPE_INVALID,
                           DBUS_TYPE_G_OBJECT_PATH, &ret,
                           G_TYPE_INVALID)) {
        g_printerr("call %s failed: %s\n", method, error->message);
        g_error_free(error);
        error = NULL;
        return -1;
    }

    printf("receive %s\n", ret);
    
    return 0;
}
{% endcodeblock %}
###* Signature
Signature: glib->DBusGSignature *, DBUS_TYPE_G_SIGNATURE, dbus->'g'
{% codeblock lang:c %}

int send_recv_signature(DBusGProxy *proxy, char *method, char *value)
{
    //typedef gchar DBusGSignature;
    DBusGSignature *signature, *ret;
    GError *error = NULL;

    signature = value;

    if (!dbus_g_proxy_call(proxy, method, &error,
                           DBUS_TYPE_G_SIGNATURE, signature,
                           G_TYPE_INVALID,
                           DBUS_TYPE_G_SIGNATURE, &ret,
                           G_TYPE_INVALID)) {
        g_printerr("call %s failed: %s\n", method, error->message);
        g_error_free(error);
        error = NULL;
        return -1;
    }
    printf("receive %s\n", ret);
    
    return 0;
}
{% endcodeblock %}

### C D-Bus 测试完整代码及脚本
测试脚本：[**all_basic_data_deliver_client.c**](/downloads/code/dbus/all_basic_data_deliver_client.c)
## Makefile
有些东西实际上没用，我也懒得去了。
{% codeblock lang:make %}
CC	= gcc

CFLAGS	= -Wall -g
CFLAGS += $(shell pkg-config --cflags glib-2.0 )
CFLAGS += $(shell pkg-config --cflags dbus-glib-1)
#CFLAGS += $(shell pkg-config --cflags gtk+-2.0)

LDFLAGS	= 
LDFLAGS += $(shell pkg-config --libs glib-2.0)
LDFLAGS += $(shell pkg-config --libs dbus-glib-1)
#LDFLAGS += $(shell pkg-config --libs gtk+-2.0)

SOURCE =  $(wildcard *.c)
TARGETS	:= $(patsubst %.c, %, $(SOURCE))
TARGETS_OUT = common_marshaler basic_data
TARGETS := $(filter-out $(TARGETS_OUT), $(TARGETS))
TARGETS := $(addsuffix .out, $(TARGETS))

%.out: %.c
	@echo CC $< -o $@
	@$(CC) $< common_marshaler.c basic_data.c $(CFLAGS) -o $@ $(LDFLAGS)

.PHONY: all clean test marshaler

all: $(TARGETS) 

marshaler: 
	glib-genmarshal --prefix _common_marshal --header common_marshaler.list > common_marshaler.h
	glib-genmarshal --prefix _common_marshal --body common_marshaler.list > common_marshaler.c
	dbus-binding-tool --prefix=airead_fan --mode=glib-server all_basic_data_deliver_server.xml > all_basic_data_deliver_server.h

clean:
	rm -f *~ a.out *.o $(TARGETS) core.*

test:
	@echo TARGETS: $(TARGETS)

{% endcodeblock lang:make %}
