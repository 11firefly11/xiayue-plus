# xiayue-plus

`xiayue-plus` 是一个 Burp Suite Java 扩展，用于辅助发现越权、未授权、参数置空和越权值绕过相关的访问控制风险。插件会监听 Burp Proxy 和 Repeater 中的 GET/POST 响应，并以原始响应作为基线，自动构造低权限、未授权、参数置空和越权值替换请求，最后在 Burp 内置标签页中展示响应差异。

本项目基于 `xia-yue` 思路进行二开，重点新增了参数置空检测和越权值检测：对参数值置空或替换为 `%`、`null`、`none`、自定义值后，判断返回包是否出现异常差异，并把疑似高风险结果高亮展示。

## 最新更新

- 新增参数置空检测：对每个可测试参数执行置空请求，并判断置空后的返回结果是否异常。
- 新增越权值检测：支持将参数替换为 `%`、`null`、`none` 或自定义值，辅助观察特殊值绕过后的响应差异。
- 新增“显示未授权”模式：当低权限包或未授权包返回长度与原始包一致时，将相关记录优先展示。
- 新增疑似未授权备注：开启“显示未授权”后，会在参数置空结果的 `一致` / `差异` 后追加 `疑似未授权(低权限,未授权)`，多个来源使用英文逗号分隔。
- 优化排序逻辑：点击“参数置空”列时，标红的重点关注结果优先；同时开启“显示未授权”时，疑似未授权记录作为第二优先级。

## 功能特性

- 监听 Burp Proxy 和 Repeater 中的 GET/POST 流量。
- 使用低权限认证头重放原始请求。
- 移除指定认证头后重放原始请求，用于观察未授权访问差异。
- 对 URL、BODY、XML、MULTIPART、JSON 参数执行置空检测，观察置空后的返回值是否异常。
- 支持 `%`、`null`、`none` 和自定义越权值检测，观察参数被替换为越权值后的返回值是否异常。
- 按响应状态码、响应体长度、响应体 MD5 对置空和越权值检测结果分组。
- 当置空或越权值请求的响应体大于原始响应体，或响应指纹与原始包不同，提示重点关注。
- 支持“显示未授权”模式，将低权限包或未授权包返回长度与原始包一致的记录优先展示，并在置空检测的一致/差异结果后追加疑似未授权备注。
- 自动跳过常见静态资源，例如图片、CSS、JavaScript、字体、媒体和 sourcemap。
- 支持 URL 白名单，便于控制测试范围。
- 使用请求方法、内容类型、参数和 body 指纹进行去重，减少重复检测。

## 环境要求

- Burp Suite，需支持加载传统 Java 扩展。
- JDK 8 或更高版本。
- Windows PowerShell，用于执行下方示例命令。

## 安装方法

1. 打开 Burp Suite。
2. 进入 `Extensions` -> `Installed` -> `Add`。
3. `Extension type` 选择 `Java`。
4. 选择 `xiayue-plus-1.0.jar`。
5. 加载成功后，Burp 中会出现 `xiayue-plus` 标签页。

## 使用说明

1. 打开 `xiayue-plus` 标签页。
2. 在低权限配置框中填写低权限认证信息，例如：

   ```http
   Cookie: JSESSIONID=test; UUID=1; userid=guest
   Authorization: Bearer test
   ```

3. 在未授权配置框中填写需要移除的认证头名称，例如：

   ```text
   Cookie
   Authorization
   Token
   ```

4. 按需勾选绕过值检测选项。
5. 如需优先查看疑似未授权结果，勾选“显示未授权”。
6. 启动插件开关。
7. 通过 Proxy 浏览目标，或使用 Repeater 发送请求。
8. 在结果表中查看原始包、低权限包、未授权包和参数置空包的差异。

结果中的“重点关注”不代表一定存在漏洞，但通常值得进一步手工确认。建议结合响应内容、业务语义、权限模型和目标系统状态综合判断。

## 检测逻辑

插件会把原始响应作为基线，对每个可测试参数分别构造变异请求：

- 参数置空检测：移除当前参数值，保留参数名和请求结构，发送置空后的请求。
- 越权值检测：将当前参数值替换为启用的越权值，例如 `%`、`null`、`none` 或自定义值。

每个变异请求返回后，插件会记录响应状态码、响应体长度和响应体 MD5，并与原始响应进行对比：

- `一致`：响应指纹与原始响应一致。
- `差异`：响应状态码、响应体长度或响应体 MD5 与原始响应不同。
- `重点关注`：变异后的响应体大于原始响应体，通常需要优先人工确认。

开启“显示未授权”后，如果低权限包或未授权包返回长度与原始包一致，结果会优先排序，并在 `一致` / `差异` 后追加 `疑似未授权(低权限,未授权)` 备注。多个命中来源用英文逗号分隔。点击“参数置空”列排序时，标红的重点关注结果仍会排在最前。

这种检测方式适合辅助发现因参数为空、参数被特殊值替换、对象 ID 被越权值覆盖而导致的异常回显、权限绕过或数据暴露问题。

## 结果说明

主结果表包含原始包、低权限包、未授权包和参数置空结果：

- `原始包长度`：原始响应 body 长度，作为后续对比基线。
- `低权限包长度`：替换或新增低权限认证信息后的响应 body 长度；出现 `✔` 表示与原始响应长度一致。
- `未授权包长度`：移除指定认证头后的响应 body 长度；出现 `✔` 表示与原始响应长度一致。
- `参数置空`：展示参数置空和越权值检测汇总，包含 `一致`、`差异`、`重点关注` 等状态。

其中 `✔` 不等于漏洞结论，只表示该变体响应长度与原始响应一致；需要结合响应内容和业务语义进一步确认。

## 编译

编译扩展类：

```powershell
New-Item -ItemType Directory -Force -Path .\out\release | Out-Null
javac -encoding UTF-8 -d .\out\release (Get-ChildItem -Path .\sources -Recurse -Filter *.java).FullName
```

打包 jar：

```powershell
jar cfm .\xiayue-plus-1.0.jar .\resources\META-INF\MANIFEST.MF -C .\out\release .
```

## 测试

项目中包含一个用于验证摘要与排序逻辑的 Java harness：

```powershell
New-Item -ItemType Directory -Force -Path .\out\test | Out-Null
javac -encoding UTF-8 -d .\out\test (Get-ChildItem -Path .\sources,.\tests -Recurse -Filter *.java).FullName
java -cp .\out\test burp.BurpExtenderSummaryHarness
```

预期输出：

```text
BurpExtenderSummaryHarness OK
```

## 项目结构

```text
sources/burp/                  Java 源码和 Burp API 兼容接口
tests/burp/                    测试 harness
resources/META-INF/MANIFEST.MF Jar manifest
xiayue-plus-1.0.jar            当前可加载的扩展 jar
```

## 安全声明

请仅在你拥有授权的系统中使用本扩展。插件会主动发送额外 HTTP 请求，并修改认证头和参数值；在敏感环境中使用前，请严格限制 Burp scope，并确认测试行为符合授权范围。

## License

当前项目尚未指定开源许可证。正式公开发布前，建议补充 License，明确他人是否可以使用、修改和分发本项目。
