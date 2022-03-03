# 1.BooleanQuery概述

BooleanQuery用来表达多个Query组合查询,会将多个Query组成Cluase实现组合查询。Cluase有四种Query，分别是：

+ MUST:必须命中,参与评分,用"+"表示
+ MUST_NOT:不能命中,用"-"表示
+ FILTER:必须命中,不参与评分,用"#"表示
+ SHOULD:命中会影响评分

# 2.BooleanQuery的属性

```java
//should子句应该匹配的最少数量
private final int minimumNumberShouldMatch;
// BooleanQuery是由BooleanClause所构成的
private final List<BooleanClause> clauses;
// BooleanQuery有四种Clause(must、must_not、should、filter),clauseSets用于存储这四种所构成的Query
// must和should对应的value是MultiSet,must_not和filter对应的value是HashSet
private final Map<Occur, Collection<Query>> clauseSets; 
// BooleanQuery里的属性都是final的,所以可以缓存hashCode来避免频繁计算hashCode
private int hashCode;
```

# 3.BooleanQuery中的重要方法

## 3.1重写Query

```java
public Query rewrite(IndexReader reader) throws IOException {...}
```

重写BooleanQuery是一个递归的过程,目的是移除一些无用的clause,提升后续处理的性能,主要逻辑如下:

**1. 如果BooleanQuery中clause小于等于1,则不需要重写,直接返回 **

```java
// 如果BooleanQuery没有子句,直接返回MatchNoDocsQuery
	if (clauses.size() == 0) {
    return new MatchNoDocsQuery("empty BooleanQuery");
  }
//如果BooleanQuery中只有一个clause,则不需要重写,直接返回
  if (clauses.size() == 1) {
    BooleanClause c = clauses.get(0);
    Query query = c.getQuery();
    if (minimumNumberShouldMatch == 1 && c.getOccur() == Occur.SHOULD) {
      return query;
    } else if (minimumNumberShouldMatch == 0) {
      switch (c.getOccur()) {
        case SHOULD:
        case MUST:
          return query;
        case FILTER:
          // no scoring clauses, so return a score of 0
          return new BoostQuery(new ConstantScoreQuery(query), 0);
        case MUST_NOT:
          // no positive clauses
          return new MatchNoDocsQuery("pure negative BooleanQuery");
        default:
          throw new AssertionError();
      }
    }
  }


```

**2. 移除重复的filter和must_not clauses**

因为FILTER不影响评分,must_not不会被召回,所以重复的filter和must_not没有意义

```java
  // 移除重复的 filter 和 must_not clauses,因为FILTER不影响评分,must_not不会被召回,所以重复的filter和must_not没有意义
  {
    int clauseCount = 0;
    for (Collection<Query> queries : clauseSets.values()) {
      clauseCount += queries.size();
    }
    //clauseSets中filter和must_not使用的是HashSet会去重,而must和should使用的是Multiset(不会去重)，所以如果clauses和clauseSets的长度不等，只有可能是filter或者must_not有重复的
    if (clauseCount != clauses.size()) {
      //去除filter和must_not的重复clause后重新生成BooleanQuery
      BooleanQuery.Builder rewritten = new BooleanQuery.Builder();
      rewritten.setMinimumNumberShouldMatch(minimumNumberShouldMatch);
      for (Map.Entry<Occur, Collection<Query>> entry : clauseSets.entrySet()) {
        final Occur occur = entry.getKey();
        for (Query query : entry.getValue()) {
          rewritten.add(query, occur);
        }
      }
      return rewritten.build();
    }
  }
```



**3. 如果BooleanQuery中同一clause既在must又在must_not,则重写成MatchNoDocsQuery**

如果must和must_not有相同的子句,该BooleanQuery的结果必然是MatchNoDocsQuery

**4. 移除filter中的MatchAllDocsQuery **

filter不参与排序,只影响召回,匹配所有文档在filter中没有意义

```java
  //检查BooleanQuery中的Query是否有冲突(相同Query既在MUST又在MUST_NOT里)
  final Collection<Query> mustNotClauses = clauseSets.get(Occur.MUST_NOT);
  if (!mustNotClauses.isEmpty()) {
    final Predicate<Query> p = clauseSets.get(Occur.MUST)::contains;
    //1.判断MUST_NOT跟MUST或FILTER是否有相同的Query,如果有则直接返回MatchNoDocsQuery,既不会返回结果
    if (mustNotClauses.stream().anyMatch(p.or(clauseSets.get(Occur.FILTER)::contains))) {
      return new MatchNoDocsQuery("FILTER or MUST clause also in MUST_NOT");
    }
    //2.判断MUST_NOT中是否含有MatchAllDocsQuery,如果有则直接返回MatchNoDocsQuery
    if (mustNotClauses.contains(new MatchAllDocsQuery())) {
      return new MatchNoDocsQuery("MUST_NOT clause is MatchAllDocsQuery");
    }
  }

```

**5. 移除filter中与must中相同的clause**

如果filter和must中的子句有着相同的召回结果,由于flter不参与评分,所以需要移除filter中相同的子句

```java
  //从filters中移除既是FILTER又是MUST的Query(filter不参与评分，所以要移除filter中的)
  //如果filter中有MatchAllDocsQuery，则将其移除
  if (clauseSets.get(Occur.MUST).size() > 0 && clauseSets.get(Occur.FILTER).size() > 0) {
    final Set<Query> filters = new HashSet<Query>(clauseSets.get(Occur.FILTER));
    //如果filter中有MatchAllDocsQuery，则将其移除
    boolean modified = filters.remove(new MatchAllDocsQuery());
    //从filters中移除既是FILTER又是MUST的Query(filter不参与评分，所以要移除filter中的)
    modified |= filters.removeAll(clauseSets.get(Occur.MUST));
    //如果发生了重写，重新生成BooleanQuery
    if (modified) {
      BooleanQuery.Builder builder = new BooleanQuery.Builder();
      builder.setMinimumNumberShouldMatch(getMinimumNumberShouldMatch());
      for (BooleanClause clause : clauses) {
        if (clause.getOccur() != Occur.FILTER) {
          builder.add(clause);
        }
      }
      for (Query filter : filters) {
        builder.add(filter, Occur.FILTER);
      }
      return builder.build();
    }
  }


```



**6. 将filter和should中相同的clause转成must中的clause**

filter参与召回,不参与评分;should参与评分.不参与召回,而must即参与评分又参与召回

```java
  //将Filter和Should中的相同Query转成Must(filter代表必须命中，should代表参与打分，合在一起就是must的含义)
  if (clauseSets.get(Occur.SHOULD).size() > 0 && clauseSets.get(Occur.FILTER).size() > 0) {
    final Collection<Query> filters = clauseSets.get(Occur.FILTER);
    final Collection<Query> shoulds = clauseSets.get(Occur.SHOULD);
    //在intersection存储 既在should又在filter出现的Query
    Set<Query> intersection = new HashSet<>(filters);
    intersection.retainAll(shoulds);
    //重新生成BooleanQuery
    if (intersection.isEmpty() == false) {
      BooleanQuery.Builder builder = new BooleanQuery.Builder();
      int minShouldMatch = getMinimumNumberShouldMatch();

      for (BooleanClause clause : clauses) {
        if (intersection.contains(clause.getQuery())) {
          if (clause.getOccur() == Occur.SHOULD) {
            //将intersection里的Query从should改写成must
            builder.add(new BooleanClause(clause.getQuery(), Occur.MUST));
            //should被改写成must，而被改写的should本来一定会命中，故minimumNumberShouldMatch要减1
            minShouldMatch--;
          }
        } else {
          builder.add(clause);
        }
      }
      //重新设置故minimumNumberShouldMatch
      builder.setMinimumNumberShouldMatch(Math.max(0, minShouldMatch));
      return builder.build();
    }
  }


```

**7. 移除should中重复的clause,保留一个并累加权重**

should参与评分,不能单独去重,需要把权重累加

```java
  //should中的Query去重，并累加权重
  if (clauseSets.get(Occur.SHOULD).size() > 0 && minimumNumberShouldMatch <= 1) {
    Map<Query, Double> shouldClauses = new HashMap<>();
    for (Query query : clauseSets.get(Occur.SHOULD)) {
      double boost = 1;
      while (query instanceof BoostQuery) {
        BoostQuery bq = (BoostQuery) query;
        boost *= bq.getBoost();
        query = bq.getQuery();
      }
      shouldClauses.put(query, shouldClauses.getOrDefault(query, 0d) + boost);
    }
    if (shouldClauses.size() != clauseSets.get(Occur.SHOULD).size()) {
      BooleanQuery.Builder builder = new BooleanQuery.Builder().setMinimumNumberShouldMatch(minimumNumberShouldMatch);
      for (Map.Entry<Query, Double> entry : shouldClauses.entrySet()) {
        Query query = entry.getKey();
        float boost = entry.getValue().floatValue();
        if (boost != 1f) {
          query = new BoostQuery(query, boost);
        }
        builder.add(query, Occur.SHOULD);
      }
      for (BooleanClause clause : clauses) {
        if (clause.getOccur() != Occur.SHOULD) {
          builder.add(clause);
        }
      }
      return builder.build();
    }
  }

```

**8. 移除must中重复的clause,保留一个并累加权重**

```java
  //must中的Query去重，并累加权重
  if (clauseSets.get(Occur.MUST).size() > 0) {
    Map<Query, Double> mustClauses = new HashMap<>();
    //遍历所有must的Clause,如果有重复的，则权重累加
    for (Query query : clauseSets.get(Occur.MUST)) {
      double boost = 1;
      while (query instanceof BoostQuery) {
        BoostQuery bq = (BoostQuery) query;
        boost *= bq.getBoost();
        query = bq.getQuery();
      }
      mustClauses.put(query, mustClauses.getOrDefault(query, 0d) + boost);
    }
    // 如果must的clause合并后和合并前数量不等，说明有重复的clause, 那么需要对boost不等于1的query重写，然后跟其他的query一起写到新的BooleanQuery中。
    if (mustClauses.size() != clauseSets.get(Occur.MUST).size()) {
      BooleanQuery.Builder builder = new BooleanQuery.Builder().setMinimumNumberShouldMatch(minimumNumberShouldMatch);
      for (Map.Entry<Query, Double> entry : mustClauses.entrySet()) {
        Query query = entry.getKey();
        float boost = entry.getValue().floatValue();
        if (boost != 1f) {
          query = new BoostQuery(query, boost);
        }
        builder.add(query, Occur.MUST);
      }
      // 把其他不是MUST的clause重写添加到新的BooleanQuery中
      for (BooleanClause clause : clauses) {
        if (clause.getOccur() != Occur.MUST) {
          builder.add(clause);
        }
      }
      return builder.build();
    }
  }

```



**9. 如果BooleanQuery中must clause只有MatchAllDocsQuery,将filter和must_not改写成ConstantScoreQuery**

重写成ConstantScoreQuery跳过计算得分,提升性能

```java
  //如果只有一个must clause 并且该must clause是MatchAllDocsQuery 则将其改写成ConstantScoreQuery
  {
    final Collection<Query> musts = clauseSets.get(Occur.MUST);
    final Collection<Query> filters = clauseSets.get(Occur.FILTER);
    if (musts.size() == 1 && filters.size() > 0) {
      Query must = musts.iterator().next();
      float boost = 1f;
      //如果must是BoostQuery 得到boost 用于后续判断构建ConstantScoreQuery还是BoostQuery(ConstantScoreQuery,boost)
      if (must instanceof BoostQuery) {
        BoostQuery boostQuery = (BoostQuery) must;
        must = boostQuery.getQuery();
        boost = boostQuery.getBoost();
      }
      if (must.getClass() == MatchAllDocsQuery.class) {
        //should不参与构建ConstantScoreQuery
        BooleanQuery.Builder builder = new BooleanQuery.Builder();
        for (BooleanClause clause : clauses) {
          switch (clause.getOccur()) {
            case FILTER:
            case MUST_NOT:
              builder.add(clause);
              break;
            default:
              break;
          }
        }
        Query rewritten = builder.build();
        //构建ConstantScoreQuery
        rewritten = new ConstantScoreQuery(rewritten);
        //如果boost不为1,将ConstantScoreQuery构建成
        if (boost != 1f) {
          rewritten = new BoostQuery(rewritten, boost);
        }

        // 将之前得到的BoostQuery或者ConstantScoreQuery 和should clause重新构建BooleanQuery
        builder = new BooleanQuery.Builder().setMinimumNumberShouldMatch(getMinimumNumberShouldMatch()).add(rewritten, Occur.MUST);
        for (Query query : clauseSets.get(Occur.SHOULD)) {
          builder.add(query, Occur.SHOULD);
        }
        rewritten = builder.build();
        return rewritten;
      }
    }
  }
```



**10. 展平嵌套的should查询,为 block-max WAND 算法做准备 **

Wand算法通过计算每个词的贡献上限来估计文档的相关性上限，并与预设的阈值比较，进而跳过一些相关性一定达不到要求的文档，从而得到提速的效果。

```java
  // 展平嵌套的should查询,为 block-max WAND 算法做准备
  // Wand 算法通过计算每个词的贡献上限来估计文档的相关性上限，并与预设的阈值比较，进而跳过一些相关性一定达不到要求的文档，从而得到提速的效果。
  if (minimumNumberShouldMatch <= 1) {
    BooleanQuery.Builder builder = new BooleanQuery.Builder();
    builder.setMinimumNumberShouldMatch(minimumNumberShouldMatch);
    boolean actuallyRewritten = false;
    for (BooleanClause clause : clauses) {
      if (clause.getOccur() == Occur.SHOULD && clause.getQuery() instanceof BooleanQuery) {
        BooleanQuery innerQuery = (BooleanQuery) clause.getQuery();
        if (innerQuery.isPureDisjunction()) {
          actuallyRewritten = true;
          for (BooleanClause innerClause : innerQuery.clauses()) {
            builder.add(innerClause);
          }
        } else {
          builder.add(clause);
        }
      } else {
        builder.add(clause);
      }
    }
    if (actuallyRewritten) {
      return builder.build();
    }
  }

```



BooleanQuery会对包含的每个BooleanClause进行重写

```java
@Override
public Query rewrite(IndexReader reader) throws IOException {

  /*-------------------------------------------------------*/
  ....//重写逻辑1
  /*-------------------------------------------------------*/
  // 递归调用
  {
    BooleanQuery.Builder builder = new BooleanQuery.Builder();
    builder.setMinimumNumberShouldMatch(getMinimumNumberShouldMatch());
    //递归遍历clauses，是否有Query发生了重写
    boolean actuallyRewritten = false;
    //BooleanQuery继承了Iterable接口,this就是遍历clauses
    for (BooleanClause clause : this) {
      Query query = clause.getQuery();
      Query rewritten = query.rewrite(reader);
      if (rewritten != query) {
        // rewrite clause
        actuallyRewritten = true;
        builder.add(rewritten, clause.getOccur());
      } else {
        // leave as-is
        builder.add(clause);
      }
    }
    //如果clauses中有Query发生了重写,就要生成新的BooleanQuery
    if (actuallyRewritten) {
      return builder.build();
    }
  }
  /*-------------------------------------------------------*/
  ...//重写逻辑2-10
  /*-------------------------------------------------------*/
  return super.rewrite(reader);
}
```