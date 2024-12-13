---
title: PostgreSQL btree_gist 插件源码分析
author: amwps290
date: 2024-12-04 00:10:00 +0800
categories: [数据库,源码分析,PostgreSQL]
tags: [database,postgresql]
mermaid: true
---

## GIST 简介

GIST(Generalized Search Tree),是通用搜索树的缩写，是一种平衡的树状结构的访问方法，是实现很多索引方案的基础模板，包括 B 树，R 树，RD 树等等。

GIST 的一个优点是它允许数据类型领域的专家(而不是数据库专家)开发自定义数据类型的访问方法，可以理解为，如果你对某个数据类型有独特的理解，可以在 PostgreSQL 中实现自己的索引访问方法。

GIST 索引相对于 B 树索引来说，除了可以比较大小和相等关系，还可以比较其他关系，例如距离的远近，词汇的相似程度等等。

## GIST 索引的实现

GIST 索引的实现必须要提供 5 个方法，包括 `same`,`consistent`,`union`,`penalty`,`picksplit` 方法。

### consistent

对于给定的一个索引项 `p` 和查询值 `q`,该函数确定该索引项是否与查询项一致。也就是说，对于该索引项所代表的任何行，谓词`indexed_column indexable_operator q` 是否为真，对于叶子索引项来说，这等同于测试可索引的条件，对与内部节点来说，这决定了是否有必要扫描树节点所代表的索引子树，当结果为真时，还必须返回一个 `recheck` 标志，这表明谓词肯定为真还是可能为真。如果 `recheck=false`, 这表明索引已经完全测试了谓词条件，而如果 `recheck=true`,则表明该行是候选匹配，这种情况下，系统会根据实际行值自动评估`index_operator` 是否真的匹配。

该函数的 SQL 声明必须如下所示：

```sql
CREATE OR REPLACE FUNCTION my_consistent(internal, data_type, smallint, oid, internal)
RETURNS bool
AS 'MODULE_PATHNAME'
LANGUAGE C STRICT;
```

调用的 C 函数会是如下的逻辑结构：

```c
PG_FUNCTION_INFO_V1(my_consistent);

Datum
my_consistent(PG_FUNCTION_ARGS)
{
    GISTENTRY  *entry = (GISTENTRY *) PG_GETARG_POINTER(0);
    data_type  *query = PG_GETARG_DATA_TYPE_P(1);
    StrategyNumber strategy = (StrategyNumber) PG_GETARG_UINT16(2);
    /* Oid subtype = PG_GETARG_OID(3); */
    bool       *recheck = (bool *) PG_GETARG_POINTER(4);
    data_type  *key = DatumGetDataType(entry->key);
    bool        retval;

    /*
     * determine return value as a function of strategy, key and query.
     *
     * Use GIST_LEAF(entry) to know where you're called in the index tree,
     * which comes handy when supporting the = operator for example (you could
     * check for non empty union() in non-leaf nodes and equality in leaf
     * nodes).
     */

    *recheck = true;        /* or false if check is exact */

    PG_RETURN_BOOL(retval);
}
```

这里的 `key` 是索引中的一个元素， `query` 是索引中正在查找的值。参数 `StrategyNumber` 表示应用运算符类中的哪个运算符，它与命令 `CREATE OPERATOR CLASS` 中的运算符编号之一相匹配。


### union 

这个方法可以整合树中的信息，如果给定了一组索引条目，该函数会生成一个新的索引条目，代表所有给定的条目

该函数的 SQL 声明必须如下所示：

```sql
CREATE OR REPLACE FUNCTION my_union(internal, internal)
RETURNS storage_type
AS 'MODULE_PATHNAME'
LANGUAGE C STRICT;
```

其中的 C 语言代码逻辑结构如下：

```c
PG_FUNCTION_INFO_V1(my_union);

Datum
my_union(PG_FUNCTION_ARGS)
{
    GistEntryVector *entryvec = (GistEntryVector *) PG_GETARG_POINTER(0);
    GISTENTRY  *ent = entryvec->vector;
    data_type  *out,
               *tmp,
               *old;
    int         numranges,
                i = 0;

    numranges = entryvec->n;
    tmp = DatumGetDataType(ent[0].key);
    out = tmp;

    if (numranges == 1)
    {
        out = data_type_deep_copy(tmp);

        PG_RETURN_DATA_TYPE_P(out);
    }

    for (i = 1; i < numranges; i++)
    {
        old = out;
        tmp = DatumGetDataType(ent[i].key);
        out = my_union_implementation(out, tmp);
    }

    PG_RETURN_DATA_TYPE_P(out);
}
```

函数 `union` 的结果必须是索引存储类型的值，不管是什么类型（可能与索引列的类型相同，或者不同），union 函数应该返回一个指向 palloc 的新分配的内存，即使类型没有改变，也不能按原样返回输入值。

如上所示，`union` 函数的第一个 `internal` 参数实际上是一个 `GistEntryVector` 指针，第二个参数是一个指向整数变量的指针，现在可以忽略。

### penalty

返回一个值，表示将新条目插入树中某个分支的“成本”，项目将沿着树中 penalty 最少的路径插入，penalty 的返回值应该是非负值，如果返回的是负值，它将被视为 0 。

SQL 函数的声明如下：

```sql
CREATE OR REPLACE FUNCTION my_penalty(internal, internal, internal)
RETURNS internal
AS 'MODULE_PATHNAME'
LANGUAGE C STRICT;  -- in some cases penalty functions need not be strict
```

C 语言的代码逻辑结构如下：

```c
PG_FUNCTION_INFO_V1(my_penalty);

Datum
my_penalty(PG_FUNCTION_ARGS)
{
    GISTENTRY  *origentry = (GISTENTRY *) PG_GETARG_POINTER(0);
    GISTENTRY  *newentry = (GISTENTRY *) PG_GETARG_POINTER(1);
    float      *penalty = (float *) PG_GETARG_POINTER(2);
    data_type  *orig = DatumGetDataType(origentry->key);
    data_type  *new = DatumGetDataType(newentry->key);

    *penalty = my_penalty_implementation(orig, new);
    PG_RETURN_POINTER(penalty);
}
```

由于历史原因，penalty 函数并不返回一个 float 结果，而是必须将值存储在第三个参数所指示的位置。

`penalty`  函数对索引性能至关重要,它在插入时用于确定新条目应该遵循树中的哪个分支


### picksplit

当需要拆分一个索引页时，该函数能将决定哪些条目留在旧页，哪些移动到新叶。

SQL 声明如下：

```sql
CREATE OR REPLACE FUNCTION my_picksplit(internal, internal)
RETURNS internal
AS 'MODULE_PATHNAME'
LANGUAGE C STRICT;
```

然后 C 语言的代码逻辑如下：

```c
PG_FUNCTION_INFO_V1(my_picksplit);

Datum
my_picksplit(PG_FUNCTION_ARGS)
{
    GistEntryVector *entryvec = (GistEntryVector *) PG_GETARG_POINTER(0);
    GIST_SPLITVEC *v = (GIST_SPLITVEC *) PG_GETARG_POINTER(1);
    OffsetNumber maxoff = entryvec->n - 1;
    GISTENTRY  *ent = entryvec->vector;
    int         i,
                nbytes;
    OffsetNumber *left,
               *right;
    data_type  *tmp_union;
    data_type  *unionL;
    data_type  *unionR;
    GISTENTRY **raw_entryvec;

    maxoff = entryvec->n - 1;
    nbytes = (maxoff + 1) * sizeof(OffsetNumber);

    v->spl_left = (OffsetNumber *) palloc(nbytes);
    left = v->spl_left;
    v->spl_nleft = 0;

    v->spl_right = (OffsetNumber *) palloc(nbytes);
    right = v->spl_right;
    v->spl_nright = 0;

    unionL = NULL;
    unionR = NULL;

    /* Initialize the raw entry vector. */
    raw_entryvec = (GISTENTRY **) malloc(entryvec->n * sizeof(void *));
    for (i = FirstOffsetNumber; i <= maxoff; i = OffsetNumberNext(i))
        raw_entryvec[i] = &(entryvec->vector[i]);

    for (i = FirstOffsetNumber; i <= maxoff; i = OffsetNumberNext(i))
    {
        int         real_index = raw_entryvec[i] - entryvec->vector;

        tmp_union = DatumGetDataType(entryvec->vector[real_index].key);
        Assert(tmp_union != NULL);

        /*
         * Choose where to put the index entries and update unionL and unionR
         * accordingly. Append the entries to either v->spl_left or
         * v->spl_right, and care about the counters.
         */

        if (my_choice_is_left(unionL, curl, unionR, curr))
        {
            if (unionL == NULL)
                unionL = tmp_union;
            else
                unionL = my_union_implementation(unionL, tmp_union);

            *left = real_index;
            ++left;
            ++(v->spl_nleft);
        }
        else
        {
            /*
             * Same on the right
             */
        }
    }

    v->spl_ldatum = DataTypeGetDatum(unionL);
    v->spl_rdatum = DataTypeGetDatum(unionR);
    PG_RETURN_POINTER(v);
}
```

这里需要注意，`picksplit` 函数的结果是通过修改传入的 v 结构来传递的。

与 `penalty` 一样，`picksplit` 函数对于索引的良好性能至关重要。设计合适的 `penalty` 和 `picksplit` 实现是实现性能良好的 GIST 索引的挑战所在。

### same

如果两个索引项相同，则返回 true, 否则返回 false.

SQL 声明如下：
```sql
CREATE OR REPLACE FUNCTION my_same(storage_type, storage_type, internal)
RETURNS internal
AS 'MODULE_PATHNAME'
LANGUAGE C STRICT;
```

C 语言的代码如下：

```c
PG_FUNCTION_INFO_V1(my_same);

Datum
my_same(PG_FUNCTION_ARGS)
{
    prefix_range *v1 = PG_GETARG_PREFIX_RANGE_P(0);
    prefix_range *v2 = PG_GETARG_PREFIX_RANGE_P(1);
    bool       *result = (bool *) PG_GETARG_POINTER(2);

    *result = my_eq(v1, v2);
    PG_RETURN_POINTER(result);
}
```

到这里我们所有必须的函数就已经介绍完成了，接下来我们就看一下 int 类型实现 GIST 索引的相关代码。

参考连接：
[btree-gist](https://www.postgresql.org/docs/14/btree-gist.html)
[GIST索引官方文档](https://www.postgresql.org/docs/14/gist-extensibility.html)

