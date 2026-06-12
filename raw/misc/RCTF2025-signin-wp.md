# Signin

## 题目简述

签到题的网页前端会把涂色完成度作为 `score` 参数提交到 `/?score=<value>`。直接阅读源码可知分数达到 100 时会返回 flag。

## 解题过程

### 关键观察

签到题的网页前端会把涂色完成度作为 `score` 参数提交到 `/?score=<value>`。

### 求解步骤

阅读题目网页源码：
       function submitResult() {
            const score =
parseFloat(document.getElementById('progressText').textContent);

            // 使用fetch获取结果，不刷新页面
            fetch(`/?score=${score}`)
                .then(response => response.text())
                .then(html => {
                    // 解析返回的HTML，提取结果信息
                    const parser = new DOMParser();
                    const doc = parser.parseFromString(html, 'text/html');

猜测 score=100 时就会给 flag，于是访问 http://<challenge-host>/?score=100 得到 flag
RCTF{W3lc0m3_T0_RCTF_2025!!!}

### PDF 外链

- <http://<challenge-host>/?score=100>

### 跨页补回：前端提交逻辑收尾

// 检查是否成功
                    if (score >= 100) {
                        const flagMatch = html.match(/ROIS\\{[^}]+\\}/);
                        if (flagMatch) {
                            console.log('🎉 恭喜你！完美涂色！🎉');
                            console.log('Flag: ' + flagMatch[0]);
                            alert('🎉 恭喜你！\\n\\n' + flagMatch[0]);
                        }
                    } else {
                        console.log(`涂色完成度: ${score.toFixed(1)}%，继续加油！
`);
                    }

                    // 更新URL但不刷新页面
                    window.history.pushState({score: score}, '', `/?
score=${score}`);
                })
                .catch(error => {
                    console.error('提交失败', error);
                });
        }

## 方法总结

- 核心技巧：前端源码审计。
- 识别信号：客户端直接把完成度作为查询参数提交。
- 复用要点：签到题先读 JS，尝试边界值而不是手动完成交互。
