---
layout: post
title: "learning nf_conntrack (1)"
date: 2013-01-15 10:16
comments: true
published: false
categories: kernel conntrack netfilter
---

# nf_conntrack_standalone.c
研究 conntrack, 首先我们从它的初始化开始。 
`kernel 2.6.32.59 net/netfilter/nf_conntrack_standalone.c`

通过 `module_init(nf_conntrack_standalone_init);` 可以发现 nf_conntrack 的初始化函数是 `nf_conntrack_standalone_init`. 它注册了 `nf_conntrack_net_ops` 结构体. 先不管它是如何注册的,结构体都包含了哪些函数.

{% codeblock lang:c %}
static struct pernet_operations nf_conntrack_net_ops = {  
	.init = nf_conntrack_net_init,  
	.exit = nf_conntrack_net_exit,  
};  
{% endcodeblock %}

其中 `nf_conntrack_net_init(struct net *net)` 是回调函数,它有一个参数是 `struct net *net` 那么这个参数是从哪来的? 这就需要看 `register_pernet_subsys(&nf_conntrack_net_ops);` 这个函数了. 它在 `net_namespace.c` 中,在这个文件中有以下全局变量.

<!--more-->
{% codeblock lang:c %}
/*
 *	Our network namespace constructor/destructor lists
 */

/* 声名一个 device list. It's not a pointer, but a entity. (struct
 * pernet_operations) will be added here. */
static LIST_HEAD(pernet_list);
/* make device meaningful */
static struct list_head *first_device = &pernet_list;
static DEFINE_MUTEX(net_mutex);

/* for_each_net() will traversal this */
LIST_HEAD(net_namespace_list);
EXPORT_SYMBOL_GPL(net_namespace_list);

/* #ifndef CONFIG_NET_NS, only use this */
struct net init_net;
EXPORT_SYMBOL(init_net);

#define INITIAL_NET_GEN_PTRS	13 /* +1 for len +2 for rcu_head */
{% endcodeblock%}

`nf_connrack_net_init()` 在 `register_pernet_operations(struct list_head *list, struct pernet_operation *ops)` 中被调用, 其中 ops 就是上面提到的 `nf_conntrack_net_ops` 了. 现在,知道了参数 `struct net *net` 是从哪来的了,那就继续看 `nf_conntrack_net_init()` 了.它的动作是

1. call nf_conntrack_init(net) to initialize conntrack
2. call nf_conntrack_standalone_init_proc() to generate proc file
3. call nf_conntrack_standalone_init_sysctl() to generate sys file

## nf_conntrack_init()
<code>
1. 算出 nf_conntrack_htable_size and nf_conntrack_max  
2. call init function: nf_conntrack_proto_init() and nf_conntrack_helper_init();
3. configure nf_conntrack_untracked  
</code>

nf_conntrack_proto_init() 注册一个 l4 通用协议, 通过 call nf_ct_l4proto_register_sysctl(&nf_conntrack_l4proto_generic), nf_conntrack_l4_proto_generic 的内容如下:

{% codeblock lang:c %}
struct nf_conntrack_l4proto nf_conntrack_l4proto_generic __read_mostly =
{
	.l3proto		= PF_UNSPEC,
	.l4proto		= 255,
	.name			= "unknown",
	.pkt_to_tuple		= generic_pkt_to_tuple,
	.invert_tuple		= generic_invert_tuple,
	.print_tuple		= generic_print_tuple,
	.packet			= packet,
	.new			= new,
#ifdef CONFIG_SYSCTL
	.ctl_table_header	= &generic_sysctl_header,
	.ctl_table		= generic_sysctl_table,
#ifdef CONFIG_NF_CONNTRACK_PROC_COMPAT
	.ctl_compat_table	= generic_compat_sysctl_table,
#endif
#endif
};
{% endcodeblock %}

nf_ct_register_sysctl() call nf_ct_register_sysctl() [in kernel/sysctl.c] 不作深入分析，可参照（sysctl 文件系统）。从以上可以猜出, nf_ct_l4proto_register_sysctl(&nf_conntrack_l4proto_generic) 只是注册了 sysctl 和 proc(可能猜错). 

现在开始分析 nf_conntrack_helper_init(). 它调用 nf_ct_alloc_hashtable(&nf_ct_helper_hsize, &nf_ct_helper_vmalloc, 0); 来创建一个 hash 表. nf_conntrack_helper.c 里的全局变量有
{% codeblock lang:c %}
static DEFINE_MUTEX(nf_ct_helper_mutex);
/* conntrack helper hash table, alloc by nc_conntrack_helper_init() */
static struct hlist_head *nf_ct_helper_hash __read_mostly;
/* alloc by nc_conntrack_helper_init() */
static unsigned int nf_ct_helper_hsize __read_mostly;
static unsigned int nf_ct_helper_count __read_mostly;
/* tag: get memory by vmalloc() or not, 0 for no, alloc by
 * nc_conntrack_helper_init() */
static int nf_ct_helper_vmalloc;
{% endcodeblock %}

nf_ct_alloc_hashtable 分别设置了 nf_ct_helper_hsize 和 nf_ct_helper_vmalloc. 接下来将 helper 注册进扩展(at nf_conntrack_extend.c). call nf_ct_extend_register(&helper_extend); 其中 helper_extend 如下:

{% codeblock lang:c %}
static struct nf_ct_ext_type helper_extend __read_mostly = {
	.len	= sizeof(struct nf_conn_help),
	.align	= __alignof__(struct nf_conn_help),
	.id	= NF_CT_EXT_HELPER,
};
{% endcodeblock %}

目前扩展的类型主要有

{% codeblock lang:c %}
enum nf_ct_ext_id
{
	NF_CT_EXT_HELPER,
	NF_CT_EXT_NAT,
	NF_CT_EXT_ACCT,
	NF_CT_EXT_ECACHE,
	NF_CT_EXT_NUM,
};

#define NF_CT_EXT_HELPER_TYPE struct nf_conn_help
#define NF_CT_EXT_NAT_TYPE struct nf_conn_nat
#define NF_CT_EXT_ACCT_TYPE struct nf_conn_counter
#define NF_CT_EXT_ECACHE_TYPE struct nf_conntrack_ecache
{% endcodeblock %}

最后 call update_alloc_size(type); 对 type->alloc_size 进行最后的更新.

after nf_conntrack_init_init_net(), let's step into nf_conntrack_init_net(net). 它主要是对 net->ct 这个结构进行初使化.
1. 初始化 hlist_nulls_head net->ct.unconfirmed 和 net->ct.dying, 
2. alloc_percpu for net->ct.stat
3. set net->ct.slabname
4. create net->ct.nf_conntrack_cachep
5. set hash table size, net->ct.hash_size
6. create hash table, net->ct.hash
7. nf_conntrack_expect_init(net);
8. nf_conntrack_acct_init(net);
9. nf_conntrack_ecache_init(net);

`nf_conntrack_expect_init` 
1. 算出 nf_conntrack_except_hsize
2. create hash table by kmem_cache_create()
3. generate proc files

acct 与 ecache 跟以前提到的 nf_conntrack_helper_init 相似.

## nf_conntrack_standalone_init_proc(net)

## nf_conntrack_standalone_init_sysctl(net);

# nf_conntrack_l3proto_ipv4.c
打开 net/ipv4/netfilter/nf_conntrack_l3proto_ipv4.c, 我们找到了 `module_init(nf_conntrack_l3proto_ipv4_init);`, 那这个模块的初始化函数为 `nf_conntrack_l3proto_ipv4_init`.  它的动作是
1. need_conntrack(); 确认依赖 nf_conntrack
2. nf_defrag_ipv4_enable(); 确认依赖 nf_defrag_ipv4
3. nf_register_sockopt(&so_getorigdst); 注册 socket opt, 先不管它
4. nf_conntrack_l4proto_register(&nf_conntrack_l4proto_tcp4); 注册 tcp4 协议
5. nf_conntrack_l4proto_register(&nf_conntrack_l4proto_udp4);
6. nf_conntrack_l4proto_register(&nf_conntrack_l4proto_icmp);
7. nf_conntrack_l3proto_register(&nf_conntrack_l3proto_ipv4); **注册 l3 proto ipv4**
8. int nf_register_hooks(struct nf_hook_ops *reg, unsigned int n) 注册 netfilter 挂载点, 这是浏览 conntrack 的主线.
9. nf_conntrack_ipv4_compat_init(); 注册 proc file

## nf_conntrack_l4proto_register()
先看一下 nf_conntrack_l4proto_tcp4 (at net/netfilter/nf_conntrack_proto_tcp):
{% codeblock lang:c %}
struct nf_conntrack_l4proto nf_conntrack_l4proto_tcp4 __read_mostly =
{
	.l3proto		= PF_INET,
	.l4proto 		= IPPROTO_TCP,
	.name 			= "tcp",
	.pkt_to_tuple 		= tcp_pkt_to_tuple,
	.invert_tuple 		= tcp_invert_tuple,
	.print_tuple 		= tcp_print_tuple,
	.print_conntrack 	= tcp_print_conntrack,
	.packet 		= tcp_packet,
	.new 			= tcp_new,
	.error			= tcp_error,
#if defined(CONFIG_NF_CT_NETLINK) || defined(CONFIG_NF_CT_NETLINK_MODULE)
	.to_nlattr		= tcp_to_nlattr,
	.nlattr_size		= tcp_nlattr_size,
	.from_nlattr		= nlattr_to_tcp,
	.tuple_to_nlattr	= nf_ct_port_tuple_to_nlattr,
	.nlattr_to_tuple	= nf_ct_port_nlattr_to_tuple,
	.nlattr_tuple_size	= tcp_nlattr_tuple_size,
	.nla_policy		= nf_ct_port_nla_policy,
#endif
#ifdef CONFIG_SYSCTL
	.ctl_table_users	= &tcp_sysctl_table_users,
	.ctl_table_header	= &tcp_sysctl_header,
	.ctl_table		= tcp_sysctl_table,
#ifdef CONFIG_NF_CONNTRACK_PROC_COMPAT
	.ctl_compat_table	= tcp_compat_sysctl_table,
#endif
#endif
};
{% endcodeblock %}

.error() 对数据包进行正确性检查
.packet() 返回失败, 则释放连接
.pkt_to_tuple() 根据 skb 填充 nf_conntrack_tuple
.invert_tuple() 反转 tuple
.new() 创建一个新的连接
.destory() 释放一个连接
.print_tuple() 打印 tuple 信息

## nf_conntrack_l3proto_register()
首先看一下 nf_conntrack_l3proto_ipv4 (at net/ipv4/netfilter/nf_conntrack_l3proto_ipv4.c):
{% codeblock lang:c %}
struct nf_conntrack_l3proto nf_conntrack_l3proto_ipv4 __read_mostly = {
	.l3proto	 = PF_INET,
	.name		 = "ipv4",
	.pkt_to_tuple	 = ipv4_pkt_to_tuple,
	.invert_tuple	 = ipv4_invert_tuple,
	.print_tuple	 = ipv4_print_tuple,
	.get_l4proto	 = ipv4_get_l4proto,
#if defined(CONFIG_NF_CT_NETLINK) || defined(CONFIG_NF_CT_NETLINK_MODULE)
	.tuple_to_nlattr = ipv4_tuple_to_nlattr,
	.nlattr_tuple_size = ipv4_nlattr_tuple_size,
	.nlattr_to_tuple = ipv4_nlattr_to_tuple,
	.nla_policy	 = ipv4_nla_policy,
#endif
#if defined(CONFIG_SYSCTL) && defined(CONFIG_NF_CONNTRACK_PROC_COMPAT)
	.ctl_table_path  = nf_net_ipv4_netfilter_sysctl_path,
	.ctl_table	 = ip_ct_sysctl_table,
#endif
	.me		 = THIS_MODULE,
};
{% endcodeblock %}

.pkt_to_tuple, inver_tuple, print_tuple 与以前提到的功能相同，现在重点关注一下 .get_l4proto, `int (*get_l4proto)(const struct sk_buff *skb, unsigned int nhoff, unsigned int *dataoff, u_int8_t *protonum)`， 它通过参数 skb, nhoff 取得数据(dataoff)和协议(protonum)。.get_l4_proto 被调用在 nf_conntrack_in() at net/netfilter/nf_conntrack_core.c. 而 `nf_conntrack_in()` 是在哪注册进内核协议栈的呢？ 它是在 ipv4_conntrack_in() 和 ipv4_conntrack_local() 中被调用的. ipv4_conntrack_in() 和 ipv4_conntrack_local() 是通过以下结构注册进 netfilter 框架内, 并最终被内核协议栈所调用. 

{% codeblock lang:c %}
/* Connection tracking may drop packets, but never alters them, so
   make it the first hook. */
static struct nf_hook_ops ipv4_conntrack_ops[] __read_mostly = {
	{
		.hook		= ipv4_conntrack_in,
		.owner		= THIS_MODULE,
		.pf		= NFPROTO_IPV4,
		.hooknum	= NF_INET_PRE_ROUTING,
		.priority	= NF_IP_PRI_CONNTRACK,
	},
	{
		.hook		= ipv4_conntrack_local,
		.owner		= THIS_MODULE,
		.pf		= NFPROTO_IPV4,
		.hooknum	= NF_INET_LOCAL_OUT,
		.priority	= NF_IP_PRI_CONNTRACK,
	},
	{
		.hook		= ipv4_confirm,
		.owner		= THIS_MODULE,
		.pf		= NFPROTO_IPV4,
		.hooknum	= NF_INET_POST_ROUTING,
		.priority	= NF_IP_PRI_CONNTRACK_CONFIRM,
	},
	{
		.hook		= ipv4_confirm,
		.owner		= THIS_MODULE,
		.pf		= NFPROTO_IPV4,
		.hooknum	= NF_INET_LOCAL_IN,
		.priority	= NF_IP_PRI_CONNTRACK_CONFIRM,
	},
};
{% endcodeblock %}

它在4个 hook 点挂载了3个函数, 从而完成了整个 conntrack 工作. 关于如何写 netfilter 模块,参考[Writing Netﬁlter modules](http://inai.de/documents/Netfilter_Modules.pdf). 
