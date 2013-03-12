---
layout: post
title: DBus glib 各数据类型接收与发送详解—C语言（3）
date: 2012-03-26 18:41
comments: true
categories: dbus-glib
---

上一篇讨论了高级数据类型的传递，这次我们就讨论更难一点的， **复杂数据类型** 的传递。为什么说复杂呢？因为它是高级数据类型的杂揉，本来高级数据类型就挺难的了，再杂揉一下，不用活了。

同样先给出 Python 编写的服务与测试
<!--more-->
## Python DBus 服务进程  
{% include_code dbus/more_advanced_data_deliver_service.py %}

## Python 测试服务
{% include_code dbus/more_advanced_data_deliver_test_py.py %}

## 使用 C 实现复杂数据类型的传递
以下代码仅仅为了演示数据类型的传递，不保证没有内存泄漏，请仔细检查后再使用。
### STRUCT_ARRAY
这次我们要传递的是结构体数组 "a(si)"。

因为没有 "(si)" 类型，所以我们自己定义。同样因为没有 "a(si)"，所以我们也自己定义。那么接下来如代码所示，就可以进行传递了。

只要知道哪种数据与哪种类型对应后，就不难了。难就难在不知道该与哪种数据类型对应，同时又对 dbus-glib 与 glib 不熟，这样的话，真的是比较头痛的一件事。
{% codeblock lang:c %}
#define DBUS_STRUCT_STRING_INT (                         \
        dbus_g_type_get_struct ( "GValueArray", G_TYPE_STRING,  \
                                 G_TYPE_INT, G_TYPE_INVALID))
#define DBUS_ARRAY_STRUCT_STRING_INT ( \
        dbus_g_type_get_collection("GPtrArray", DBUS_STRUCT_STRING_INT) )

int send_recv_struct_array(DBusGProxy *proxy)
{
    gchar *method;
    GError *error = NULL;
    GPtrArray *gparray, *ret;
    GValueArray *garray[3], *tmp_garray;
    GValue gval[3][2] = {{{0}}};
    GValue *tmp_gval;
    gchar *str[3] = {"apple", "banana", "cherry"};
    gint num[3] = {1, 2, 5};
    int i, j;

    for (i = 0; i < 3; i++) {
        g_value_init (&gval[i][0], G_TYPE_STRING);
        g_value_set_string(&gval[i][0], str[i]);
        g_value_init (&gval[i][1], G_TYPE_INT);
        g_value_set_int(&gval[i][1], num[i]);
    }
    
    gparray = g_ptr_array_new();
    for (i = 0; i < 3; i++) {
        garray[i] = g_value_array_new(0);
        for (j = 0; j < 2 ; j++) {
            g_value_array_append(garray[i], &gval[i][j]);
        }
        g_ptr_array_add(gparray, garray[i]);
    }

    method = "StructArrayPrint";
    if (!dbus_g_proxy_call(proxy, method, &error,
                           DBUS_ARRAY_STRUCT_STRING_INT, gparray,
                           G_TYPE_INVALID,
                           DBUS_ARRAY_STRUCT_STRING_INT, &ret,
                           G_TYPE_INVALID)) {
        g_printerr("call %s failed: %s\n", method, error->message);
        g_error_free(error);
        error = NULL;
        return -1;
    }

    for (i = 0; i < ret->len; i++) {
        tmp_garray = g_ptr_array_index(ret, i);
        tmp_gval = g_value_array_get_nth(tmp_garray, 0);
        g_print("%s: ", g_value_get_string(tmp_gval));
        tmp_gval = g_value_array_get_nth(tmp_garray, 1);
        g_print("%d\n", g_value_get_int(tmp_gval));
    }
    g_print("=================================\n\n");

    return 0;
}
{% endcodeblock %}
*** DICT_DICT
下面演示的是一个 "a{sv}" 的数据类型，特别的是这里的 "v" 我们用它再来容纳一个 "a{ss}" 数据类型。这样的话是不是有点复杂了哇？

源代码如下，俗话说，源代码上没有任何能够隐藏的秘密，有这句话吧?
{% codeblock lang:c %}
#define DBUS_TYPE_G_STRING_VALUE_HASHTABLE                             \
    dbus_g_type_get_map ( "GHashTable", G_TYPE_STRING, G_TYPE_VALUE)

int send_recv_dictdict(DBusGProxy *proxy)
{
    int i;
    char *method;
    GHashTable *table, *ret, *subtable;
    GHashTableIter iter, subiter;
    gpointer key, value, subkey, subvalue;
    GError *error = NULL;
    GValue gval[2] = {{0}};
    gchar *table_value[2][3] = { {"renhao", "24", "male"},
                                {"wenfeng", "22", "female"}};

    table = g_hash_table_new(NULL, NULL);

    for (i = 0; i < 2; i++) {
        g_value_init(&gval[i], DBUS_TYPE_G_STRING_STRING_HASHTABLE);
        g_value_take_boxed(&gval[i], 
                           dbus_g_type_specialized_construct(
                               DBUS_TYPE_G_STRING_STRING_HASHTABLE));
        subtable = g_value_get_boxed(&gval[i]);
        g_hash_table_insert(subtable, "name", table_value[i][0]);
        g_hash_table_insert(subtable, "age", table_value[i][1]);
        g_hash_table_insert(subtable, "gender", table_value[i][2]);
    }

    g_hash_table_insert(table, "fanrenhao", &gval[0]);
    g_hash_table_insert(table, "liwenfeng", &gval[1]);

    method = "DictDictPrint";
    if (!dbus_g_proxy_call(proxy, method, &error,
                           DBUS_TYPE_G_STRING_VALUE_HASHTABLE, table,
                           G_TYPE_INVALID,
                           DBUS_TYPE_G_STRING_VALUE_HASHTABLE, &ret,
                           G_TYPE_INVALID)) {
        g_printerr("call %s failed: %s\n", method, error->message);
        g_error_free(error);
        error = NULL;
        return -1;
    }

    g_print("receive: dictionary\n");
    g_hash_table_iter_init(&iter, ret);
    while (g_hash_table_iter_next(&iter, &key, &value)) {
            g_print("%s:\n", (char *)key);
            subtable = g_value_get_boxed(value);
            g_hash_table_iter_init(&subiter, subtable);
            while (g_hash_table_iter_next(&subiter, &subkey, &subvalue)) {
                g_print("%s, %s\n", (char *)subkey, (char *)subvalue);
            }
            g_print("---------------------------------\n");
        }
    g_print("=================================\n\n");

    return 0;
}
{% endcodeblock %}

###  ObjectPath_Dict_Struct_Array
这是一个 "a(oa{sv})" 的数据类型。也就是说首先要定义一个 "a{sv}" 的数据类型， 再由 "a{sv}" 定义一个 "(oa{sv})"，最后再定义 "a(oa{sv})" 的数据类型。这很复杂吧，现实中真的传递过这样复杂的数据吗？ 真的出现过，就在 **connman** (connect manager 类似 network-manager 的东东) 的服务进程中！ 我就是因为它才接触到了 D-Bus, 它的 "a(oa{sv})" 真的是害得我不浅，所以才有了这篇文章。

具体代码如下：
{% codeblock lang:c %}
int send_recv_objectpath_dict_struct_array(DBusGProxy *proxy)
{
    //这个当成是期末考试的试题吧 ^_^
    //好吧，我承认是我懒了
    return 0;
}
{% endcodeblock %}

### C D-Bus 测试完整代码
{% include_code dbus/more_advanced_data_deliver_test_c.c %}
## Makefile
见上篇
