# Claude Code 應設定的 10 大資安設定（摘要）

> 原文：[【2026 年最新版】在 Claude Code 中應設定的 10 大資安設定](https://codelove.tw/@tony/post/3kkez3)
> 內容以 2026 年 3 月的 Claude Code 官方文件為基準。

Claude Code 能執行終端機指令與讀寫檔案，若不調整安全設定，可能帶來意料之外的風險。以下整理十項建議設定。

---

## 啟用沙箱（Sandbox）
這是最重要的設定。
啟用後，Claude Code 執行的 Bash 指令會在作業系統層級被隔離，檔案系統與網路存取都會受限。macOS 使用 Seatbelt、Linux 使用 Bubblewrap，可用 `/sandbox` 指令查看狀態。

```json
// .claude/settings.json
{
  "sandbox": {
    "enabled": true
  }
}
```

---

## 封堵沙箱的逃脫出口
沙箱預設保留「escape hatch」，部分指令可能跑到沙箱外執行。
把 `allowUnsandboxedCommands` 設為 `false`，即可讓 `dangerouslyDisableSandbox` 這類繞過手段完全失效。

```json
{
  "sandbox": {
    "enabled": true,
    "allowUnsandboxedCommands": false
  }
}
```

---

## 以 deny 規則阻擋危險指令
用 `permissions.deny` 明確列出禁止執行的指令。
規則評估順序為 **deny → ask → allow**，deny 優先權最高。curl / wget 雖預設已封鎖，明確寫進 deny 可避免被其他 allow 設定意外覆蓋。

```json
{
  "permissions": {
    "deny": [
      "Bash(rm -rf *)",
      "Bash(curl *)",
      "Bash(wget *)",
      "Bash(git push *)",
      "Bash(chmod 777 *)"
    ]
  }
}
```

---

## 拒絕存取機密檔案
封鎖 `.env`、金鑰、憑證等敏感檔案的讀取。
搭配沙箱的 `denyRead`，可同時擋住 Claude Code 檔案工具與 Bash 指令兩條讀取路徑。

```json
{
  "permissions": {
    "deny": [
      "Read(./.env)",
      "Read(./.env.*)",
      "Read(./secrets/**)",
      "Read(./config/credentials.json)",
      "Read(**/*.pem)",
      "Read(**/*.key)"
    ]
  }
}
```

```json
{
  "sandbox": {
    "filesystem": {
      "denyRead": ["~/.aws/credentials", "~/.ssh"]
    }
  }
}
```

---

## 限制網路存取
透過沙箱的網路控制，以白名單方式只允許特定網域。
可大幅降低資料被傳到外部伺服器的風險，特別能防範提示注入（prompt injection）造成的資料外洩。

```json
{
  "sandbox": {
    "network": {
      "allowedDomains": [
        "github.com",
        "*.githubusercontent.com",
        "*.npmjs.org",
        "registry.yarnpkg.com",
        "pypi.org"
      ]
    }
  }
}
```

---

## 停用 bypassPermissions 模式
`--dangerously-skip-permissions` 會跳過所有權限檢查，團隊開發時建議直接停用。
若寫在 Managed Settings 中，使用者端無法覆寫。

```json
{
  "permissions": {
    "disableBypassPermissionsMode": "disable"
  }
}
```

---

## 用 PreToolUse hook 加入自訂安全檢查
Hooks 可在 Claude Code 生命週期的各階段執行自訂腳本；`PreToolUse` 會在工具執行前觸發，適合做額外檢查。腳本以 `exit 2` 結束即可阻止該工具執行。
Hook 類型共有 4 種：command、HTTP webhook、LLM 提示評估、agent。

```json
// .claude/settings.json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": ".claude/hooks/validate-command.sh"
          }
        ]
      }
    ]
  }
}
```

```bash
#!/bin/bash
# .claude/hooks/validate-command.sh
COMMAND=$(jq -r '.tool_input.command' < /dev/stdin)

# 阻擋 rm -rf
if echo "$COMMAND" | grep -q 'rm -rf'; then
  echo "Blocked: rm -rf commands are not allowed" >&2
  exit 2
fi

# 阻擋連線到正式環境
if echo "$COMMAND" | grep -q 'prod'; then
  echo "Blocked: production access is not allowed" >&2
  exit 2
fi

exit 0
```

---

## 用 /permissions 定期檢視權限
`/permissions` 可列出目前的權限設定，建議定期檢查：
- 會話中是否累積了過多「Always allow」規則
- 是否有不再需要的允許規則
- deny 規則是否如預期生效

另外 `/status` 可查看目前載入了哪些設定檔，也能發現設定檔錯誤。

```text
/permissions
/status
```

---

## ⑨ 用 devcontainer 建立完全隔離環境
最安全的做法是讓 Claude Code 跑在容器裡。Anthropic 官方有提供 devcontainer 參考實作，特點包括：預設 deny 的防火牆網路控制、與主機完全隔離、可用 VS Code Remote Containers 快速建置。
在此環境中，若是可信任的倉庫，自動化流程使用 `--dangerously-skip-permissions` 相對較安全（僅限可信來源）。

```bash
# 方法1：使用官方倉庫的 devcontainer 參考實作
git clone https://github.com/anthropics/claude-code.git
# 在 VS Code 開啟後選擇「Reopen in Container」

# 方法2：用 devcontainer features 加入既有專案
# 參考：https://github.com/anthropics/devcontainer-features
```

---

## ⑩ 團隊：以 Managed Settings 強制組織政策
Managed Settings 優先權最高，使用者與專案層級都無法覆寫，適合集中管理組織政策。有兩種方式：

- **Server-managed settings**（public beta）：從 Claude.ai 管理後台下發，不需 MDM，適合遠端 / BYOD；Teams / Enterprise 方案可用，登入時自動套用。
- **Endpoint-managed settings**：透過 MDM（Jamf、Intune 等）部署到裝置，適合重視安全的組織。檔案位置：
  - macOS：`/Library/Application Support/ClaudeCode/managed-settings.json`
  - Linux：`/etc/claude-code/managed-settings.json`
  - Windows：`C:\Program Files\ClaudeCode\managed-settings.json`

| 設定鍵 | 效果 |
|---|---|
| `disableBypassPermissionsMode` | 禁止使用 `--dangerously-skip-permissions` |
| `allowManagedPermissionRulesOnly` | 只採用管理者的 allow / deny / ask 規則 |
| `allowManagedHooksOnly` | 停用使用者與專案 hooks，只用管理者設定的 |
| `allowManagedMcpServersOnly` | 只允許管理者核可的 MCP 伺服器 |
| `allowManagedDomainsOnly` | 網路只能連到管理者指定的網域 |

```json
{
  "permissions": {
    "disableBypassPermissionsMode": "disable"
  },
  "allowManagedPermissionRulesOnly": true,
  "allowManagedHooksOnly": true,
  "allowManagedMcpServersOnly": true,
  "sandbox": {
    "enabled": true,
    "allowUnsandboxedCommands": false,
    "network": {
      "allowManagedDomainsOnly": true,
      "allowedDomains": ["github.com", "*.npmjs.org"]
    }
  }
}
```

---

## 結語
便利性與安全性需要取捨。建議先完成 ①～④ 的基本設定，再逐步導入 ⑤ 之後的項目。

## 參考連結
- [Security](https://code.claude.com/docs/en/security)
- [Permissions](https://code.claude.com/docs/en/permissions)
- [Sandboxing](https://code.claude.com/docs/en/sandboxing)
- [Hooks Reference](https://code.claude.com/docs/en/hooks)
- [Settings](https://code.claude.com/docs/en/settings)
- [Server-managed Settings](https://code.claude.com/docs/en/server-managed-settings)
