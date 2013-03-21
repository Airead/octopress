---
layout: post
title: "extract SNAT from kernel and compile"
date: 2013-03-19 09:42
comments: true
published: false
categories: iptables kernel
---

In this post, we will extract the code of SNAT from kernel and compile it. There are two files we need to create. ipt_MYSNAT.c will be compiled to ipt_MYSNAT.ko and libipt_MYSNAT.c will be compile to libipt_MYSNAT.so, then put them at some certain directories. if all done, we can just call iptables to invoke our new MYSNAT target.
<!--more-->

## extract SNAT target from kernel 3.5.7
ipt_snat_target is found in net/ipv4/netfilter/nf_nat_rule.c. All about SNAT in it are as follows:
{% codeblock lang:c %}
static unsigned int
ipt_snat_target(struct sk_buff *skb, const struct xt_action_param *par)
{
	struct nf_conn *ct;
	enum ip_conntrack_info ctinfo;
	const struct nf_nat_ipv4_multi_range_compat *mr = par->targinfo;

	NF_CT_ASSERT(par->hooknum == NF_INET_POST_ROUTING ||
		     par->hooknum == NF_INET_LOCAL_IN);

	ct = nf_ct_get(skb, &ctinfo);

	/* Connection must be valid and new. */
	NF_CT_ASSERT(ct && (ctinfo == IP_CT_NEW || ctinfo == IP_CT_RELATED ||
			    ctinfo == IP_CT_RELATED_REPLY));
	NF_CT_ASSERT(par->out != NULL);

	return nf_nat_setup_info(ct, &mr->range[0], NF_NAT_MANIP_SRC);
}


static int ipt_snat_checkentry(const struct xt_tgchk_param *par)
{
	const struct nf_nat_ipv4_multi_range_compat *mr = par->targinfo;

	/* Must be a valid range */
	if (mr->rangesize != 1) {
		pr_info("SNAT: multiple ranges no longer supported\n");
		return -EINVAL;
	}
	return 0;
}


static struct xt_target ipt_snat_reg __read_mostly = {
	.name		= "SNAT",
	.target		= ipt_snat_target,
	.targetsize	= sizeof(struct nf_nat_ipv4_multi_range_compat),
	.table		= "nat",
	.hooks		= (1 << NF_INET_POST_ROUTING) | (1 << NF_INET_LOCAL_IN),
	.checkentry	= ipt_snat_checkentry,
	.family		= AF_INET,
};


int __init nf_nat_rule_init(void)
{
	int ret;

	ret = register_pernet_subsys(&nf_nat_rule_net_ops);
	if (ret != 0)
		goto out;
	ret = xt_register_target(&ipt_snat_reg);
	if (ret != 0)
		goto unregister_table;

	ret = xt_register_target(&ipt_dnat_reg);
	if (ret != 0)
		goto unregister_snat;

	return ret;

 unregister_snat:
	xt_unregister_target(&ipt_snat_reg);
 unregister_table:
	unregister_pernet_subsys(&nf_nat_rule_net_ops);
 out:
	return ret;
}


void nf_nat_rule_cleanup(void)
{
	xt_unregister_target(&ipt_dnat_reg);
	xt_unregister_target(&ipt_snat_reg);
	unregister_pernet_subsys(&nf_nat_rule_net_ops);
}

{% endcodeblock %}

### __init function
simplify MYSNAT init function.
{% codeblock lang:c %}
static int __init mysnat_target_init(void)  
{  
        return xt_register_target(&ipt_mysnat_reg);  
}  
  
static void __exit mysnat_target_exit(void)  
{  
        xt_unregister_target(&ipt_mysnat_reg);  
}  

module_init(mysnat_target_init);  
module_exit(mysnat_target_exit);  
{% endcode %}

### ipt_mysnat_reg
extract ipt_mysnat_reg from kernel
{% codeblock lang:c %}
static struct xt_target ipt_mysnat_reg __read_mostly = {
	.name		= "MYSNAT",       /* name is the name of the target that you define */
	.target		= ipt_mysnat_target,
	.targetsize	= sizeof(struct nf_nat_ipv4_multi_range_compat),
	.table		= "nat",
    /*
     * hooks is a bitmask and may contain zero or more of the following flags:
     *   • 1 << NF_INET_PRE_ROUTING
     *   • 1 << NF_INET_LOCAL_IN
     *   • 1 << NF_INET_FORWARD
     *   • 1 << NF_INET_LOCAL_OUT
     *   • 1 << NF_INET_POST_ROUTING
     */
	.hooks		= (1 << NF_INET_POST_ROUTING) | (1 << NF_INET_LOCAL_IN),
	/* Called when user tries to insert an entry of this type:
           hook_mask is a bitmask of hooks from which it can be
           called. */
	/* Should return 0 on success or an error code otherwise (-Exxxx). */
	.checkentry	= ipt_mysnat_checkentry,
	.family		= AF_INET,
};
{% endcodeblock %}

### Checkentry And Target
{% codeblock lang:c %}
/* Source NAT */
static unsigned int
ipt_mysnat_target(struct sk_buff *skb, const struct xt_action_param *par)
{
	struct nf_conn *ct;
	enum ip_conntrack_info ctinfo;
	const struct nf_nat_ipv4_multi_range_compat *mr = par->targinfo;

	NF_CT_ASSERT(par->hooknum == NF_INET_POST_ROUTING ||
		     par->hooknum == NF_INET_LOCAL_IN);

	ct = nf_ct_get(skb, &ctinfo);

	/* Connection must be valid and new. */
	NF_CT_ASSERT(ct && (ctinfo == IP_CT_NEW || ctinfo == IP_CT_RELATED ||
			    ctinfo == IP_CT_RELATED_REPLY));
	NF_CT_ASSERT(par->out != NULL);

	return nf_nat_setup_info(ct, &mr->range[0], NF_NAT_MANIP_SRC);
}

static int ipt_mysnat_checkentry(const struct xt_tgchk_param *par)
{
	const struct nf_nat_ipv4_multi_range_compat *mr = par->targinfo;

	/* Must be a valid range */
	if (mr->rangesize != 1) {
		pr_info("SNAT: multiple ranges no longer supported\n");
		return -EINVAL;
	}
	return 0;
}
{% endcodeblock %}

put them togethter, we can get a file([ipt_MYSNAT.c](/downloads/code/iptables/ipt_MYSNAT.c)) which can compile to lib_MYSNAT.ko.

## extract SNAT from iptables 1.4.12 code
{% codeblock lang:c %}
#include <stdio.h>
#include <netdb.h>
#include <string.h>
#include <stdlib.h>
#include <xtables.h>
#include <iptables.h>
#include <limits.h> /* INT_MAX in ip_tables.h */
#include <linux/netfilter_ipv4/ip_tables.h>
#include <net/netfilter/nf_nat.h>

enum {
	O_TO_SRC = 0,
	O_RANDOM,
	O_PERSISTENT,
	O_X_TO_SRC,
	F_TO_SRC   = 1 << O_TO_SRC,
	F_RANDOM   = 1 << O_RANDOM,
	F_X_TO_SRC = 1 << O_X_TO_SRC,
};

/* Source NAT data consists of a multi-range, indicating where to map
   to. */
struct ipt_natinfo
{
	struct xt_entry_target t;
	struct nf_nat_multi_range mr;
};

static void SNAT_help(void)
{
	printf(
"SNAT target options:\n"
" --to-source [<ipaddr>[-<ipaddr>]][:port[-port]]\n"
"				Address to map source to.\n"
"[--random] [--persistent]\n");
}

static const struct xt_option_entry SNAT_opts[] = {
	{.name = "to-source", .id = O_TO_SRC, .type = XTTYPE_STRING,
	 .flags = XTOPT_MAND | XTOPT_MULTI},
	{.name = "random", .id = O_RANDOM, .type = XTTYPE_NONE},
	{.name = "persistent", .id = O_PERSISTENT, .type = XTTYPE_NONE},
	XTOPT_TABLEEND,
};

static struct ipt_natinfo *
append_range(struct ipt_natinfo *info, const struct nf_nat_range *range)
{
	unsigned int size;

	/* One rangesize already in struct ipt_natinfo */
	size = XT_ALIGN(sizeof(*info) + info->mr.rangesize * sizeof(*range));

	info = realloc(info, size);
	if (!info)
		xtables_error(OTHER_PROBLEM, "Out of memory\n");

	info->t.u.target_size = size;
	info->mr.range[info->mr.rangesize] = *range;
	info->mr.rangesize++;

	return info;
}

/* Ranges expected in network order. */
static struct xt_entry_target *
parse_to(const char *orig_arg, int portok, struct ipt_natinfo *info)
{
	struct nf_nat_range range;
	char *arg, *colon, *dash, *error;
	const struct in_addr *ip;

	arg = strdup(orig_arg);
	if (arg == NULL)
		xtables_error(RESOURCE_PROBLEM, "strdup");
	memset(&range, 0, sizeof(range));
	colon = strchr(arg, ':');

	if (colon) {
		int port;

		if (!portok)
			xtables_error(PARAMETER_PROBLEM,
				   "Need TCP, UDP, SCTP or DCCP with port specification");

		range.flags |= IP_NAT_RANGE_PROTO_SPECIFIED;

		port = atoi(colon+1);
		if (port <= 0 || port > 65535)
			xtables_error(PARAMETER_PROBLEM,
				   "Port `%s' not valid\n", colon+1);

		error = strchr(colon+1, ':');
		if (error)
			xtables_error(PARAMETER_PROBLEM,
				   "Invalid port:port syntax - use dash\n");

		dash = strchr(colon, '-');
		if (!dash) {
			range.min.tcp.port
				= range.max.tcp.port
				= htons(port);
		} else {
			int maxport;

			maxport = atoi(dash + 1);
			if (maxport <= 0 || maxport > 65535)
				xtables_error(PARAMETER_PROBLEM,
					   "Port `%s' not valid\n", dash+1);
			if (maxport < port)
				/* People are stupid. */
				xtables_error(PARAMETER_PROBLEM,
					   "Port range `%s' funky\n", colon+1);
			range.min.tcp.port = htons(port);
			range.max.tcp.port = htons(maxport);
		}
		/* Starts with a colon? No IP info...*/
		if (colon == arg) {
			free(arg);
			return &(append_range(info, &range)->t);
		}
		*colon = '\0';
	}

	range.flags |= IP_NAT_RANGE_MAP_IPS;
	dash = strchr(arg, '-');
	if (colon && dash && dash > colon)
		dash = NULL;

	if (dash)
		*dash = '\0';

	ip = xtables_numeric_to_ipaddr(arg);
	if (!ip)
		xtables_error(PARAMETER_PROBLEM, "Bad IP address \"%s\"\n",
			   arg);
	range.min_ip = ip->s_addr;
	if (dash) {
		ip = xtables_numeric_to_ipaddr(dash+1);
		if (!ip)
			xtables_error(PARAMETER_PROBLEM, "Bad IP address \"%s\"\n",
				   dash+1);
		range.max_ip = ip->s_addr;
	} else
		range.max_ip = range.min_ip;

	free(arg);
	return &(append_range(info, &range)->t);
}

static void SNAT_parse(struct xt_option_call *cb)
{
	const struct ipt_entry *entry = cb->xt_entry;
	struct ipt_natinfo *info = (void *)(*cb->target);
	int portok;

	if (entry->ip.proto == IPPROTO_TCP
	    || entry->ip.proto == IPPROTO_UDP
	    || entry->ip.proto == IPPROTO_SCTP
	    || entry->ip.proto == IPPROTO_DCCP
	    || entry->ip.proto == IPPROTO_ICMP)
		portok = 1;
	else
		portok = 0;

	xtables_option_parse(cb);
	switch (cb->entry->id) {
	case O_TO_SRC:
		if (cb->xflags & F_X_TO_SRC) {
			if (!kernel_version)
				get_kernel_version();
			if (kernel_version > LINUX_VERSION(2, 6, 10))
				xtables_error(PARAMETER_PROBLEM,
					   "SNAT: Multiple --to-source not supported");
		}
		*cb->target = parse_to(cb->arg, portok, info);
		/* WTF do we need this for?? */
		if (cb->xflags & F_RANDOM)
			info->mr.range[0].flags |= IP_NAT_RANGE_PROTO_RANDOM;
		cb->xflags |= F_X_TO_SRC;
		break;
	case O_RANDOM:
		if (cb->xflags & F_TO_SRC)
			info->mr.range[0].flags |= IP_NAT_RANGE_PROTO_RANDOM;
		break;
	case O_PERSISTENT:
		info->mr.range[0].flags |= IP_NAT_RANGE_PERSISTENT;
		break;
	}
}

static void print_range(const struct nf_nat_range *r)
{
	if (r->flags & IP_NAT_RANGE_MAP_IPS) {
		struct in_addr a;

		a.s_addr = r->min_ip;
		printf("%s", xtables_ipaddr_to_numeric(&a));
		if (r->max_ip != r->min_ip) {
			a.s_addr = r->max_ip;
			printf("-%s", xtables_ipaddr_to_numeric(&a));
		}
	}
	if (r->flags & IP_NAT_RANGE_PROTO_SPECIFIED) {
		printf(":");
		printf("%hu", ntohs(r->min.tcp.port));
		if (r->max.tcp.port != r->min.tcp.port)
			printf("-%hu", ntohs(r->max.tcp.port));
	}
}

static void SNAT_print(const void *ip, const struct xt_entry_target *target,
                       int numeric)
{
	const struct ipt_natinfo *info = (const void *)target;
	unsigned int i = 0;

	printf(" to:");
	for (i = 0; i < info->mr.rangesize; i++) {
		print_range(&info->mr.range[i]);
		if (info->mr.range[i].flags & IP_NAT_RANGE_PROTO_RANDOM)
			printf(" random");
		if (info->mr.range[i].flags & IP_NAT_RANGE_PERSISTENT)
			printf(" persistent");
	}
}

static void SNAT_save(const void *ip, const struct xt_entry_target *target)
{
	const struct ipt_natinfo *info = (const void *)target;
	unsigned int i = 0;

	for (i = 0; i < info->mr.rangesize; i++) {
		printf(" --to-source ");
		print_range(&info->mr.range[i]);
		if (info->mr.range[i].flags & IP_NAT_RANGE_PROTO_RANDOM)
			printf(" --random");
		if (info->mr.range[i].flags & IP_NAT_RANGE_PERSISTENT)
			printf(" --persistent");
	}
}

static struct xtables_target snat_tg_reg = {
	.name		= "SNAT",
	.version	= XTABLES_VERSION,
	.family		= NFPROTO_IPV4,
	.size		= XT_ALIGN(sizeof(struct nf_nat_multi_range)),
	.userspacesize	= XT_ALIGN(sizeof(struct nf_nat_multi_range)),
	.help		= SNAT_help,
	.x6_parse	= SNAT_parse,
	.print		= SNAT_print,
	.save		= SNAT_save,
	.x6_options	= SNAT_opts,
};

void _init(void)
{
	xtables_register_target(&snat_tg_reg);
}
{% endcodeblock %}

## make
{% codeblock lang:make %}
CC=gcc  

IPTABLES_SRC=/home/airead/downloads/tar/iptables-v1.4.12
INCLUDE=-I$(IPTABLES_SRC)/include  
KERNEL_SRC=/lib/modules/`uname -r`/build  
MOD=ipt_MYSNAT.ko  

obj-m = ipt_MYSNAT.o

all: modules libipt_MYSNAT.so  

modules: $(MOD)  

ipt_MYSNAT.ko:  ipt_MYSNAT.c  
	$(MAKE) -C $(KERNEL_SRC) SUBDIRS=$(PWD) modules  

libipt_MYSNAT.so: libipt_MYSNAT.c  
	$(CC)  $(INCLUDE) -fPIC -c libipt_MYSNAT.c  
	ld -shared -o libipt_MYSNAT.so libipt_MYSNAT.o  

clean:  
	-rm -f *.o *.so *.ko .*.cmd *.mod.c *.symvers *.order  

install: all  
	cp -rf libipt_MYSNAT.so /lib/xtables/  
	cp -rf $(MOD) /lib/modules/`uname -r`/kernel/net/ipv4/netfilter/  
{% endcodeblock %}

