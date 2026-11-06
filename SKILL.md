---
name: site-deploy
description: 把一個資料夾裡的靜態網站部署上線,最後一定要 curl 到 200 才算完成。流程:做網站 → 本機 127.0.0.1 預覽給使用者看 → 問要上架到 GitHub Pages(原型 / demo / 說明頁)還是 Cloudflare Pages(正式站 / 自己的網域)→ 部署。用在使用者說「幫我把這個資料夾部署上線」「幫我做個網站並上線」「換成正式站」「綁我的網域」的時候。
license: MIT
---

# site-deploy:一句話把網站部署上線

使用者只會講一句話。你要把它變成一個**真的打得開的網址**,過程中盡量不叫使用者打指令。

**流程固定三步:需求 → 本機預覽 → 上線。** 先讓使用者在自己電腦上看到網站、說好,再問要放哪個平台,最後才部署。
遇到「你幫不了的三步」(帳號 / 金鑰 / 帳單)就停下來,告訴他**去哪個頁面、按什麼**,等他說好了再繼續。
指令裡的 `OWNER` `REPO` `NAME` `DOMAIN` 是佔位字,換成實際的值。每條事實的來源在文末 `[S#]`。

## 紅線

- token / 密碼不進對話、不印出來、不寫進 repo。不要 `cat` 金鑰檔、不要 `echo $CLOUDFLARE_API_TOKEN`、不要 `env` 整包印出來。
- 開公開 repo 之前先跟使用者說:「這個 repo 和網站都會公開,裡面不要放密碼或私人資料。」等他同意。
- 不刪、不覆蓋使用者既有的 repo / Pages 設定 / Cloudflare 專案 / DNS 紀錄;要覆蓋先問。
- 沒 curl 到 `200` 不算完成,不要回報成功。
- 不替使用者升級付費方案、不替他買網域。
- 每個指令都要短(終端機指令有逾時,常見上限 2 分鐘):要等的事一律「查一次 → 不是就再查一次」,不要寫一個跑 10 分鐘的迴圈。

## 0. 先看狀況(不要先動手)

1. **這個資料夾能不能整個公開?** 目前資料夾是 `$HOME`、桌面、下載、文件,或檔案多到不像一個網站(例如超過 500 個)→ **停**,請使用者開一個新的空資料夾再來。
2. 資料夾裡有沒有網站(`index.html`)?沒有就問一句要做什麼內容,做成純 HTML / CSS 一頁式(除非使用者要框架)。
   - 連結和圖片一律用**相對路徑**(`style.css`,不要 `/style.css`)。GitHub Pages 專案站的網址是 `https://OWNER.github.io/REPO/` [S4],所以 `/style.css` 會指到 `OWNER.github.io/style.css`,不在這個網站裡(這是推論,不是官方原文)。
   - `find . -type l` 有 symlink → **停**,問使用者:打包時 symlink 會被展開成實際檔案,指到資料夾外的話,外面的檔案也會被公開 [S8]。
3. 工具在不在:`git --version`、`gh --version`、`gh auth status`。沒有 `gh` → 請使用者照 https://cli.github.com 安裝;沒登入 → 見「你幫不了的三步」。
4. git 身分:commit 上的 email 會跟著公開 repo 一起公開。
   - `git config user.email` 有值、而且不是 `@users.noreply.github.com` → **commit 之前**跟使用者說「commit 會帶著 <那個 email> 公開,要改用 GitHub 的 noreply 地址嗎?」(建議改),等他回答。
   - 是空的、或使用者說要改 → **不要用 `--global`、不要用使用者的私人 email**,只替這個 repo 設 noreply 地址(要先 `git init`):
   ```bash
   LOGIN=$(gh api user -q .login); ID=$(gh api user -q .id)
   git config user.name "$LOGIN"; git config user.email "$ID+$LOGIN@users.noreply.github.com"
   ```
5. 這裡是不是已經是 git repo、已經有 remote(`git remote -v`)?有的話沿用,不要重建,先跟使用者確認;有舊歷史要告訴他「整段歷史都會一起公開」。
6. **先做網站、本機預覽**(還沒問平台之前就做):網站做好後在背景起一個只給本機看的伺服器,請使用者打開看:
   ```bash
   python3 -m http.server 8000 --bind 127.0.0.1    # Windows 用 python;8000 被占用就換 8001…
   ```
   跟使用者說:「網站做好了,先在你電腦上看:http://127.0.0.1:8000 。要改哪裡跟我說,沒問題的話我們再上線。」
   使用者要改 → 改完請他重新整理再看;說好了才往下。上線前把這個伺服器關掉。
7. **問要上架到哪個平台**(使用者已經講過就照他說的,不要再問):
   > 要放到哪裡?
   > ① **GitHub Pages**:做 demo、原型、專案說明頁、活動頁、作品集草稿。免費,網址是 `你的帳號.github.io/網站名`,不用買網域。
   > ② **Cloudflare Pages**:正式網站、要用自己的網域(例如 `yourname.com`)。網站本身免費,網域要自己買。
   選 ① → 第 1 條路;選 ② → 先走第 1 條路建好 repo(之後改版要用),再走第 2 條路。
8. **名字**:repo / 專案名只能用英文小寫、數字、`-`。資料夾名是中文或有空格(例如「我的網站」)→ 轉成小寫英數加 `-`;轉完是空的就問使用者要一個英文名字。

## 1. GitHub Pages(demo / 原型 / 說明頁)

**事先講清楚**:免費帳號只能用**公開 repo** 開 Pages;就算付費用私有 repo,網站本身還是公開的 [S1][S3]。

順序很重要:**先開 Pages,再 push workflow**。workflow 比 Pages 先上去,第一次 run 會在 configure-pages 失敗(`enablement` 預設 false,錯誤訊息「Get Pages site failed」)[S22]。

```bash
# 1. 本機 repo(新版 git 不一定預設 main,要明寫)
git init -b main            # 已經是 repo 就跳過;分支不是 main 先問使用者再 git branch -m main
printf '.env*\n*.pem\n*.key\nid_*\nnode_modules/\n.DS_Store\n' >> .gitignore
git add -A --dry-run        # 🔴 把這份清單給使用者看,他說 OK 才往下
git add -A && git commit -m "first site"

# 2. 建 GitHub repo(先不要 --push)。先確認名字沒被用過
OWNER=$(gh api user -q .login)
gh repo view "$OWNER/NAME" >/dev/null 2>&1 && echo "已經有同名 repo → 停,問使用者"
gh repo create NAME --public --source . || exit 1     # 失敗就停,後面不要跑
REPO=$(gh repo view --json name -q .name)             # 讀回真正的名字,不要沿用變數

# 3. 開 Pages,來源設成 GitHub Actions [S5]
gh api repos/$OWNER/$REPO/pages        # 404 = 還沒開;200 = 已經有了
gh api -X POST repos/$OWNER/$REPO/pages -f build_type=workflow    # 404 時用;成功回 201
# 已經有 Pages(POST 回 409)→ 先問使用者能不能改成 Actions 部署,同意才:
# gh api -X PUT repos/$OWNER/$REPO/pages -f build_type=workflow   # 成功回 204
```

4. 寫 `.github/workflows/pages.yml`(全文如下,版本是 2026-11-05 查到的最新主版本 [S6][S7]):

```yaml
name: Deploy static content to Pages
on:
  push:
    branches: [main]
  workflow_dispatch:
permissions:
  contents: read
  pages: write
  id-token: write
concurrency:
  group: "pages"
  cancel-in-progress: false
jobs:
  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v7
      - name: Setup Pages
        uses: actions/configure-pages@v6
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v5
        with:
          path: '.'
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v5
```

   - 不要照抄官方 starter 的舊版本號,也不要留 `[$default-branch]` 佔位字 [S6]。
   - `path: '.'` 會打包整個資料夾,但 dotfile 預設不打包,`.git` `.github` 一律排除;網站真的需要 dotfile 才在 `with:` 加 `include-hidden-files: true` [S8]。
   - 網站在子資料夾(例如 `dist/`)就把 `path` 改成那個資料夾。

```bash
# 5. push
git add .github/workflows/pages.yml && git commit -m "deploy to GitHub Pages"
git push -u origin main
# 被拒、訊息提到 workflow scope → gh 的 token 少了 workflow 權限 [S10],
# 請使用者在自己的終端機跑 gh auth refresh -s workflow(要開瀏覽器授權),好了再 push

# 6. 等部署跑完。用這次 commit 鎖定 run(不然會拿到上一次的舊 run)[S9]
RUN=$(gh run list --workflow pages.yml --commit "$(git rev-parse HEAD)" --limit 1 --json databaseId -q '.[0].databaseId')
# 空的 = run 還沒建出來 → 等 10 秒再跑這一行
gh run view "$RUN" --json status,conclusion -q '.status + " " + .conclusion'
# 還沒 completed → 等 20 秒再查一次(每次只查一次,不要寫長迴圈)
# completed failure → gh run view "$RUN" --log-failed 讀錯誤、修、push 再跑一次
# 只是時序問題(Pages 晚開)→ gh run rerun "$RUN"
```

7. **驗證**:網址跟 GitHub 要,不要自己拼(repo 叫 `OWNER.github.io` 時網址沒有 `/REPO/`)[S5]。run 綠了也可能要等一下,官方說最多 10 分鐘 [S3]:

```bash
URL=$(gh api repos/$OWNER/$REPO/pages -q .html_url)
for i in 1 2 3; do code=$(curl -s -o /dev/null -w '%{http_code}' "$URL"); echo "$code"; [ "$code" = 200 ] && break; sleep 30; done
```

   一次最多 90 秒。不是 200 就再跑一次這兩行;累計 10 分鐘還不是 200 → 讀錯誤、修、再跑,不要回報成功。是 200 → 把網址貼給使用者。

## 2. Cloudflare Pages(正式站 / 自己的網域)

**為什麼是 Pages 不是 Workers(照實講)**:Cloudflare 的 Pages 文件首頁現在建議「新專案用 Workers」,說 Workers 是主力平台;但 Pages 沒有被棄用,指令、API、免費額度頁都還在 [S11]。
純靜態網站兩邊用起來差不多。這個 skill 選 Pages,因為不用寫 Worker 設定檔、指令單純,而且網域的 DNS 不在 Cloudflare 時,子網域也能用 CNAME 接上(Workers 自訂網域只收 DNS 在 Cloudflare 的網域)[S12][S16]。
使用者若問起,就這樣講。他明確要 Workers 的話,告訴他這個 skill 不做,請照官方遷移指南 [S12]。

🔴 **不要走錯路**:這條路只用 `wrangler pages ...`。不要跑 `wrangler deploy`(沒有 `pages` 那個是 Workers)、不要建 `wrangler.toml` / `wrangler.jsonc`。

1. **事先講清楚**:用 wrangler 上傳的 Pages 專案(Direct Upload)**以後不能改接 Git**,production branch 也不能在後台改,是單向的選擇 [S13]。
2. 確認工具:`node --version`(wrangler 用 `npx wrangler@4` 跑,不用全域安裝)。
3. 請使用者照「你幫不了的三步」做好 Cloudflare 帳號、API token、Account ID。金鑰檔**由你建空殼、由他填值**(變數名是官方的 [S14]):
   ```bash
   mkdir -p ~/.config/site-deploy
   [ -f ~/.config/site-deploy/cloudflare.env ] || printf 'CLOUDFLARE_API_TOKEN=\nCLOUDFLARE_ACCOUNT_ID=\n' > ~/.config/site-deploy/cloudflare.env
   chmod 600 ~/.config/site-deploy/cloudflare.env
   # 用圖形編輯器打開給他填:macOS `open -e`、Windows `notepad`、Linux `xdg-open`
   ```
   跟他說:「把 token 和 Account ID 貼在 `=` 後面,存檔,**不要貼到對話裡**。好了跟我說一聲。」
   你只檢查檔案在不在、權限對不對:`ls -l ~/.config/site-deploy/cloudflare.env`。不要打開、不要 `cat`。
4. **只上傳網站檔案**。🔴 Cloudflare 會上傳資料夾裡除了 `.git` / `node_modules` 以外的**所有**檔案,包括 `.env` 這種 dotfile(跟第 1 條路不一樣)[S15]。所以先把 git 裡的網站檔複製到 `_site/`,列給使用者看(要先走過第 1 條路,資料夾已經是 git repo):
   ```bash
   rm -rf _site && mkdir _site
   git ls-files -z | grep -zv '^\.' | xargs -0 cp --parents -t _site
   find _site -type f                     # 🔴 給使用者看清單
   find _site \( -name '.env*' -o -name '*.pem' -o -name '*.key' -o -name 'id_*' \) | grep -q . && echo "有私密檔 → 停"
   grep -qx '_site/' .gitignore || echo '_site/' >> .gitignore
   ```
5. 建專案、部署(專案名用小寫英數和連字號;一定要先建專案,`deploy` 遇到不存在的專案會跳互動問題卡住 [S13]):

```bash
set -a; . ~/.config/site-deploy/cloudflare.env; set +a
npx wrangler@4 pages project list | grep -qw NAME && echo "已經有同名專案 → 停,問使用者"
npx wrangler@4 pages project create NAME --production-branch main \
  && npx wrangler@4 pages deploy _site --project-name NAME --branch main      # [S15]
```

   - `--branch` 一定要寫,而且要等於建專案時的 production branch。在 git repo 裡 wrangler 會自己讀目前分支,不一致(例如 `master`)就會變成預覽版 `BRANCH.NAME.pages.dev`,正式網址沒更新 [S13]。
   - 部署完正式網址是 `https://NAME.pages.dev`,用第 1 條路第 7 步的 curl 兩行驗到 200。
6. **綁網域**(使用者有網域才做)。請使用者打開 https://dash.cloudflare.com/?to=/:account/workers-and-pages → **選這個專案 → Custom domains → Set up a domain → 輸入網域 → Continue** [S16]。
   - 網域在同一個 Cloudflare 帳號(例如在 Cloudflare 買的)→ 確認後 CNAME 會自動加好 [S16][S18]。
   - 根網域(`example.com`)一定要是同帳號的 Cloudflare zone;子網域的 DNS 在別家 → 在那家加 CNAME 指向 `NAME.pages.dev` [S16]。
   - **一定先在後台綁,再加 DNS**。沒先綁就手動加 CNAME 會 522 [S16]。不要自己先建 DNS 紀錄。
   - 也可以用 API 加:`POST https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/pages/projects/NAME/domains`,body `{"name":"DOMAIN"}`,header 用 `Authorization: Bearer $CLOUDFLARE_API_TOKEN`(不要把 header 印出來)[S17]。API 失敗或 DNS 沒接上,就改請使用者用上面的後台流程。
7. **驗證**:`https://DOMAIN/` 用同樣的 curl 兩行驗到 200。新網域要等憑證發下來,第一次不是 200 很正常,重試幾分鐘再判斷。

### (選用)push 就自動部署

使用者想要「改完 push 就自動更新」才做。官方範例用 `cloudflare/wrangler-action@v4`(不要釘 v4.1.0 / v4.1.1,官方標記壞掉)[S14][S19];checkout 官方範例寫 v6,這裡跟 Pages workflow 一致用 v7。刻意省略官方範例的 `gitHubToken` / `deployments: write`(只是在 GitHub 顯示部署狀態,不影響上線)。
請使用者**自己**到 `https://github.com/OWNER/REPO/settings/secrets/actions` 按 **New repository secret**,加 `CLOUDFLARE_API_TOKEN` 和 `CLOUDFLARE_ACCOUNT_ID`(值不要貼進對話)。然後寫 `.github/workflows/cloudflare.yml`:

```yaml
name: Deploy to Cloudflare Pages
on:
  push:
    branches: [main]
jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@v7
      - name: Collect site files (no dotfiles)
        run: mkdir _site && git ls-files -z | grep -zv '^\.' | xargs -0 cp --parents -t _site
      - uses: cloudflare/wrangler-action@v4
        with:
          apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          accountId: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
          command: pages deploy _site --project-name=NAME --branch=main
```

同一個 repo 同時有 `pages.yml` 時,先問使用者要不要停掉 GitHub Pages 那條(不要自己刪)。

## 之後要更新

使用者說「把 XX 改成 YY,更新上線」:改檔 → 本機預覽給他看(同第 0 步第 6 點)→ 說好 → commit + push。
GitHub Pages 和設了自動部署的 Cloudflare Pages,push 就會自己上線;沒設自動部署的 Cloudflare Pages,再跑一次第 2 條路的 `pages deploy`。最後一樣 curl 到 200 才算完成。

## 你幫不了的三步(停下來,講清楚請使用者自己按)

1. **帳號**(頁面有真人驗證,你代替不了):
   - GitHub 註冊:https://github.com/signup 。登入 gh:請他在自己的終端機跑 `gh auth login`,選 GitHub.com → HTTPS → 問「Authenticate Git with your GitHub credentials?」答 **Y** → Login with a web browser。之後 push 還要帳密 → 請他跑 `gh auth setup-git`。
   - Cloudflare 註冊:https://dash.cloudflare.com/sign-up 。
   - 買網域(要付錢):https://dash.cloudflare.com/?to=/:account/registrar/register (Cloudflare Registrar)。Cloudflare 是成本價(註冊局 + ICANN 的費用,不加價),**不是免費** [S18][S23]。
2. **金鑰**(權限給到剛好):
   - Cloudflare API token:打開 https://dash.cloudflare.com/?to=/:account/api-tokens (Account API Tokens 頁)→ **Create Token → Custom Token「Get started」→ 取名 → Permissions 選 Account / Cloudflare Pages / Edit → Continue to summary → Create Token** [S14]。只給這一項,不要給整個帳號。token 只顯示一次,請他直接存進金鑰檔。
   - Account ID:打開 https://dash.cloudflare.com/?to=/:account/workers-and-pages ,在 **Account Details** 區塊按 **Account ID** 旁的複製鈕(或在任何頁面按 Ctrl/Cmd+K,搜尋 `Copy account ID`)[S24]。不需要先有網域。
3. **帳單**(門檻請他自己看,不要替他升級):
   - GitHub 方案:https://docs.github.com/en/get-started/learning-about-github/githubs-plans 。Pages 限制:網站最多 1GB、每月頻寬 100GB(soft limit)[S20]。
   - Cloudflare Pages 免費方案:每月 500 次 build、每站 20,000 個檔案、單檔 25 MiB [S21]。

## 來源(2026-11-05 查證)

- [S1] https://docs.github.com/en/get-started/learning-about-github/githubs-plans
- [S2] https://docs.github.com/en/pages/getting-started-with-github-pages/using-custom-workflows-with-github-pages
- [S3] https://docs.github.com/en/pages/getting-started-with-github-pages/creating-a-github-pages-site
- [S4] https://docs.github.com/en/pages/getting-started-with-github-pages/what-is-github-pages
- [S5] https://docs.github.com/en/rest/pages/pages
- [S6] https://github.com/actions/starter-workflows/blob/main/pages/static.yml
- [S7] https://github.com/actions/deploy-pages/releases (checkout v7、configure-pages v6、upload-pages-artifact v5、deploy-pages v5)
- [S8] https://github.com/actions/upload-pages-artifact/blob/main/action.yml
- [S9] https://cli.github.com/manual/gh_run_watch 、https://github.com/cli/cli/blob/trunk/pkg/cmd/run/watch/watch.go
- [S10] https://github.com/cli/cli/blob/trunk/pkg/cmd/auth/shared/git_credential.go
- [S11] https://developers.cloudflare.com/pages/
- [S12] https://developers.cloudflare.com/workers/static-assets/migration-guides/migrate-from-pages/
- [S13] https://developers.cloudflare.com/pages/get-started/direct-upload/
- [S14] https://developers.cloudflare.com/pages/how-to/use-direct-upload-with-continuous-integration/
- [S15] https://developers.cloudflare.com/workers/wrangler/commands/pages/
- [S16] https://developers.cloudflare.com/pages/configuration/custom-domains/
- [S17] https://developers.cloudflare.com/api/resources/pages/subresources/projects/subresources/domains/methods/create/
- [S18] https://developers.cloudflare.com/registrar/ 、https://developers.cloudflare.com/registrar/faq/
- [S19] https://github.com/cloudflare/wrangler-action/releases
- [S20] https://docs.github.com/en/pages/getting-started-with-github-pages/github-pages-limits
- [S21] https://developers.cloudflare.com/pages/platform/limits/
- [S22] https://github.com/actions/configure-pages/blob/main/action.yml
- [S23] https://developers.cloudflare.com/registrar/get-started/register-domain/
- [S24] https://developers.cloudflare.com/fundamentals/account/find-account-and-zone-ids/
