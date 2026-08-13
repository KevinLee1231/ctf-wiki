# hi doggy

## 题目简述

服务端把用户输入解析为 Pug AST，只保留少数节点类型，并检查普通属性值必须是引号包裹的字符串，试图防止 SSTI。过滤器却只遍历 `attrs`、`nodes` 和 `block`，漏掉 Pug `&attributes(...)` 对应的动态属性块，表达式最终仍会被代码生成器执行。

## 解题过程

普通属性会进入 `node.attrs` 并接受正则检查，而 `&attributes` 的表达式保存在另一 AST 字段。其父节点仍是白名单中的 `Tag`，所以既不会被 `filterNodes` 删除，也不会被 `validateAttrs` 检查。

提交一行 Pug：

```pug
a&attributes({'href': global.process.mainModule.constructor._load("child_process").execSync('/readflag GIVEFLAGPLS').toString()}) a
```

执行过程如下：

1. `global.process.mainModule.constructor._load("child_process")` 取得 Node.js 内置模块；
2. `execSync` 运行 `/readflag GIVEFLAGPLS`；
3. SUID `readflag` 校验固定口令后读取普通 Node 用户无权直接访问的 `/flag`；
4. 命令输出被转成字符串，作为生成的 `href` 属性值随 HTML 响应返回。

渲染结果中即可读到：

```text
grey{I_cAn'T_THInK_0F_AnytHing_clever_T0_pu7_h3r3}
```

## 方法总结

对复杂模板 AST 做自制白名单时，必须理解所有可执行表达式的存储位置；只遍历几个常见子字段会留下旁路。更安全的做法是不把不可信输入当作模板，或使用只接受纯数据的受限渲染路径。若确需 AST 筛选，应采取默认拒绝并递归覆盖完整 schema，而不是遇到白名单父节点就原样保留未知字段。
