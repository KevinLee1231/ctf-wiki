# easyweb

## 题目简述

题目是一个基于 CodeIgniter 和 SmartyBC 的 PHP Web 题。用户信息查询处存在过滤顺序错误导致的二次 SQL 注入，注入结果会被拼接进 `data://` 模板资源并交给 Smarty 渲染，因此可以控制模板内容。预期链条是在限制直接执行 PHP 的情况下，分析 Smarty 对 `data`、`phar`、`php` 等协议的处理差异，通过 `php:phar://` 进入 `Smarty_Internal_Resource_Php` 的文件路径检查逻辑，借 `is_file` 触发 phar 反序列化，再用 CodeIgniter 组件中的 POP 链完成 RCE。

题目资料地址：https://github.com/ox1234/d3ctf_easyweb

公开源码中保留了 `CodeIgniter-3.1.11` 应用、Docker/MySQL 环境和 `www/index.php` 入口。`User::index()` 会从 `Render_model` 取用户视图内容，拼成 `data:,` 后直接交给 `$this->ci_smarty->display()`；`File::index()` 的上传目录为 `/tmp/{session userId}/`，只去掉文件名中的 `..` 并禁止点号开头文件。控制器构造函数中设置了 `Smarty_Security`，禁用 `php_functions`、设置 `php_handling = Smarty::PHP_REMOVE` 并清空 `streams`，这解释了为什么预期解要围绕 Smarty 协议处理、`php:phar://` 和 phar 反序列化触发点展开。

## 解题过程

#### 前言

这道题思路其实就是，如果在php中遇到了模版注入，但是限制了不允许执行php代码的时候，怎么通过模版注入达到RCE的效果

#### 考点

预期：

非预期：

##### 二次注入

可以看到，在 `Render_model::get_view()` 里，会先用 session 中的 `userId` 从 `userTable` 查出 `username`，再依次调用 `sql_safe()` 和 `safe_render()` 处理用户名，最后拼进 `SELECT userView FROM userRender WHERE username='$username'`。`safe_render()` 只删除 `{`、`}`，但它在 SQL 黑名单之后才执行，因此 `se{lect` 这类写法能先绕过黑名单，再被删成 `select` 进入第二次查询。

`User::index()` 拿到 `get_view()` 返回的 `userView` 后，会拼成 `data:,<userView>` 并直接传给 `$this->ci_smarty->display()`，所以二次 SQL 注入的目标不是直接打印数据，而是控制数据库里的模板内容，让 Smarty 解析攻击者写入的模板。
然后模版的标签 `{` 可以通过 SQL 的 16 进制绕过，这两步应该是很容易看出来的

##### 模版注入

可以看到，index这里调用了get\_view方法，然后将获取到的结果拼接到data协议中，之后将整个data协议的内容直接插入到了display函数中，很容易发现这里有一个模版注入的问题，我们只要通过union select就可以控制整个模版的内容。

这里有一点要注意的就是，我们需要将我们控制的字符串放到返回结果的第一行中，因为union select是在原先查询的下面添加一行结果，所以用limit 1,1即可返回我们控制的模版字符串

##### 正文开始

因为当时将smarty嵌入到CI框架中，是根据网上其他师傅的博客来写的，但是因为参考文章的时间可能比较久，导致在整合的时候，其实是用的smartyBC，这是一个兼容低版本smarty的引擎，而不是最新的Smarty引擎

而这个SmartyBC就是一切非预期的开始

###### 0x01 非预期解

在构造方法中，为了不让各位师傅直接通过SSTI执行php命令，在这里设置了Smarty引擎的安全规则，默认不允许任何php方法，不允许php脚本的解析，看起来是没有什么问题，但是因为没有仔细看官方文档，结果发现还是可以执行php代码

结果，php\_handling 不能限制 `{php}{/php}` 这样的标签，所以可以直接通过 `phpinfo();` 验证代码执行。

这里我犯了两个错误：

1. 没有发现php\_handling是不能限制{php}{/php}的
2. 使用的是SmartyBC而不是Smarty，在Smarty中，这个标签已经被废弃了

所以直接：  
`data:,phpinfo();`

##### 0x02 预期解

###### smarty对协议的处理

在CI中添加一个test路由，直接在display中调用data协议，使用xdebug跟一下display的逻辑

可以看到这里调用了createTemplate函数，根据函数名，这里就是创建我们的模版的地方，跟进去看一下，因我们传入的字符串是$template变量，所以重点关注对$template的处理，跟到\_getTemplateId函数，进入

这里根据我们传入的字符串，拼接上模版目录生成了一个字符串作为tempateId

这里实例化了一个对象，对应的对象为： `public $template_class = 'Smarty_Internal_Template';`

在构造函数中设置了很多属性，重点关注$this->template\_resource,$this->source，调用了Smarty\_Template\_Source的load方法

来到了重点：load方法

这个正则，其实匹配了我们对display的输入，将输入的字符串根据`:`分割，第一部分为协议名，第二部分为协议的内容

我们进入Smarty\_Template\_Source中

在smarty中，不同的协议有不同的handler来处理，这里通过Smarty\_Resource::load来获取对应的handler

可以看到，在这里进行了很多次判断，是否是缓存，是否是注册以后的模版等等的判断，可以看到红框框出来的地方进行了对流的判断

在stream\_get\_wrappers的地方，获取了smarty支持的所有流的类型：

可以看到在smarty文档中也提到了，smarty支持流的方式去获取模版

这里只要我们的流在这个名单中，即可返回$handler(Smarty\_Internal\_Resource\_Stream)，也就是只要走到这一步，就会直接调用Smarty\_Internal\_Resource\_Stream类的populate方法：

第一步将我们输入的协议统一转换为：data://这样子，再调用getContent函数

通过fopen获取到模版字符串

##### phar协议

smarty对协议的处理上面也分析过了，主要就是通过协议的不同，获取不同的类进行处理，不同协议的实现差异其实就是对应的handler不同

所以我们只要关注handler的获取就可以了，phar可以触发反序列化这个漏洞应该大家早就不陌生了，那么diaplay的参数可控，真的就可以触发反序列化吗？

payload: `$this->ci_smarty->display('phar:///etc/passwd');`

可以看到，获取的还是Smarty\_Internal\_Resource\_Stream，和data协议一样，同样会走到getContent

但是这里有个问题，想要使用fopen触发phar反序列化，对应的php.ini的phar.readonly的值必须要为false，而默认是true，所以如果在默认环境下，phar是无法触发反序列化的。

###### 奇怪的php协议

按照理论来说，php协议应该会被Smarty\_Internal\_Resource\_Stream所处理，但是如果你跟了php协议的handler的话，你会发现好像并不是这样

`$this->ci_smarty->display('php:phar:///xxx/xxx.phar');`

这里我们刚开始没有关注，但是如果你跟了php协议的处理的话，你会发现居然在这里sysplugins里面有php对应的处理

所以在这里直接返回了smarty\_internal\_resource\_php.php这个php文件中定义的类，也就是Smarty\_Internal\_Resource\_Php这个类

可以看到不仅仅支持php，其他类没有详细看，有时间可以分析一下其他类是干什么的。

所以接下来会调用Smarty\_Internal\_Resource\_Php这个类的populate方法，结果发现这个类并没有这个方法，所以去父类去找，用ctrl+h可以很方便的看出来一个类的继承关系

所以去Smarty\_Internal\_Resource\_File这个类里面去找：

这里第一步调用了buildFilePath函数，进入这个函数，可以看到有很多is\_file的判断，而is\_file也是phar反序列化触发的入口之一，而且不需要phar.readonly的限制，所以考虑是不是可以通过这种方式触发反序列化

最后发现在170行，is\_file函数参数完全可控，触发反序列化：

这个时候，我们就可以通过： `data:,` 来触发phar反序列化

现在，我们可以触发反序列化了，接下来，我们需要找一个pop链，来达到RCE的效果

###### 很沙雕的pop链

CI 这个框架的 POP 链不太直接，一个重要原因是 CI 的类不是自动加载，而是按需加载；要加载的类需要在 config 文件里添加。因此即使全局搜索到很多 `__destruct` 方法，也不代表这些类在实际反序列化链中可用。

找了半天，总算找了个文件包含，但是限制了文件名要满足一定格式，而且要知道mysql的用户名和密码，就很憨憨。。。。

这也就是为什么我没有限制文件的后缀名

先全局搜索 **destruct方法，发现在Cache\_redis中有一个** destruct方法，调用了任意一个对象的close方法

然后全局搜索close方法，在CI\_Session\_database\_driver中调用了本类中的\_release\_lock方法

在这里又调用了db属性的query方法，所以我们可以通过这个地方调用任意的query方法

发现query这个方法是DB\_driver实现的，在里面有一个load\_rdriver函数

看到里面的$this->dbdriver可控，所以说我们可以通过目录穿越

，来包含到任意目录下的xxx\_result.php文件，从而达到RCE的目的

但是想要进入到load\_rdriver函数，要保证前面的所有函数都正常执行，而在前面有一处sql语句执行，如果执行不成功的话就无法到load\_rdriver的地方，所以我们需要正确的mysql配置来绕过

这样就可以利用这个文件包含达到RCE的效果。

###### 写exp

因为很多属性都不是public的，而且我们要保证mysql的连接对象没有任何问题，所以我们可以通过向pop链中的类添加一些公共的set方法来覆盖其中的属性，和java bean一样

```php
// Cache_redis
public function set($param){
        $this->_redis = $param;
    }

// Session_database_driver
public function set($param1){
        $this->_lock = TRUE;
        $this->_platform = "mysql";
        $this->_db = $param1;
    }

// mysqli_driver
public function set(){
        $this->dbdriver = "../../../../../../../tmp/a";
    }

// 控制器
public function payload(){
        $obj1 = $this->cache->redis;
        $obj2 = $this->session;
        $obj3 = $this->db;
        $obj2->set($obj3);
        $obj1->set($obj2);
        echo urlencode(serialize($obj1));
    }
```

这里因为query是在DB\_driver中的，所以，db对象应该是CI\_DB\_mysqli\_driver

生成phar文件：

```php
public function payload(){
        $obj1 = $this->cache->redis;
        $obj2 = $this->session;
        $obj3 = $this->db;
        $obj2->set($obj3);
        $obj1->set($obj2);
        $phar = new Phar("phar.phar"); //后缀名必须为phar
        $phar->startBuffering();
        $phar->setStub("<?php __HALT_COMPILER(); ?>"); //设置stub
        $phar->setMetadata($obj1); //将自定义的meta-data存入manifest
        $phar->addFromString("test.txt", "test"); //添加要压缩的文件
        //签名自动计算
        $phar->stopBuffering();
    }
```

上传 phar 文件到 tmp 下面，之后通过前面可控的模板资源参数触发 `php:phar://...` 路径即可进入反序列化链。原始官方 WP 此处没有贴出最终 URL，复现时需要把上传后的 phar 绝对路径替换进触发点。

## 方法总结

- 核心技巧：利用二次 SQL 注入控制 Smarty 模板资源，再从 SmartyBC 的协议 handler 差异转向 `php:phar://` 触发 phar 反序列化。
- 识别信号：看到模板内容来自数据库字段、模板引擎支持 stream wrapper、且禁用 PHP 执行不完整时，应同时检查 SSTI 直接执行和协议处理链。
- 复用要点：`phar.readonly` 会影响部分 phar 触发面，但 `is_file`、`file_exists` 等文件检查仍可能触发元数据反序列化；POP 链需要确认相关类在框架中实际加载，不能只凭源码中存在 `__destruct` 判断可用。
