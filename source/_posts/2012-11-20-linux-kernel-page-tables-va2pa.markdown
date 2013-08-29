---
layout: post
title: "linux kernel page tables va2pa"
date: 2012-11-20 14:38
comments: true
published: false
categories: [kerenl, memory, ]
---

kernel: 2.6.32.59
<!--more-->

{% codeblock lang:c %}
static int vir2phy(unsigned long va) 
{     
    struct task_struct *pcb_tmp;     
    pcb_tmp = current;     
    pgd_tmp = pgd_offset(pcb_tmp->mm,va);     
    pud_tmp = pud_offset(pgd_tmp,va);     
    pmd_tmp = pmd_offset(pud_tmp,va);     
    pte_tmp = pte_offset_kernel(pmd_tmp,va);      
    pa = (pte_val(*pte_tmp) & PAGE_MASK) |(va & ~PAGE_MASK);     
    return pa; 
}
{% endcodeblock %}

{% codeblock lang:c%}
#ifdef CONFIG_PGTABLE_4
#define PGDIR_SHIFT		(PUD_SHIFT + (PTRS_PER_PTD_SHIFT))
#else
#define PGDIR_SHIFT		(PMD_SHIFT + (PTRS_PER_PTD_SHIFT))
#endif

/*
 * How many pointers will a page table level hold expressed in shift
 */
#define PTRS_PER_PTD_SHIFT	(PAGE_SHIFT-3)

#define PTRS_PER_PGD_SHIFT	PTRS_PER_PTD_SHIFT
#define PTRS_PER_PGD		(1UL << PTRS_PER_PGD_SHIFT)

/* PAGE_SHIFT determines the page size */

#define PAGE_SHIFT	(12)
#define PAGE_SIZE	(1UL << PAGE_SHIFT)
#define PAGE_MASK	(~(PAGE_SIZE-1))

static inline unsigned long
pgd_index (unsigned long address)
{
	unsigned long region = address >> 61;
	unsigned long l1index = (address >> PGDIR_SHIFT) & ((PTRS_PER_PGD >> 3) - 1);

	return (region << (PAGE_SHIFT - 6)) | l1index;
}

/* The offset in the 1-level directory is given by the 3 region bits
   (61..63) and the level-1 bits.  */
static inline pgd_t*
pgd_offset (const struct mm_struct *mm, unsigned long address)
{
	return mm->pgd + pgd_index(address);
}
{% endcodeblock %}

{% codeblock lang:c%}

{% endcodeblock %}
{% codeblock lang:c%}

{% endcodeblock %}
{% codeblock lang:c%}

{% endcodeblock %}
{% codeblock lang:c%}

{% endcodeblock %}
{% codeblock lang:c%}

{% endcodeblock %}
## References
 - [虚拟地址转化为物理地址](http://hi.baidu.com/aokikyon/item/0bc3353aa4ba73fdde222131)
 - Professional Linux Kernel Architecture
