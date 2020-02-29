[TOC]

## 一、问题描述

周中发现一个问题，metastore根据条件获取分区时发生异常，导致扫描所有分区，最终导致gc异常。

hive编译时会进行逻辑优化，在执行分区裁剪时，会根据相关的分区过滤条件去metastore查询要扫描的分区目录。metastore会根据hiveserver传过来的条件表达式进行解析，然后过滤不需要的分区。

**目前的问题是hiveserver传了一个 'date_p is not null'的子句，metastore这边无法解析（不支持），最终导致解析异常**。

另外经过测试发现如果hive QL中有between子句，并且join on中有分区字段，hiveserver查询分区时就会拼接 **'date_p is not null'**的条件给metastore，导致metastore解析异常。

sql语句如下,其中date_p 是test表的一个分区

```mysql
select
  COUNT(1)
from
  (
    select
      date_p
    from
      test
    where
      date_p BETWEEN 1
      and 2
  ) a
  inner join (
    select
      date_p
    from
      test
    where
      date_p BETWEEN 1
      and 2
  ) b on a.date_p = b.date_p;
```

metastore这边会收到分区过滤条件的语句：“date_p BETWEEN 1 AND 2 and date_p is not null”。

另外，将between换成大于、小于语句则可以正常运行。hiveserver 就不会自动拼接 "date_p is not null"给metastore。

## 二、解决方案

在metastore服务这边，`PartFilterExprUtil#makeExpressionTree(PartitionExpressionProxy proxy,byte[] expr) `会接收hiveserver传过来的分区过滤表达式，然后生成一个 ExpressionTree 后面用于去mysql中扫描分区。

代码如下

```java
public static ExpressionTree makeExpressionTree(PartitionExpressionProxy expressionProxy,
    byte[] expr) throws MetaException {
  String filter = null;
  try {
    //使用PartitionExpressionProxy解析hiveserver传过来的数据，并生成分区过滤表达式
    filter = expressionProxy.convertExprToFilter(expr);
  } catch (MetaException ex) {
    throw new IMetaStoreClient.IncompatibleMetastoreException(ex.getMessage());
  }
  //根据分区过滤表达式构建ExpressionTree。如果filter中有 date_p is not null，因为不支持，此处就会报错
  return PartFilterExprUtil.makeExpressionTree(filter);
}
```

现在问题在于hiveserver传过来的 expr 中可能会有IsNotNull类型的过滤条件，metastore不支持，因此最简单的做法就是搜索 expr 中的所有节点，然后将IsNotNull节点移除，之后再去计算分区过滤表达式就不会带上`date_p is Not Null`了。

计算分区过滤表达式主要是`PartitionExpressionProxy`的工作，这是一个接口，metastore用的是它的实现类PartitionExpressionForMetastore，因此我们修改这个类的convertExprToFilter方法即可。

`PartitionExpressionForMetastore#convertExprToFilter`方法的原代码如下

```java
@Override
public String convertExprToFilter(byte[] exprBytes) throws MetaException {
  return deserializeExpr(exprBytes).getExprString();
}
```

改成如下代码

```java
@Override
public String convertExprToFilter(byte[] exprBytes) throws MetaException {
  ExprNodeGenericFuncDesc exprNodeGenericFuncDesc = deserializeExpr(exprBytes);
  GenericUDF genericUDF = exprNodeGenericFuncDesc.getGenericUDF();
  //如果是not null类型的过滤，就不处理
  if(genericUDF.getClass() == GenericUDFOPNotNull.class){
    return "";
  }
  //如果是and或者or类型，就检查子句中是否有not null类型的子句，有的话去掉
  Iterator<ExprNodeDesc> iterator = exprNodeGenericFuncDesc.getChildren().iterator();
  while (iterator.hasNext()){
    ExprNodeDesc child = iterator.next();
    if(child.getClass() == ExprNodeGenericFuncDesc.class){
      GenericUDF childUdf = ((ExprNodeGenericFuncDesc) child).getGenericUDF();
      if(childUdf.getClass() == GenericUDFOPNotNull.class){
        iterator.remove();
      }
    }
  }
  return exprNodeGenericFuncDesc.getExprString();
}
```

改完重新编译hive/ql 模块代码，然后替换到metastore的lib下的包重启即可。

**由于`PartitionExpressionForMetastore#convertExprToFilter`中只有metastore的PartFilterExprUtil类会调用到，因此改造这个方法不会引起其他的问题。**