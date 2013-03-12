---
layout: post
title: "linux snat implement"
date: 2013-03-04 10:16
comments: true
published: false
categories: kernel snat
---

如果想要在 linux 主机上配上 SNAT 命令，一般可以这么做：  
`$ iptalbes -t nat -A POSTROUTING -o eth0  -s 172.16.22.0/24 -j SNAT --to-source 192.0.2.1`  
那么它又是如何被内核接收，并执行的呢？

在 kernel 源码树内 net/ipv4/netfilter/Makefile 内，有以下内容：  
`iptable_nat-y	:= nf_nat_rule.o nf_nat_standalone.o`  
这说明 nf_nat_rule.c 与 nf_nat_standalone.c 与 iptables 之间是可能是有联系的。经分析可发现初始化 iptable_nat.ko 时会调用 `nf_nat_standalone_init()` --> `nf_nat_rule_init()` --> `xt_register_target(&ipt_snat_reg)`, 至此终于找到了iptalbe在内核中的注册点。ip_sant_reg 内容如下：
{% codeblock lang:c %}
static struct xt_target ipt_snat_reg __read_mostly = {
	.name		= "SNAT",
	.target		= ipt_snat_target,
	.targetsize	= sizeof(struct nf_nat_multi_range_compat),
	.table		= "nat",
	.hooks		= (1 << NF_INET_POST_ROUTING) | (1 << NF_INET_LOCAL_IN),
	.checkentry	= ipt_snat_checkentry,
	.family		= AF_INET,
};
{% endcodeblock %}

由经验可知，`ipt_snat_target()`为确定进行 snat 的回调函数。在验证一些条件后，它调用了 `nf_nat_setup_info()` 从名称上看，它就是为了设置nat的各种信息。接下来就以它为切入点，好好分析一下 SNAT 的设置、查找与修改。光设置完信息还不够，当数据包到来的时候还要知道如何使用这些信息。在 nf_nat_core.c 中有函数 `nf_nat_packet()`，它的注释为"Do packet manipulations according to nf_nat_setup_info.",它明显就是利用 nf_nat_setup_info() 设置的信息去进行包的修改了。

## nat hooks
{% codeblock lang:c %}
/* We must be after connection tracking and before packet filtering. */
static struct nf_hook_ops nf_nat_ops[] __read_mostly = {
	/* Before packet filtering, change destination */
	{
		.hook		= nf_nat_in,
		.owner		= THIS_MODULE,
		.pf		= NFPROTO_IPV4,
		.hooknum	= NF_INET_PRE_ROUTING,
		.priority	= NF_IP_PRI_NAT_DST,
	},
	/* After packet filtering, change source */
	{
		.hook		= nf_nat_out,
		.owner		= THIS_MODULE,
		.pf		= NFPROTO_IPV4,
		.hooknum	= NF_INET_POST_ROUTING,
		.priority	= NF_IP_PRI_NAT_SRC,
	},
	/* Before packet filtering, change destination */
	{
		.hook		= nf_nat_local_fn,
		.owner		= THIS_MODULE,
		.pf		= NFPROTO_IPV4,
		.hooknum	= NF_INET_LOCAL_OUT,
		.priority	= NF_IP_PRI_NAT_DST,
	},
	/* After packet filtering, change source */
	{
		.hook		= nf_nat_fn,
		.owner		= THIS_MODULE,
		.pf		= NFPROTO_IPV4,
		.hooknum	= NF_INET_LOCAL_IN,
		.priority	= NF_IP_PRI_NAT_SRC,
	},
};
{% endcodeblock %}
共注册了4个函数：nf_nat_in(), nf_nat_out(), nf_nat_local_fn(), nf_nat_fn()，而这四个函数都毫无例外的调用了 nf_nat_fn(). HOOK2MANIP 跟据 hooknum 判断 nat 操作是 snat 还是 dnat.

nf_nat_fn() 先进行一系列的条件判断，然后进入 switch 语句。
{% codeblock lang:c %}
switch (ctinfo) {
case IP_CT_RELATED:
case IP_CT_RELATED_REPLY:
	if (ip_hdr(skb)->protocol == IPPROTO_ICMP) {
		if (!nf_nat_icmp_reply_translation(ct, ctinfo,
						   hooknum, skb))
			return NF_DROP;
		else
			return NF_ACCEPT;
	}
	/* Fall thru... (Only ICMPs can be IP_CT_IS_REPLY) */
case IP_CT_NEW:

    /* Seen it before?  This can happen for loopback, retrans,
	   or local packets.. */
	if (!nf_nat_initialized(ct, maniptype)) {
		unsigned int ret;

		ret = nf_nat_rule_find(skb, hooknum, in, out, ct);
		if (ret != NF_ACCEPT)
			return ret;
	} else
		pr_debug("Already setup manip %s for ct %p\n",
			 maniptype == IP_NAT_MANIP_SRC ? "SRC" : "DST",
			 ct);
	break;

default:
	/* ESTABLISHED */
	NF_CT_ASSERT(ctinfo == IP_CT_ESTABLISHED ||
		     ctinfo == IP_CT_ESTABLISHED_REPLY);
}
{% endcodeblock %}
switch 根据 ctinfo 分四个不同流程：  
  1. IP_CT_RELATED/IP_CT_RELATED_REPLY：判断数据包是否为 icmp 协议，如果是的话就拦截，转换后发送；否则，直接流向 IP_CT_NEW。  
  2. IP_CT_NEW：判断数据包是否已经初始化，也就是说 nf_nat_setup_info() 中是否在 ct->status 设置了 IPS_SRC_NAT_DONE/IPS_DST_NAT_DONE. 如果没有初始化过，就调用 `nf_nat_rule_find()`-->`ipt_do_table()`，遍历所有防火墙规则，返回是否接受。其中 ipt_do_table() 会调用 ipt_snat_reg 中的 ipt_snat_target()函数，进行 nf_nat_setup_info() 的设置工作。就是通过 `verdict = t->u.kernel.target->target(skb, &acpar);`, t 为`const struct xt_entry_target *t;`  
  3. ESTABLISHED：验证是数据包是否处于 ESTABLISHED／ESTABLISHED_REPLY 状态。  
最后调用 nf_nat_packet(ct, ctinfo, hooknum, skb) 利用 nf_nat_setup_info() 设置的信息进行包的修改。

nf_nat_packet(ct, ctinfo, hooknum, skb):  
  1. 根据 hooknum 判断进行 IPS_SRC_NAT 还是 IPS_DST_NAT  
  2. 再根据方向修订进行 IPS_SRC_NAT 还是 IPS_DST_NAT  
  3. 如果 ct->status & statusbit 为假，返回 NF_ACCEPT；否则，调用 manip_pkt 修改源或目的地址。  
manip_pkt() 会调用协议相关的函数，如 `.manip_pkt = tcp_manip_pkt`，然后计算 IP 头的 checksum. 最后返回到 nf_nat_fn() 结束。
  
iptable_nat-y	:= nf_nat_rule.o nf_nat_standalone.o：  
负责用户态的 iptables 与内核态交互。 其中 nf_nat_standalone.c 中注册 netfilter hook 函数。  
  1. 在 NF_INET_PRE_ROUTING 上以 NF_IP_PRI_NAT_DST 优先级注册 nf_nat_in()，猜测是用来处理 reply 方向上 SNAT 包，修改 dstip.  
  2. 在 NF_INET_POST_ROUTING 上以 NF_IP_PRI_NAT_SRC 优先级注册 nf_nat_out()，猜测是用对 origin 方向进行SNAT，修改 srcip.  
nf_nat_in() 与 nf_nat_out() 都调用 nf_nat_fn()，它通过 nf_nat_find_rule -> ipt_do_table 来调用 iptables 中的 target 函数，target 函数被用来调用 nf_nat_setup_info() 设置 NAT 信息。然后再通过 nf_nat_packet() 对比 nf_nat_setup_info() 设置的内容对数据包进行更改。在更改的过程中调用了协议相关的 manip_pkt 函数。

nf_nat-y		:= nf_nat_core.o nf_nat_helper.o nf_nat_proto_unknown.o nf_nat_proto_common.o nf_nat_proto_tcp.o nf_nat_proto_udp.o nf_nat_proto_icmp.o  
包含了具体修改数据包的代码：如 nf_nat_setup_info() 与 nf_nat_packet() 等。通过 iptalbe_nat 与用户态数据进行交互。

下面我们分析一下 nf_nat_setup_info(struct nf_conn *ct, const struct nf_nat_ipv4_range *range, enum nf_nat_manip_type maniptype)， 它有3个参数：
  1. struct nf_conn *ct: 获取于 ipt_snat_target() 中的 `ct = nf_ct_get(skb, &ctinfo);` 也就是说，skb 走到这里后, ct 是一定存在的；  
  2. const struct nf_nat_ipv4_multi_range_compat *mr = par->targinfo: 来自于用户态 iptables 配置的数据, 内核中的结构是 struct nf_nat_ipv4_range;
  3. enum nf_nat_manip_type：NF_NAT_MANIP_SRC，说明是设置 SNAT 还是 DNAT。  
如果直接构造一个 struct nf_nat_ipv4_multi_range_compat 结构，就可以不用管 ipt_snat_target, ipt_do_table, nf_nat_rule_find, 直接根据 ctinfo 及 skb 调用 nf_nat_setup_info() 就可以了。

## nf_nat_setup_info()
  1. 找到或分配一个 struct nf_conn_nat nat;
  2. new_tuple = get_unique_tuple()
  3. 如果 new_tuple != curr_tuple, 则根据 new_tuple 选择一个 reply_tuple, 再把它设给 ct->tuplehash[*REPLY*].tuple
  4. 根据 ct->tuplehash[IP_CT_DIR_GOIGINAL].tuple 取得 srchash
  5. 将第1步的 nat 添加到 net->ipv4.nat_bysource[srchash] 中
  
### get_unique_tuple
  1. 根据 ct 获取 net
  2. 

bool nf_ct_invert_tuplepr(struct nf_conntrack_tuple *inverse, const struct nf_conntrack_tuple *orig):  
nf_ct_invert_tuple(inverse, orig, l3proto, l4proto):  
  1. 复制三层协议号
  2. 调用三层转换函数
  3. 反转方向
  4. 复制四层协议号
  5. 调用四层转换函数

## nat flow
 
 |          | PREROUTING      | PREROUTING                      | POSTROUTING                    | POSTROUTING |
 |----------+-----------------+---------------------------------+--------------------------------+-------------|
 | original | nf_conntrack_in | nf_nat_fn                       | nf_nat_fn                      | connfirm    |
 |----------+-----------------+---------------------------------+--------------------------------+-------------|
 |          |                 | cinfo = IPT_NEW                 | cinfo = IPT_NEW                |             |
 |          |                 | dir = orig                      | dir = orig                     |             |
 |          |                 | mtype = MANIP_DST               | mtype = MANIP_SRC              |             |
 |          |                 | statusbit = IPT_DST_NAT         | statusbit = IPT_SRC_NAT        |             |
 |          |                 | ct->status = IPT_SRC_NAT        | ct->status = IPT_SRC_NAT       |             |
 |          |                 | statusbit & ct->status not true | statusbit & ct->status is true |             |
 |          |                 | not invoke nf_nat_packet()      | invoke nf_nat_packet() to SNAT |             |
 |----------+-----------------+---------------------------------+--------------------------------+-------------|
 | reply    | nf_conntrack_in | nf_nat_fn                       | nf_nat_fn                      | connfirm    |
 |----------+-----------------+---------------------------------+--------------------------------+-------------|
 |          |                 | cinfo = IPT_ESTABLISHED_REPLY   | cinfo = IPT_ESTABLISHED_REPLY  |             |
 |          |                 | dir = reply                     | dir = reply                    |             |
 |          |                 | mtype = MANIP_DST               | mtype = MANIP_SRC              |             |
 |          |                 | statusbit = IPT_DST_NAT         | statusbit = IPT_SRC_NAT        |             |
 |          |                 | beause dir is reply             | beause dir is reply            |             |
 |          |                 | statusbit = IPT_SRC_NAT         | statusbit = IPT_DST_NAT        |             |
 |          |                 | ct->status = IPT_SRC_NAT        | ct->status = IPT_SRC_NAT       |             |
 |          |                 | invoke nf_nat_packet() to DNAT  | not invoke nf_nat_packet()     |             |
## SNAT Detail

### nf_conntrack_in(): 
ct->tuplehash[IP_CT_DIR_ORIGNAL].tuple =  [A10, B80] (0x01)  
ct->tuplehash[IP_CT_DIR_REPLY].tuple = [B80, A10] (0x02)  

### nf_nat_setup_info():
nf_ct_invert_tuplepr(&curr_tuple, &ct->tuplehash[IP_CT_DIR_REPLY].tuple);  
curr_tuple: [A10, B80] [0x03]  dir:original
get_unique_tuple(&new_tuple, &curr_tuple, range, ct, maniptype);  
new_tuple: [NAT20, B80] [0x04]
reply: [B80, NAT20]
nf_conntrack_alter_reply(ct, &reply);
ct->tuplehash[IP_CT_DIR_REPLY].tuple = reply, [B80, NAT20] (0X02)

### nf_nat_packet():
#### dir = IP_CT_DIR_ORIGINAL
nf_ct_invert_tuplepr(&target, &ct->tuplehash[!dir].tuple); NOW ct->..REPLY].tuple: [B80, NAT20]
target: [NAT20, B80]
iphdr [A10, B80], after manip_pkt at POSTROUTING 
iphdr: [NAT20, B80]
#### dir = IP_CT_DIR_REPLY
nf_ct_invert_tuplepr(&target, &ct->tuplehash[!dir].tuple); NOW ct->..ORIGINAL].tuple: [A10,B80]
target: [B80, A10]
iphdr [B80, NAT20], after manip_pkt at PREROUTING
iphdr: [B80, A10]

