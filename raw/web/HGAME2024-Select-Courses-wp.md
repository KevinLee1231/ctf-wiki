# Select Courses

## 题目简述

页面包含 5 门课程。后端每隔 30 至 180 秒随机放出一门尚未选中的课程；如果 5 秒内无人选中，课程会再次变为满员。已经成功选中的课程不会重新放出，全部选中后点击“选完了”即可得到 flag。主要障碍是持续监控页面状态并在短时间窗口内自动点击。

## 解题过程

手工等待并点击可以完成，但耗时且容易错过 5 秒窗口。更稳定的做法是让 Selenium 周期性刷新页面，展开每门课程的面板，读取状态列；只要状态不是“已满”，就立即点击选课按钮并确认弹窗。

PDF 中被分页打断的 XPath 已恢复为完整字符串，整理后的脚本如下：

```python
from time import sleep

from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support import expected_conditions as EC
from selenium.webdriver.support.ui import WebDriverWait

TARGET = "http://challenge.host/"

driver = webdriver.Chrome()
driver.get(TARGET)
sleep(3)

courses = []
for index in range(1, 6):
    courses.append(
        {
            "panel": f'//*[@id="selector-container"]/section[{index}]/div[1]',
            "status": (
                f'//*[@id="selector-container"]/section[{index}]'
                "/div[2]/table/tbody/tr/td[5]"
            ),
            "submit": (
                f'//*[@id="selector-container"]/section[{index}]'
                "/div[2]/table/tbody/tr/td[6]/button"
            ),
        }
    )

while courses:
    driver.refresh()
    sleep(2)

    for course in courses:
        driver.find_element(By.XPATH, course["panel"]).click()
        status = driver.find_element(By.XPATH, course["status"]).text

        if status != "已满":
            driver.find_element(By.XPATH, course["submit"]).click()
            WebDriverWait(driver, 5).until(EC.alert_is_present())
            driver.switch_to.alert.accept()
            courses.remove(course)
            break

# 此时在页面点击“选完了”获取 flag，再关闭浏览器。
input("确认已经读取 flag 后按回车关闭浏览器：")
driver.quit()
```

也可以绕过浏览器界面，直接持续调用选课接口。判断标准是响应 JSON 出现：

```json
{"full": 0, "message": "选课成功！"}
```

当 5 门课都返回成功后，再提交最终检查请求。官方题解验证得到 `hgame{w0W_!_1E4Rn_To_u5e_5cripT_^_^}`。

## 方法总结

- 核心技巧：用浏览器自动化或直接 HTTP 轮询抢占短暂开放的课程状态。
- 识别信号：资源在随机时间开放、开放窗口很短、成功状态可通过页面字段或 JSON 响应稳定判断。
- 复用要点：轮询间隔要短于开放窗口，同时避免无意义的高频请求；直接调用接口通常比依赖易变的 XPath 更稳，但需要先从浏览器请求中确认接口和参数。
