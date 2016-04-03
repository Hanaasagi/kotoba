#XPath injection  
Edit by Qsaka  

----  
本文以HCTF2015 injection一题为例，来分析 XPath 注入攻击。  
##HCTF2015 injection  
题目源码如下  

	//index.php  

	<?php
    $re = array('and','or','count','select','from','union','group','by','limit','insert','where','order','alter','delete','having','max','min','avg','sum','sqrt','rand','concat','sleep');
    setcookie('injection','c3FsaSBpcyBub3QgdGhlIG9ubHkgd2F5IGZvciBpbmplY3Rpb24=',time()+100000);
    if(file_exists('t3stt3st.xml')) {
        $xml = simplexml_load_file('t3stt3st.xml');
        $user=$_GET['user'];
    	$user=str_replace($re, ' ', $user);
      //  $user=str_replace("'", "&apos", $user);
        $query="user/username[@name='".$user."']";
      
        $ans = $xml->xpath($query);
        foreach($ans as $x => $x_value)
        {
            echo $x.":  " . $x_value;
            echo "<br />";
        }
    }
    ?>  

	//t3stt3st.xml  

	<?xml version="1.0" encoding="utf-8"?>
    <root1>
        <user>
            <username name='user1'>user1</username>
            <key>KEY:1</key>
            <username name='user2'>user2</username>
            <key>KEY:2</key>
            <username name='user3'>user3</username>
            <key>KEY:3</key>
            <username name='user4'>user4</username>
            <key>KEY:4</key>
            <username name='user5'>user5</username>
            <key>KEY:5</key>
            <username name='user6'>user6</username>
            <key>KEY:6</key>
            <username name='user7'>user7</username>
            <key>KEY:7</key>
            <username name='user8'>user8</username>
            <key>KEY:8</key>
            <username name='user9'>user9</username>
            <key>KEY:9</key>
        </user>
        <hctfadmin>    
            <username name='hctf1'>hctf</username>
            <key>flag:hctf{Dd0g_fac3_t0_k3yboard233}</key>
        </hctfadmin>
    </root1>

##XPath基础  
XPath 是一门在 XML 文档中查找信息的语言。
####结点
在 XPath 中，有七种类型的节点：元素、属性、文本、命名空间、处理指令、注释以及文档节点（或称为根节点）。节点之间存在父、子、先辈、后代、同胞关系，以t3stt3st.xml为例  
根节点`<root1>` 、元素节点`<user><username><key><hctfadmin>` 、属性节点`name='user1'`  
`<root1>`是`<user>`和`<htcfadmin>`的父节点，同时也是`<user><hctfadmin><username><key>`的先辈。`<username>`和`<key>`是同胞节点。
####路径表达式  
>|表达式       |     描述                                                |
>|:----------:|:-------------------------------------------------------:|
>|nodename    |	    选取此节点的所有子节点                                 |
>|/ 		  |     从根节点选取                                          |
>|// 	       |     从匹配选择的当前节点选择文档中的节点，而不考虑它们的位置  |
>|.            |    选取当前节点                                         |
>|..           |   选取当前节点的父节点                                   |
>|@ 	      |     选取属性                                            |
>|[]          |    查找某个特定的节点或者包含某个指定的值的节点。            |

####通配符  
>|通配符   | 描述             |
>|:------:|:----------------:|
>|* 	  | 匹配任何元素节点   |
>|@* 	  | 匹配任何属性节点   |
>|node()  | 匹配任何类型的节点 | 

####选取若干路径  
通过在路径表达式中使用“|”运算符，可以选取若干个路径。  

##XPath注入
从index.php的源码可知XPath查询语句为`$query="user/username[@name='".$user."']"`， 且`$user`经过关键字替换。但是黑名单`$re`中都为SQL关键字，所以并不影响对XPath进行注入。我们可以构造payload如 `']|//*|zzz['`来进行注入，获取文档中的所有元素节点。  
##Reference  
[HCTF2015 writeup](https://github.com/hduisa/hctf2015-all-problems)  
[XPath基础教程](http://www.w3school.com.cn/xpath/)