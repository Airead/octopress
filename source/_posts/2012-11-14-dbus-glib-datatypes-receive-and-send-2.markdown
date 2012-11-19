---
layout: post
title: DBus glib 各数据类型接收与发送详解—C语言（2）
date: 2012-03-26 15:59
comments: true
categories: dbus-glib
---

上一篇讨论了基本的数据类型的传递，这次我们就讨论难一点的， **高级数据类型** 的传递。这里我们会讨论四种高级的（也就是难一点的）数据类型的传递： ARRAY, STRUCT, DICT_ENTRY, VARIANT， 具体请参照 [D-Bus Specification](http://dbus.freedesktop.org/doc/dbus-specification.html)。

同样先给出 Python 编写的服务与测试（这次没有 shell 脚本了）。
<!--more-->

## Python DBus 服务进程  
{% include_code dbus/advanced_data_deliver_service.py %}

## Python 测试服务
{% include_code dbus/advanced_data_deliver_test_py.py %}
## 使用 C 实现高级数据类型的传递
以下代码仅仅为了演示数据类型的传递，不保证没有内存泄漏，请仔细检查后再使用。
### ARRAY
ARRAY: glib->garray *, G_TYPE_ARRAY, dbus->'a'

传递字节数组的话使用 "ay", 传递 ObjectPath 数组的话使用 "ao", 传递整数数组的话使用 "ai", 具体可以使用 Python 来写 service 进行测试。下面是一个传递整数数组的例子。在这个例子中有几个要点：
 - garray 的相关操作；
 - 传递 int array 时，要使用 DBUS_TYPE_G_INT_ARRAY. 关于 DBUS_TYPE_G_INT_ARRAY 是怎么来的将在后面讨论；
 - 注意传递的是指针。
{% codeblock lang:c %}
int send_recv_int_array(DBusGProxy *proxy)
{
    char *method;
    GError *error;
    GArray *garray, *ret;
    gint i, j;
    
    garray = g_array_new (FALSE, FALSE, sizeof (gint));
    for (i = 0; i < 6; i++) {
        j = i + 1;
        g_array_append_val(garray, j);
    }

    method = "IntArrayPrint";
    if (!dbus_g_proxy_call(proxy, method, &error,
                           DBUS_TYPE_G_INT_ARRAY, garray,
                           G_TYPE_INVALID,
                           DBUS_TYPE_G_INT_ARRAY, &ret,
                           G_TYPE_INVALID)) {
        g_printerr("call %s failed: %s\n", method, error->message);
        g_error_free(error);
        error = NULL;
        return -1;
    }

    g_print("receive int array:\n");
    for (i = 0; i < ret->len; i++) {
        g_print("%d ", g_array_index(ret, gint, i));
    }
    g_print("\n=================================\n\n");
    
    return 0;
}
{% endcodeblock %}

### STRUCT
STRUCT: glib->GValueArray *, ????, dbus->'()'

结构体在 D-Bus 服务进程中的声名为 (??), 其中 `?' 可以任意数据类型。比如
{% codeblock lang:c %}
// (si)
struct str_int {
    gchar *s;
    gint i;
};

// (sidb)
struct str_int_double_boolean {
    gchar *s;
    gint i;
    gdouble d;
    gboolean b;
};
{% endcodeblock %}

下面演示了 (sidb) 在 D-Bus 中的传递。代码分为三大块:
 1. define 需要的结构；
 2. 创建输入数据；
 2. 调用命令；
 3. 打印接收到的数据；
{% codeblock lang:c %}
#define DBUS_STRUCT_STRING_INT_DOUBLE_BOOLEAN (                         \
        dbus_g_type_get_struct ( "GValueArray", G_TYPE_STRING, G_TYPE_INT, \
                                 G_TYPE_DOUBLE, G_TYPE_BOOLEAN, G_TYPE_INVALID))

int send_recv_struct(DBusGProxy *proxy)
{
    char *method;
    GError *error = NULL;
    GValueArray *ret, *send_array;
    GValue *gval;
    GValue send_gval[4] = {{0}};
    int i;
    
    g_value_init (&send_gval[0], G_TYPE_STRING);
    g_value_set_string(&send_gval[0], "fan");
    g_value_init (&send_gval[1], G_TYPE_INT);
    g_value_set_int(&send_gval[1], 24);
    g_value_init (&send_gval[2], G_TYPE_DOUBLE);
    g_value_set_double(&send_gval[2], 70.2);
    g_value_init (&send_gval[3], G_TYPE_BOOLEAN);
    g_value_set_boolean(&send_gval[3], FALSE);
    
    send_array = g_value_array_new(0);
    for (i = 0; i < 4; i++) {
        send_array = g_value_array_append(send_array, &send_gval[i]);
    }

    method = "StructPrint";
    if (!dbus_g_proxy_call(proxy, method, &error,
                           DBUS_STRUCT_STRING_INT_DOUBLE_BOOLEAN, send_array,
                           G_TYPE_INVALID,
                           DBUS_STRUCT_STRING_INT_DOUBLE_BOOLEAN, &ret,
                           G_TYPE_INVALID)) {
        g_printerr("call %s failed: %s\n", method, error->message);
        g_error_free(error);
        error = NULL;
        return -1;
    }

    g_print("receive struct:\n");
    for (i = 0; i < ret->n_values; i++) {
        gval = g_value_array_get_nth(ret, i);
        if (G_VALUE_TYPE(gval) == G_TYPE_STRING) {
            g_print("%s\n", g_value_get_string(gval));
        } else if (G_VALUE_TYPE(gval) == G_TYPE_DOUBLE) {
            g_print("%f\n", g_value_get_double(gval));
        } else if (G_VALUE_TYPE(gval) == G_TYPE_INT) {
            g_print("%d\n", g_value_get_int(gval));
        } else if (G_VALUE_TYPE(gval) == G_TYPE_BOOLEAN) {
            g_print("%d\n", g_value_get_boolean(gval));
        }
    }

    g_print("\n=================================\n\n");
    
    return 0;
}

{% endcodeblock %}
回忆一下，在传递 ARRAY 中我们遇到了 **DBUS_TYPE_G_INT_ARRAY** , 那么它是如何定义的呢？
{% codeblock lang:c %}
#define DBUS_TYPE_G_INT_ARRAY (dbus_g_type_get_collection ("GArray", G_TYPE_INT))
{% endcodeblock %}
 它是 dbus-glib 库帮我定义的，使用到了 dbus_g_type_get_collection() 函数。当我们传递某些特殊数据的时候，如果 dbus-glib 库没有它的定义，那我们需要使用
 - dbus_g_type_get_collection()
 - dbus_g_type_get_map()
 - dbus_g_type_get_struct()

等相关函数自行定义。上面的例子中我们就定义一个 struct 类型:
{% codeblock lang:c %}
#define DBUS_STRUCT_STRING_INT_DOUBLE_BOOLEAN (                         \
        dbus_g_type_get_struct ( "GValueArray", G_TYPE_STRING, G_TYPE_INT, \
                                 G_TYPE_DOUBLE, G_TYPE_BOOLEAN, G_TYPE_INVALID))
{% endcodeblock %}
具体细节请参考 [Specializable GType System](http://dbus.freedesktop.org/doc/dbus-glib/dbus-glib-Specializable-GType-System.html)，以及详解中其它的类型定义。 map 对应 GHashTable, collection 对应 GArray, struct 对 GValueArray 等。

另外，STRUCT 数据还有另一种构造方法，在 `使用 VARINAT 传递 STRUCT' 中，我也不知道哪种方法更好。
### DICT_ENTRY
DICT_ENTRY: glib->GHashTable, ????, dbus->a{??}

参考 STRUCT 中 '?' 的示例，下面我们演示了 a{ss} 的传递代码。其中 DBUS_TYPE_G_STRING_STRING_HASHTABLE，是库帮我们定义的：
`#define DBUS_TYPE_G_STRING_STRING_HASHTABLE (dbus_g_type_get_map ("GHashTable", G_TYPE_STRING, G_TYPE_STRING))`
代码同样分为三大块，与 STUCT 流程相似。
{% codeblock lang:c %}
int send_recv_dict(DBusGProxy *proxy)
{
    char *method;
    GHashTable *table, *ret;
    GHashTableIter iter;
    gpointer key, value;
    GError *error = NULL;
    char *str[3];

    table = g_hash_table_new(NULL, NULL);

    str[0] = "apple";
    str[1] = "banana";
    str[2] = "cherry";

    g_hash_table_insert(table, "a", str[0]);
    g_hash_table_insert(table, "b", str[1]);
    g_hash_table_insert(table, "c", str[2]);

    method = "DictPrint";
    if (!dbus_g_proxy_call(proxy, method, &error,
                           DBUS_TYPE_G_STRING_STRING_HASHTABLE, table,
                           G_TYPE_INVALID,
                           DBUS_TYPE_G_STRING_STRING_HASHTABLE, &ret,
                           G_TYPE_INVALID)) {
        g_printerr("call %s failed: %s\n", method, error->message);
        g_error_free(error);
        error = NULL;
        return -1;
    }

    g_print("receive: dictionary\n");
    g_hash_table_iter_init(&iter, ret);
    while (g_hash_table_iter_next(&iter, &key, &value)) {
        g_print("key: %s, %s\n", (char *)key, (char *)value);
    }
    g_print("=================================\n\n");

    g_hash_table_unref(table);
    g_hash_table_unref(ret);

    return 0;
}
{% endcodeblock %}

### VARIANT
VARIANT: glib->GValue, G_TYPE_VALUE, dbus->'v'

VARIANT 是一个通用的容器，它可以装任意数据类型。如：所有基本类型，ARRAY, STRUCT, DICT_ENTRY 甚至容纳自身。以下代码演示了装 INT_ARRAY 和 STRUCT 的 VARIANT 的传递，它们分为两个函数。
{% codeblock lang:c %}
int send_recv_variant(DBusGProxy *proxy)
{
    char *method;
  
    method = "VariantPrint";
    
    send_recv_variant_int_array(proxy, method);
    send_recv_variant_struct(proxy, method);

    return 0;
}
{% endcodeblock %}

#### 使用 VARINAT 传递 INT_ARRAY
重点就是如何将 INT_ARRAY 装入 GValue 中。这里的办法是，先产生一个容器，然后从容器中获取指针进行赋值。试这个的时候费了我老大劲了-_-!。注意，服务进程返回的是 DICT_ENTRY 类型的数据。

{% codeblock lang:c %}
int send_recv_variant_int_array(DBusGProxy *proxy, char *method)
{
    GError *error = NULL;
    GValue gval = G_VALUE_INIT;
    GValue ret = G_VALUE_INIT;
    GHashTable *table;
    GHashTableIter iter;
    gpointer key, value;
    GArray *garray;
    gint i, j;
    
    g_value_init(&gval, DBUS_TYPE_G_INT_ARRAY);
    g_value_take_boxed(&gval, dbus_g_type_specialized_construct(DBUS_TYPE_G_INT_ARRAY));
    garray = g_value_get_boxed(&gval);
    for (i = 0; i < 6; i++) {
        j = i + 1;
        g_array_append_val(garray, j);
    }

    if (!dbus_g_proxy_call(proxy, method, &error,
                           G_TYPE_VALUE, &gval,
                           G_TYPE_INVALID,
                           G_TYPE_VALUE, &ret,
                           G_TYPE_INVALID)) {
        g_printerr("call %s failed: %s\n", method, error->message);
        g_error_free(error);
        error = NULL;
        return -1;
    }

    g_print("receive variant:\n");
    table = g_value_get_boxed(&ret);
    
    g_hash_table_iter_init(&iter, table);
    while (g_hash_table_iter_next(&iter, &key, &value)) {
        g_print("key: %s, %s\n", (char *)key, (char *)value);
    }
   
    g_print("\n=================================\n\n");

    return 0;
}
{% endcodeblock %}

#### 使用 VARINAT 传递 INT_STRUCT
重点就是如何将 STRUCT 装入 GValue 中。这里出现了构造 STRUCT 的第二种方法。注意，服务进程返回的是 DICT_ENTRY 类型的数据。

{% codeblock lang:c %}
int send_recv_variant_struct(DBusGProxy *proxy, char *method)
{
    GError *error = NULL;
    GValue gval = G_VALUE_INIT;
    GValue ret = G_VALUE_INIT;
    GHashTable *table;
    GHashTableIter iter;
    gpointer key, value;
    
    g_value_init(&gval, DBUS_STRUCT_STRING_INT_DOUBLE_BOOLEAN);
    g_value_take_boxed(&gval, dbus_g_type_specialized_construct(DBUS_STRUCT_STRING_INT_DOUBLE_BOOLEAN));
    
    dbus_g_type_struct_set(&gval, 0, "fan",
                           1, 24,
                           2, 70.1,
                           3, FALSE, G_MAXUINT);

    if (!dbus_g_proxy_call(proxy, method, &error,
                           G_TYPE_VALUE, &gval,
                           G_TYPE_INVALID,
                           G_TYPE_VALUE, &ret,
                           G_TYPE_INVALID)) {
        g_printerr("call %s failed: %s\n", method, error->message);
        g_error_free(error);
        error = NULL;
        return -1;
    }

    g_print("receive variant:\n");
    table = g_value_get_boxed(&ret);
    
    g_hash_table_iter_init(&iter, table);
    while (g_hash_table_iter_next(&iter, &key, &value)) {
        g_print("key: %s, %s\n", (char *)key, (char *)value);
    }
   
    g_print("\n=================================\n\n");

    return 0;
}
{% endcodeblock %}

### C D-Bus 测试完整代码
[advanced_data_deliver_test_c.c](/downloads/code/dbus/advanced_data_deliver_test_c.c)
## Makefile
见上篇
