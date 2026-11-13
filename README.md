## 🚀 Bot-hosting 自动续期（GitHub Actions）

这是一个基于 GitHub Actions 的自动化脚本，用于定时（可由 Cloudflare Worker 触发）登录自动续期 [Bot-hosting](https://bot-hosting.net) 服务。

⚠️ 有cf盾,太垃圾的机房节点可能过不了，建议用稍微干净点的节点,[B2proxy住宅代理](https://www.b2proxy.com/signup?code=0F5133)

━━━━━━━━━━━━━━━━━━━━━━

### 之前用Discord Token登录的方式仍然可用，Discord token有时会变化，用账号密码更稳定。

━━━━━━━━━━━━━━━━━━━━━━

### 🔐 Secrets 配置说明

| Secret 名称         | 是否必填 | 说明                                              |
|---------------------|----------|---------------------------------------------------|
| EMAIL              | ✅ 必填  | 登录和通知使用的Email,必须填写                          |
| PASSWORD           | ✅ 必填  | 登录密码,必填,在控制台右上角头像里去设置                 |
| NODE_LINK          | ❌ 可选  | 代理链接（如 vless:// vmess:// trojan:// hysteria2:// tuic:// anytls:// socks5:// )|
| TG_BOT_TOKEN       | ❌ 可选  | Telegram Bot Token（用于发送通知）                      |
| TG_CHAT_ID         | ❌ 可选  | Telegram Chat ID（接收通知的用户或群组 ID）              |

> 💡 **多账号场景**：登录凭据改用 `ACC1_EMAIL` / `ACC1_PASSWORD` 这类带分支前缀的命名（分支 `acc1` 对应 `ACC1_*`），详见下方 [多账号方案](#-多账号方案5-个分支--cloudflare-worker-定时触发)。

━━━━━━━━━━━━━━━━━━━━━━

## 部署步骤
1：fork 本项目，在actions菜单允许工作流

2：在`setting`➡`secrets and variables`➡`Actions` 里添加上方必填的secrets

3：去actions菜单手动试运行工作流,确认续期成功后,再按下方 [多账号方案](#-多账号方案5-个分支--cloudflare-worker-定时触发) 配置 Cloudflare Worker 定时触发


## 🧩 多账号方案（5 个分支 + Cloudflare Worker 定时触发）

支持 5 个账号，每个账号对应一个分支（`acc1` ~ `acc5`），脚本会根据分支名自动读取对应前缀的 Secret，互不干扰。定时触发由 Cloudflare Worker 完成，不依赖 GitHub 自带定时器。

### 1. 创建 5 个分支

分支上不需要做任何文件改动，直接从最新 `main` 拉出来即可：

```bash
git checkout main && git pull
git push origin main
git branch acc1
git branch acc2
git branch acc3
git branch acc4
git branch acc5
git push origin acc1 acc2 acc3 acc4 acc5
```

> 也可以直接在 GitHub 网页端操作：仓库页面 ➡ 分支下拉框输入 `acc1` 回车，依次创建 5 个分支。

### 2. Secrets 配置

每个账号的登录凭据按分支前缀命名（分支 `acc1` 对应 `ACC1_*`，依此类推）：

| Secret 名称 | 是否必填 | 说明 |
|---|---|---|
| ACC1_EMAIL ~ ACC5_EMAIL | ✅ 必填 | 对应账号的登录邮箱 |
| ACC1_PASSWORD ~ ACC5_PASSWORD | ✅ 必填 | 对应账号的登录密码 |
| ACC1_NODE_LINK ~ ACC5_NODE_LINK | ❌ 可选 | 对应账号的代理链接（vless:// vmess:// trojan:// 等） |

共享 Secret（所有账号共用一份）：

| Secret 名称 | 说明 |
|---|---|
| TG_BOT_TOKEN / TG_CHAT_ID | TG 通知（所有账号共用） |

> Secret 名不区分大小写，统一用大写即可。
> 💡 `main` 分支兼容单账号模式：自动回退读取不带前缀的 `EMAIL` / `PASSWORD` / `NODE_LINK`（与单账号配置一致），多账号只走 `acc1`~`acc5` 分支。

### 3. Cloudflare Worker 定时触发

Worker 每天定时向 GitHub Actions API 发起 5 次手动触发，每次指定一个分支：

```js
// wrangler.jsonc 中的 cron 触发器（按自己服务的到期时间调整）
// { "crons": [ "0 3 * * *" ] }

const REPO = "你的用户名/Renew-Bothosting";
const BRANCHES = ["acc1", "acc2", "acc3", "acc4", "acc5"];

export default {
  async scheduled(event, env, ctx) {
    const token = env.GITHUB_TOKEN; // Worker 的 Secret 中存放的触发 token
    for (const ref of BRANCHES) {
      const res = await fetch(
        `https://api.github.com/repos/${REPO}/actions/workflows/renew.yml/dispatches`,
        {
          method: "POST",
          headers: {
            "Authorization": `Bearer ${token}`,
            "Accept": "application/vnd.github+json",
            "X-GitHub-Api-Version": "2022-11-28",
          },
          body: JSON.stringify({ ref }),
        }
      );
      console.log(ref, res.status); // 成功返回 204
    }
  },
};
```

> 触发用的 token 需要 GitHub(classic) `repo` + `workflow` 权限（或 fine-grained token 勾选该仓库 Actions 读写），放在 Worker 的 Secret 里，不要明文写死。

## 注意事项
* 必填变量必须要填写
* NODE_LINK支持的代理协议有：vmess,vless,hysteria2,tuic,anytls,socks5等
* 自动续期不代表可以无底线的薅羊毛,不建议多账号
* cron运行时间不一定准确,得根据实际到期时间修改,可在设置里暂停actions功能再开启

## ⚠️ 免责声明
* 本程序仅供学习了解, 非盈利目的，如转载须注明来源。
* 使用本程序必循遵守部署服务器所在地、所在国家和用户所在国家的法律法规, 程序作者不对使用者任何不当行为负责。
