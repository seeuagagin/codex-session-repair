# 诊断与入口修复

## 分清故障层

| 实际证据 | 处理方向 |
| --- | --- |
| codex_app 工具未加载；command/cwd 指向不存在路径 | 检查 MCP 启动入口和当前应用版本 |
| usage_limit_exceeded | 额度阻塞；不据此修改历史或重复发消息 |
| 子代理报告只看到加密占位，主代理调用返回空成功 | 检查正文是否到达、真实回复、运行状态和错误；此现象本身不证明密文损坏 |
| modelProviderMismatch | 核对保存与当前 provider，通过受支持的切回或 fork 继续 |
| App-server task is not ready: model-provider | 通用未就绪状态；须查看更具体错误，不能直接等同 provider mismatch |
| invalid_encrypted_content，指定 item 验证/解密/解析失败 | 定位所属 turn，进入历史修复流程 |
| execution-config-loading 等未就绪 | 查实际加载状态；不当成同一个加密故障 |

优先使用 read_thread、分页历史与紧邻错误日志。需要 SQLite 辅助定位时，仅打开已确认数据库的 URI mode=ro，先看 schema，参数化查询精确 UUID 和必要字段。数据库版本、路径和列名均须现场确认。

## codex_app 启动路径失效

1. 核对当前会话实际加载的 MCP 配置、应用版本、command/cwd 是否存在。只读取必要键，避免输出完整配置或环境。
2. 在已有授权内备份受影响配置，记录 SHA-256、属主、权限及恢复方式。配置可能含凭据，沿本机秘密管理规则保护备份。
3. 优先复用本机已经登记且可执行的稳定启动入口。修正必要 command/cwd，保留现有工具审批设置和其他配置；不把某次缓存版本号作为永久路径写入技能。
4. 若需重启，核对授权与其他活动会话，协调安全停点。确认旧进程退出和新版本入口实际加载，不能只发出重启命令就报告恢复。
5. 按需要验证新建和跨 chat 转发。创建用户可见测试聊天须符合当前工具授权要求；原生测试通过不能代替桌面新建验证。

## provider 不匹配

[官方 thread/resume / fork 文档](https://learn.chatgpt.com/docs/app-server#start-or-resume-a-thread)规定，保存的 provider 与当前配置不匹配时可返回 -32600 / modelProviderMismatch。读取历史不要求先成功恢复运行时。

优先使用受支持的 provider 切回；选择当前 provider 时可 fork 历史。不要直接修改保存记录中的 provider，也不要把仅支持模型/推理强度的技能当成 provider 迁移工具。

记录原 chat → 恢复 chat 映射。在实际使用的 provider 下做两轮续聊验证；切换前通过的结果不能证明切换后可用。fork 不能自动迁移业务执行器登记或旧原生代理树。
