# site-deploy

跟你的 AI agent 講一句話,它就幫你把一個資料夾裡的網站部署上線,給你一個真的打得開的網址。

- 流程:**需求 → 本機預覽(127.0.0.1)→ 上線**。先在你電腦上看到網站、你說好,才上線。
- 上線前會問你放哪:**GitHub Pages**(demo、原型、說明頁,免費)或 **Cloudflare Pages**(正式站、自己的網域)。
- 之後說「把 XX 改成 YY,更新上線」,一句話更新。
- 要註冊帳號、產金鑰、付錢的那幾步,agent 會停下來告訴你去哪個頁面按什麼,不會替你按。

## 安裝

跟你的 agent 說:

> 幫我裝這個 skill:https://github.com/kirinchen/site-deploy-skill

### 手動安裝(或給 agent 看的安裝方式)

**Claude Code**:clone 到個人 skill 資料夾,資料夾名稱要叫 `site-deploy`(跟 skill 名稱一致):

```bash
git clone https://github.com/kirinchen/site-deploy-skill ~/.claude/skills/site-deploy
```

- 裝完通常當下就能用。如果這台電腦原本沒有 `~/.claude/skills/` 這個資料夾,要在 Claude Code 裡跑 `/reload-skills`,或關掉重開。
- Claude Code Desktop 的 Code 分頁(本機)讀的是同一個資料夾;**Cowork 和雲端工作階段不讀**這個資料夾。

**Codex CLI / Gemini CLI**(格式相容,作者沒實測):兩者都讀 `~/.agents/skills/`:

```bash
git clone https://github.com/kirinchen/site-deploy-skill ~/.agents/skills/site-deploy
```

Gemini CLI 也可以用 `gemini skills install https://github.com/kirinchen/site-deploy-skill.git --consent`;已經開著的 session 跑 `/skills reload`。

> 只裝你信得過的來源的 skill。裝之前可以先打開 `SKILL.md` 看它會做什麼。

## 用法

> 幫我把這個資料夾部署上線。

> 換成正式站,綁我的網域 example.com。

## 支援的 agent

| Agent | 狀態 |
|---|---|
| Claude Code(CLI、Desktop 的 Code 分頁) | 作者實測 |
| Codex CLI | 格式相容,作者沒實測 |
| Gemini CLI | 格式相容,作者沒實測 |

## 需要什麼

- `git`、[GitHub CLI(`gh`)](https://cli.github.com) 和一個 GitHub 帳號。
- 走正式站另外要 Node.js 和 Cloudflare 帳號;要綁網域就得有一個網域。

## 參考

- Claude Code skills:https://code.claude.com/docs/en/skills
- Agent Skills 規格:https://agentskills.io/specification
- Codex skills:https://learn.chatgpt.com/docs/build-skills
- Gemini CLI skills:https://github.com/google-gemini/gemini-cli/blob/main/docs/cli/skills.md

## 授權

MIT,見 [LICENSE](LICENSE)。

## 作者

衝浪阿北(YouTube 頻道「熱血吧 衝浪阿北」)
