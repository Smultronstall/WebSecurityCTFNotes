# 题目：Web2

[![p9cW3hd.md.png](https://s1.ax1x.com/2023/05/14/p9cW3hd.md.png)](https://imgse.com/i/p9cW3hd)

+ 打开网页，可以发现是一个解密题目，主要看字符串经过了怎样的加密过程，网页的加密代码如下：

  ```php
  <?php
  $miwen="a1zLbgQsCESEIqRLwuQAyMwLyq2L5VwBxqGA3RQAyumZ0tmMvSGM2ZwB4tws";
  
  function encode($str){
      $_o=strrev($str);
      // echo $_o;
          
      for($_0=0;$_0<strlen($_o);$_0++){
         
          $_c=substr($_o,$_0,1);
          $__=ord($_c)+1;
          $_c=chr($__);
          $_=$_.$_c;   
      } 
      return str_rot13(strrev(base64_encode($_)));
  }
  
  highlight_file(__FILE__);
  /*
     逆向加密算法，解密$miwen就是flag
  */
  ?> 
  ```

## 代码分析

+ `strrev(str)`：翻转字符串`str`，比如字符串是"Hello"，则`strrev(str)="olleH"`，对于它的解密就使用它本身，再次翻转字符串即可；
+ `ord(str)`：获取字符串`str`的`ASCII`码值，返回值是一个`INT`；
+ `chr(int)`：将`int`值转化为对应的`ASCII`码，返回值是一个`STR`。可见`ord(str)`与`chr(int)`是一组相互加密解密的函数，通过调用彼此能获得明文；
+ `str_rot13(str)`：`ROT13` 编码把每一个字母在字母表中向前移动 13 个字母。数字和非字母字符保持不变。加密解密都是使用同一个函数；
+ `base64_encode(str)`：对应的解密函数是`base64_decode(str)`；

通过分析源码我们可以得知，它的加密过程是：

1. 先对原字符串进行`strrev`翻转；
2. 在`for`循环中，可以分析得知`$_c=substr($_o, $_0, 1)`就是每次截取一个字符，按顺序向后截取；
3. 对于截取到的字符，先获取它的`ASCII`码值，再加上1，将结果转化为`ASCII`码字符，由这些新的`ASCII`码字符按顺序拼接成`$_`；
4. 对于拼接完成的`$_`，先进行`base64_encode`再使用`strrev`翻转字符，最终进行一次`str_rot13`编码，这便是最终的密文；

## 解题过程

+ 根据上面的分析，我们的解码思路如下：

  1. 将密文首先进行`str_rot13`解码，再使用`strrev`翻转，最终`base64_decode`解码得到`for`循环中处理的字符串；
  2. 通过`for`循环进行逐个字符解码，依然是使用`$_c=substr($_o, $_0, 1)`每次截取一个字符操作；
  3. 对于截取到的字符，获取它的`ASCII`码值，再减去1，恢复其真实值，再将结果转化为`ASCII`码字符；
  4. 逐个字符解码后拼接形成`$_`，最后在`strrev`翻转得到真实的明文；

+ 解码脚本如下（去掉`echo`的注释可以看到详细的解码步骤）：

  ```php
  <?php
      $miwen = "a1zLbgQsCESEIqRLwuQAyMwLyq2L5VwBxqGA3RQAyumZ0tmMvSGM2ZwB4tws";
      echo decode($miwen);
      
      function decode($str){
          $_o = base64_decode(strrev(str_rot13($str)));
          for($_0=0;$_0<strlen($_o);$_0++){
              $_c = substr($_o,$_0,1);
              #echo "cut str:" . $_c ."<br>";
              $__ = ord($_c) - 1;
              #echo "real ASCII:" . $__ . "<br>";
              $_c = chr($__);
              #echo "real char:" . $_c . "<br>";
              $_ = $_.$_c;
              #echo "added mingwen:" . $_ . "<br>";
          }
          return strrev($_);
      }
  ?>
  ```

+ 成功得到flag：

  [![p9cfU2R.png](https://s1.ax1x.com/2023/05/14/p9cfU2R.png)](https://imgse.com/i/p9cfU2R)

