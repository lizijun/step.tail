    最近，实施反馈有条sql执行很慢。由于篇幅过长和对公司隐私的保护，我对sql的查询字段进行了精简，表名也马赛克了一下。
``` sql
SELECT
	T1.UUID AS uuid,
	T3.RULE_CODE AS ruleCode,
	T2.RISK_TYPE AS riskType,
	ID AS id
FROM
	(
	SELECT
		* 
	FROM
		(
		SELECT
			row_.*,
			ROWNUM rownum_ 
		FROM
			(
			SELECT
				ID,
				UUID
			FROM
				TEST_MASAIKE 
			WHERE
				OPER_USER = 'admin' 
				AND STATUS = '0' 
				AND TRANS_TIME >= to_date( '2019-11-09 00:00:00', 'YYYY-MM-DD 
				HH24:MI:SS' ) 
				AND TRANS_TIME <= to_date( '2019-11-15 23:59:59', 'YYYY-MM-DD HH24:MI:SS' ) 
			ORDER BY
				TRANS_TIME DESC 
			) row_ 
		) 
	WHERE
		rownum_ > 0 
		AND rownum_ <= 0 + 25 
	) T1
	LEFT JOIN (
	SELECT
		UUID,
		WM_CONCAT ( DISTINCT RISK_TYPE ) AS RISK_TYPE 
	FROM
		TEST_MASAIKE_RISKTYPE 
	WHERE
		TRANS_TIME >= to_date( '2019-11-09 00:00:00', 'YYYY-MM-DD HH24:MI:SS' ) 
		AND TRANS_TIME <= to_date( '2019-11-15 
		23:59:59', 'YYYY-MM-DD HH24:MI:SS' ) 
	GROUP BY
		UUID 
	) T2 ON T1.UUID = T2.UUID
	LEFT JOIN (
	SELECT
		UUID,
		WM_CONCAT ( DISTINCT RULE_CODE ) AS RULE_CODE 
	FROM
		TEST_MASAIKE_RISKS 
	WHERE
		TRANS_TIME >= to_date( '2019-11-09 00:00:00', 'YYYY-MM-DD HH24:MI:SS' ) 
		AND TRANS_TIME <= to_date( '2019-11-15 
		23:59:59', 'YYYY-MM-DD HH24:MI:SS' ) 
	GROUP BY
		UUID 
	) T3 ON T1.UUID = T3.UUID 

```

**查看各个表的数据量**
```
SQL> select count(*) from TEST_MASAIKE_RISKS;

  COUNT(*)
----------
   1593267

SQL> select count(*) from TEST_MASAIKE;

  COUNT(*)
----------
    881511

SQL> select count(*) from TEST_MASAIKE_RISKTYPE;

  COUNT(*)
----------
    312277

```

**通过表中数据看出，数据量不大，但是执行几十秒才能返回。如果按以下三个步骤执行sql，那么不应该会这么慢**。

- 第一个子查询T1会分页查出25条数据，所以第一个子查询效率应该不会太差。

- 通过T1的uuid索引查询T2中的数据。

- 通过T1的uuid索引查询T3中的数据。

  
#### 第一步，建索引
**首先看了下表中的索引，发现TEST_MASAIKE_RISKTYPE的uuid未建索引，开发给出的理由是TEST_MASAIKE_RISKTYPE和TEST_MASAIKE是多对一的关系，可能存在uuid相同的情况，重复数据可能比较多。但是，理论上不重复的数据只要超过50%，我认为都可以建索引。**

**加上索引后，重新获取了执行计划（这里需要说明获取执行计划是根据sqlId执行dbms_xplan.display_cursor函数获取的,因为display函数获取的执行计划不是真实执行的，是根据oracle的统计信息生成的。）**
```
select * from table(dbms_xplan.display_cursor(sql_id,null)); 


Plan hash value: 3382258508
 
-------------------------------------------------------------------------------------------------------------------
| Id  | Operation                          | Name                         | Rows  | Bytes | Cost (%CPU)| Time     |
-------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                   |                              |       |       |    10 (100)|          |
|   1 |  SORT ORDER BY                     |                              |     1 |  8493 |    10  (40)| 00:00:01 |
|*  2 |   HASH JOIN                        |                              |     1 |  8493 |     9  (34)| 00:00:01 |
|*  3 |    HASH JOIN                       |                              |     1 |  6458 |     6  (34)| 00:00:01 |
|*  4 |     VIEW                           |                              |     1 |  4423 |     2   (0)| 00:00:01 |
|   5 |      COUNT                         |                              |       |       |            |          |
|   6 |       VIEW                         |                              |     1 |  4410 |     2   (0)| 00:00:01 |
|   7 |        TABLE ACCESS BY INDEX ROWID | TEST_MASAIKE            |     1 |   372 |     2   (0)| 00:00:01 |
|*  8 |         INDEX RANGE SCAN DESCENDING| IDX_TEST_CHK_TRANS_TIME   |     1 |       |     1   (0)| 00:00:01 |
|   9 |     VIEW                           |                              |     1 |  2035 |     3  (34)| 00:00:01 |
|  10 |      SORT GROUP BY                 |                              |     1 |    48 |     3  (34)| 00:00:01 |
|  11 |       TABLE ACCESS BY INDEX ROWID  | TEST_MASAIKE_RISKTYPE   |     1 |    48 |     2   (0)| 00:00:01 |
|* 12 |        INDEX RANGE SCAN            | IDX_CHK_RISK_TYPE_TRANS_TIME |     1 |       |     1   (0)| 00:00:01 |
|  13 |    VIEW                            |                              |     1 |  2035 |     3  (34)| 00:00:01 |
|  14 |     SORT GROUP BY                  |                              |     1 |    55 |     3  (34)| 00:00:01 |
|  15 |      TABLE ACCESS BY INDEX ROWID   | TEST_MASAIKE_RISKS      |     1 |    55 |     2   (0)| 00:00:01 |
|* 16 |       INDEX RANGE SCAN             | IDX_CHK_RISK_TRANS_TIME      |     1 |       |     1   (0)| 00:00:01 |
-------------------------------------------------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   2 - access("T2"."UUID"="T3"."UUID")
   3 - access("from$_subquery$_002"."UUID"="T2"."UUID")
   4 - filter(("ROWNUM_">0 AND "ROWNUM_"<=25))
   8 - access("TRANS_TIME"<=TIMESTAMP' 2019-11-13 23:59:59' AND "TRANS_TIME">=TIMESTAMP' 2019-11-01 
              00:00:00')
  12 - access("TRANS_TIME">=TIMESTAMP' 2019-11-01 00:00:00' AND "TRANS_TIME"<=TIMESTAMP' 2019-11-13 
              23:59:59')
  16 - access("TRANS_TIME">=TIMESTAMP' 2019-11-01 00:00:00' AND "TRANS_TIME"<=TIMESTAMP' 2019-11-13 
              23:59:59')
```

通过执行计划可以看出，三个子查询分别通过时间索引查询，其中第二个子查询会group by得到结果后，和T1进行hash join，hash join的结果会和第三个子查询group by得到结果再次hash join，这个执行计划和我们想象的显然不一样，事实也证明，查询效率依然没有改善。
虽然三个子查询均走了时间索引，但是由于该时间范围内的数据比较大（原因是造的测试数据比较集中），索引效果很差。
看到这个执行计划，我先是疑惑为什么加的uuid索引没什么鸟用，于是我想先看看索引是否生效。

```·sql
-- 先通过一条简单的sql先看下索引是否生效
select * from TEST_MASAIKE_RISKTYPE where uuid = '5ed10315a5bd424ab639dd0ada837cbf';

 Plan Hash Value  : 3608587963 

----------------------------------------------------------------------------------
| Id  | Operation           | Name              | Rows | Bytes | Cost | Time     |
----------------------------------------------------------------------------------
|   0 | SELECT STATEMENT    |                   |    1 |   131 | 4958 | 00:01:00 |
| * 1 |   TABLE ACCESS FULL | TEST_MASAIKE |    1 |   131 | 4958 | 00:01:00 |
----------------------------------------------------------------------------------

Predicate Information (identified by operation id):
------------------------------------------
* 1 - filter("UUID"='5ed10315a5bd424ab639dd0ada837cbf')
```
通过该sql的查询计划发现也未走索引，想到应该是统计信息不准确，导致未走索引。通过执行
DBMS_STATS.GATHER_TABLE_STATS(ownname VARCHAR2, tabname VARCHAR2)，重新统计信息，然后发现索引生效，此时再看之前sql的执行计划。

```
  Plan Hash Value  : 270543148 

--------------------------------------------------------------------------------------------------------------
| Id   | Operation                  | Name                       | Rows    | Bytes      | Cost    | Time     |
--------------------------------------------------------------------------------------------------------------
|    0 | SELECT STATEMENT           |                            |  880120 | 6973190760 | 2038362 | 06:47:41 |
|    1 |   SORT ORDER BY            |                            |  880120 | 6973190760 | 2038362 | 06:47:41 |
|  * 2 |    HASH JOIN RIGHT OUTER   |                            |  880120 | 6973190760 |  587958 | 01:57:36 |
|    3 |     VIEW                   |                            |  312277 |  635483695 |    4668 | 00:00:57 |
|    4 |      SORT GROUP BY         |                            |  312277 |   15613850 |    4668 | 00:00:57 |
|  * 5 |       TABLE ACCESS FULL    | TEST_MASAIKE_RISKTYPE |  312277 |   15613850 |     822 | 00:00:10 |
|  * 6 |     HASH JOIN RIGHT OUTER  |                            |  880120 | 5182146560 |  307598 | 01:01:32 |
|    7 |      VIEW                  |                            |  881856 | 1794576960 |   30677 | 00:06:09 |
|    8 |       SORT GROUP BY        |                            |  881856 |   47620224 |   30677 | 00:06:09 |
|  * 9 |        TABLE ACCESS FULL   | TEST_MASAIKE_RISKS    | 1593267 |   86036418 |   15657 | 00:03:08 |
| * 10 |      VIEW                  |                            |  880120 | 3391102360 |   30778 | 00:06:10 |
|   11 |       COUNT                |                            |         |            |         |          |
|   12 |        VIEW                |                            |  880120 | 3379660800 |   30778 | 00:06:10 |
|   13 |         SORT ORDER BY      |                            |  880120 |  115295720 |   30778 | 00:06:10 |
| * 14 |          TABLE ACCESS FULL | TEST_MASAIKE          |  880120 |  115295720 |    4984 | 00:01:00 |
--------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
------------------------------------------
* 2 - access("from$_subquery$_002"."UUID"="T2"."UUID"(+))
* 5 - filter("TRANS_TIME">=TIMESTAMP' 2019-11-09 00:00:00' AND "TRANS_TIME"<=TIMESTAMP' 2019-11-15 23:59:59')
* 6 - access("from$_subquery$_002"."UUID"="T3"."UUID"(+))
* 9 - filter("TRANS_TIME">=TIMESTAMP' 2019-11-09 00:00:00' AND "TRANS_TIME"<=TIMESTAMP' 2019-11-15 23:59:59')
* 10 - filter("ROWNUM_">0 AND "ROWNUM_"<=25)
* 14 - filter("TRANS_TIME"<=TIMESTAMP' 2019-11-15 23:59:59' AND "OPER_USER"='admin' AND "STATUS"=0 AND "TRANS_TIME">=TIMESTAMP' 2019-11-09 00:00:00')
```

这个执行计划，走了几个全表扫描，最后还是hash join， 执行耗时也非常大。即使把时间范围缩小，仍然只是走时间索引。

#### 第二步，改写sql
其实可以从执行计划中看出第二个和第三个子查询比较耗时，于是我想是不是能把聚合函数往外提，不要在子查询中group by。 通过改写，能够将sql的耗时降到3秒。
具体的方式，是将第二个和第三个子查询去除，直接关联表查询，这样可以避免先执行子查询的消耗。
```
SELECT
  UUID AS uuid,
  WM_CONCAT ( RULECODE ) AS ruleCode,
  WM_CONCAT ( RISKTYPE ) AS riskType,
  ID AS id 
FROM
  (
  SELECT
    T1.UUID AS uuid,
    T3.RULE_CODE AS ruleCode,
    T2.RISK_TYPE AS riskType,
    T1.ID AS id 
  FROM
    (
    SELECT
      * 
    FROM
      (
      SELECT
        row_.*,
        ROWNUM rownum_ 
      FROM
        (
        SELECT
          ID,
          UUID 
        FROM
          TEST_MASAIKE 
        WHERE
          OPER_USER = 'admin' 
          AND STATUS = '0' 
          AND TRANS_TIME >= to_date( '2019-11-09 00:00:00', 'YYYY-MM-DD 
          HH24:MI:SS' ) 
          AND TRANS_TIME <= to_date( '2019-11-15 23:59:59', 'YYYY-MM-DD HH24:MI:SS' ) 
        ORDER BY
          TRANS_TIME DESC 
        ) row_ 
      ) 
    WHERE
      rownum_ > 0 
      AND rownum_ <= 0 + 25 
    ) T1
    LEFT JOIN TEST_MASAIKE_RISKTYPE T2 ON T1.UUID = T2.UUID
    LEFT JOIN TEST_MASAIKE_RISKS T3 ON T1.UUID = T3.UUID 
  ORDER BY
    T2.TRANS_TIME DESC 
  ) G 
GROUP BY
  G.UUID,
  G.id
  ```
改写后的执行计划如下：
```
 Plan Hash Value  : 1078327277 

------------------------------------------------------------------------------------------------------------
| Id   | Operation                   | Name                       | Rows    | Bytes     | Cost  | Time     |
------------------------------------------------------------------------------------------------------------
|    0 | SELECT STATEMENT            |                            |  312277 |  39971456 | 78066 | 00:15:37 |
|    1 |   SORT GROUP BY             |                            |  312277 |  39971456 | 78066 | 00:15:37 |
|    2 |    MERGE JOIN OUTER         |                            | 1590131 | 203536768 | 78066 | 00:15:37 |
|    3 |     MERGE JOIN OUTER        |                            |  880120 |  74810200 | 44907 | 00:08:59 |
|    4 |      SORT JOIN              |                            |  880120 |  40485520 | 40948 | 00:08:12 |
|  * 5 |       VIEW                  |                            |  880120 |  40485520 | 30778 | 00:06:10 |
|    6 |        COUNT                |                            |         |           |       |          |
|    7 |         VIEW                |                            |  880120 |  29043960 | 30778 | 00:06:10 |
|    8 |          SORT ORDER BY      |                            |  880120 | 115295720 | 30778 | 00:06:10 |
|  * 9 |           TABLE ACCESS FULL | TEST_MASAIKE          |  880120 | 115295720 |  4984 | 00:01:00 |
| * 10 |      SORT JOIN              |                            |  312277 |  12178803 |  3960 | 00:00:48 |
|   11 |       TABLE ACCESS FULL     | TEST_MASAIKE_RISKTYPE |  312277 |  12178803 |   821 | 00:00:10 |
| * 12 |     SORT JOIN               |                            | 1593267 |  68510481 | 33158 | 00:06:38 |
|   13 |      TABLE ACCESS FULL      | TEST_MASAIKE_RISKS    | 1593267 |  68510481 | 15651 | 00:03:08 |
------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
------------------------------------------
* 5 - filter("ROWNUM_">0 AND "ROWNUM_"<=25)
* 9 - filter("TRANS_TIME"<=TIMESTAMP' 2019-11-15 23:59:59' AND "OPER_USER"='admin' AND "STATUS"=0 AND "TRANS_TIME">=TIMESTAMP' 2019-11-09 00:00:00')
* 10 - access("from$_subquery$_003"."UUID"="T2"."UUID"(+))
* 10 - filter("from$_subquery$_003"."UUID"="T2"."UUID"(+))
* 12 - access("from$_subquery$_003"."UUID"="T3"."UUID"(+))
* 12 - filter("from$_subquery$_003"."UUID"="T3"."UUID"(+))
```

从执行计划来看，依旧有几个全表扫描，而且之前的hash join消除了，但是出现了merge join。另外，有个开发告诉我，有些查询条件是TEST_MASAIKE_RISKTYPE和TEST_MASAIKE_RISKS表字段，这就意味着我们简单的用这种方式改写，这种改写很容易导致与需求不一致。于是还是放弃了这个优化方式，实际上，我认为按照我们之前预想的方式，使用nested loops关联才是最佳的。
#### 第二步优化失败，尝试第三步优化
优化器为什么没有采用nested loops， 网上看到说由于子查询不是物理表，所以可能统计信息并不准确，导致优化器无法产生我们预想的执行计划。这个说的通，但是有待验证。
既然优化器不使用nested loops，我们就自己指定nested loops，oracle有一种hint的方式，可以调整执行计划。
我们这里使用use_nl。

```
SELECT
    /*+use_nl(T2,T3) */
	T1.UUID AS uuid,
	T3.RULE_CODE AS ruleCode,
	T2.RISK_TYPE AS riskType,
	ID AS id
FROM
	(
	SELECT
		* 
	FROM
		(
		SELECT
			row_.*,
			ROWNUM rownum_ 
		FROM
			(
			SELECT
				ID,
				UUID
			FROM
				TEST_MASAIKE 
			WHERE
				OPER_USER = 'admin' 
				AND STATUS = '0' 
				AND TRANS_TIME >= to_date( '2019-11-09 00:00:00', 'YYYY-MM-DD 
				HH24:MI:SS' ) 
				AND TRANS_TIME <= to_date( '2019-11-15 23:59:59', 'YYYY-MM-DD HH24:MI:SS' ) 
			ORDER BY
				TRANS_TIME DESC 
			) row_ 
		) 
	WHERE
		rownum_ > 0 
		AND rownum_ <= 0 + 25 
	) T1
	LEFT JOIN (
	SELECT
		UUID,
		WM_CONCAT ( DISTINCT RISK_TYPE ) AS RISK_TYPE 
	FROM
		TEST_MASAIKE_RISKTYPE 
	WHERE
		TRANS_TIME >= to_date( '2019-11-09 00:00:00', 'YYYY-MM-DD HH24:MI:SS' ) 
		AND TRANS_TIME <= to_date( '2019-11-15 
		23:59:59', 'YYYY-MM-DD HH24:MI:SS' ) 
	GROUP BY
		UUID 
	) T2 ON T1.UUID = T2.UUID
	LEFT JOIN (
	SELECT
		UUID,
		WM_CONCAT ( DISTINCT RULE_CODE ) AS RULE_CODE 
	FROM
		TEST_MASAIKE_RISKS 
	WHERE
		TRANS_TIME >= to_date( '2019-11-01 10:00:00', 'YYYY-MM-DD HH24:MI:SS' ) 
		AND TRANS_TIME <= to_date( '2019-11-15 
		23:59:59', 'YYYY-MM-DD HH24:MI:SS' ) 
	GROUP BY
		UUID 
	) T3 ON T1.UUID = T3.UUID 
	


```
改写的sql就是这样的，然后看看执行计划。
```
 Plan Hash Value  : 1044424505 

----------------------------------------------------------------------------------------------------------------------
| Id   | Operation                          | Name                        | Rows   | Bytes      | Cost    | Time     |
----------------------------------------------------------------------------------------------------------------------
|    0 | SELECT STATEMENT                   |                             | 880120 | 6915102840 | 7953943 | 26:30:48 |
|    1 |   NESTED LOOPS OUTER               |                             | 880120 | 6915102840 | 7953943 | 26:30:48 |
|    2 |    NESTED LOOPS OUTER              |                             | 880120 | 5153102600 | 3552182 | 11:50:27 |
|  * 3 |     VIEW                           |                             | 880120 | 3391102360 |   30778 | 00:06:10 |
|    4 |      COUNT                         |                             |        |            |         |          |
|    5 |       VIEW                         |                             | 880120 | 3379660800 |   30778 | 00:06:10 |
|    6 |        SORT ORDER BY               |                             | 880120 |  115295720 |   30778 | 00:06:10 |
|  * 7 |         TABLE ACCESS FULL          | TEST_MASAIKE           | 880120 |  115295720 |    4984 | 00:01:00 |
|    8 |     VIEW PUSHED PREDICATE          |                             |      1 |       2002 |       4 | 00:00:01 |
|  * 9 |      FILTER                        |                             |        |            |         |          |
|   10 |       SORT GROUP BY                |                             |      1 |         50 |         |          |
| * 11 |        TABLE ACCESS BY INDEX ROWID | TEST_MASAIKE_RISKTYPE  |      1 |         50 |       4 | 00:00:01 |
| * 12 |         INDEX RANGE SCAN           | IDX_CHK_RISK_TYPE_UUID      |      1 |            |       3 | 00:00:01 |
|   13 |    VIEW PUSHED PREDICATE           |                             |      1 |       2002 |       5 | 00:00:01 |
| * 14 |     FILTER                         |                             |        |            |         |          |
|   15 |      SORT GROUP BY                 |                             |      1 |         54 |         |          |
| * 16 |       TABLE ACCESS BY INDEX ROWID  | TEST_MASAIKE_RISKS     |      2 |        108 |       5 | 00:00:01 |
| * 17 |        INDEX RANGE SCAN            | IDX_TEST_CHK_RISK_CHK_ID |      2 |            |       3 | 00:00:01 |
----------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
------------------------------------------
* 3 - filter("ROWNUM_">0 AND "ROWNUM_"<=25)
* 7 - filter("TRANS_TIME"<=TIMESTAMP' 2019-11-15 23:59:59' AND "OPER_USER"='admin' AND "STATUS"=0 AND "TRANS_TIME">=TIMESTAMP' 2019-11-09 00:00:00')
* 9 - filter(COUNT(*)>0)
* 11 - filter("TRANS_TIME">=TIMESTAMP' 2019-11-09 00:00:00' AND "TRANS_TIME"<=TIMESTAMP' 2019-11-15 23:59:59')
* 12 - access("UUID"="from$_subquery$_002"."UUID")
* 14 - filter(COUNT(*)>0)
* 16 - filter("TRANS_TIME">=TIMESTAMP' 2019-11-01 10:00:00' AND "TRANS_TIME"<=TIMESTAMP' 2019-11-15 23:59:59')
* 17 - access("UUID"="from$_subquery$_002"."UUID")
```

我们可以看出连接方式终于变成了nested loops，而且TEST_MASAIKE_RISKS和TEST_MASAIKE_RISKTYPE都走了索引查询。
这时再看看查询耗时在2秒左右。通过执行计划可以看出优化后的主要耗时应该在于TEST_MASAIKE的全表扫描，这里是因为数据比较集中，所以未走索引。

由于水平实在有限，加上没有太多时间去优化，只能暂时以加hint的方式来优化。加hint其实有很多缺点，它改变了执行计划，意味着优化器不再根据统计信息进行自动优化，在不同的环境或者随着数据量的改变很有可能降低执行效率。
如果有更好的解决方式，希望大家不吝赐教。
