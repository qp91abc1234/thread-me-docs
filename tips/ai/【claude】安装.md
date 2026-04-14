## 安装
```sh
# powershell
irm https://claude.ai/install.ps1 | iex
```

设置环境变量
```sh
# User Path
C:\Users\35001\.local\bin
```

添加 api 配置
参考：https://api-docs.deepseek.com/zh-cn/guides/anthropic_api
```json
{
  "autoUpdatesChannel": "latest",
  "env":{
    "ANTHROPIC_BASE_URL": "https://api.deepseek.com/anthropic",
    "API_TIMEOUT_MS": "600000",
    "ANTHROPIC_AUTH_TOKEN": "..............",
    "ANTHROPIC_MODEL": "deepseek-chat",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "deepseek-chat",
    "CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC": "1"
  }
}
```

安装 vscode 插件
```sh
anthropic.claude-code
```

## 删除
清理环境变量
删除以下路径

```sh
C:\Users\35001\.claude
C:\Users\35001\.local
```
