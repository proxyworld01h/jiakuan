https://blog.imxiaohe.com/2026/09/123.html

Cloudflare Worker 脚本代码下载地址： [点此跳转]
前往并打开上方占位符中的 Worker 脚本代码链接，将代码全部复制备用。
登录你的 Cloudflare 控制台。
在左侧导航进入 Workers 和 Pages ➔ 点击 创建应用程序 ➔ 选择 创建 Worker。
为 Worker 命名后点击右下角 部署。
部署完成后点击 编辑代码，清空原有代码，将第 1 步复制的代码粘贴进去，点击右上角 部署 保存。
记下生成的测速域名：在 Worker 详情页记下分配的默认域名（例如 xxxx.xxxx.workers.dev），或者在 设置 ➔ 触发器 中绑定的自定义域名。该域名在后续步骤中将作为 Worker 检测域名 填入代码。
第二步：新建 GitHub 仓库与上传代码
1. 新建 GitHub 仓库
登录 GitHub，点击右上角 + ➔ New repository。
仓库名称自定义（例如 gate）。
仓库类型必须选择 Public（公开）。
勾选 Add a README file，点击 Create repository 创建完成。
2. 源码下载与普通文件上传
请前往以下预设的代码文件下载地址获取基础代码文件：

代码文件 下载地址1： [点此跳转]
代码文件 下载地址2： [点此跳转]
下载完成后，在你的 GitHub 仓库主页点击 Add file ➔ Upload files，将对应的核心运行脚本（如 vpngate.py、requirements.txt）以及 web/ 静态模板文件上传并点击 Commit changes 保存。

3. 手动创建无法直接上传的工作流文件（必做）
⚠️ 注意事项： GitHub 网页端不支持直接拖拽上传带点开头的路径（.github/），因此必须通过网页端新建文件并手动录入路径。
在仓库根目录点击 Add file ➔ Create new file。
在文件名输入框中输入路径及文件名：.gitignore。
在下方代码编辑框中，完整粘贴以下忽略规则代码：
# 由 vpngate.py 运行时生成 (data.json + index.html), 不入库
public/
__pycache__/
*.pyc
粘贴完成后，点击页面最下方的 Commit changes 保存文件。

在仓库根目录再次点击 Add file ➔ Create new file。
在文件名输入框中输入路径及文件名：.github/workflows/check.yml（每输入一个斜杠 / 系统会自动生成目录结构）。
在下方代码编辑框中，完整粘贴仓库原版的工作流配置代码：
name: VPN Gate Node Check

on:
  # 定时检测: 默认每 30 分钟一次。改成每 60 分钟: cron: "0 * * * *"
  schedule:
    - cron: "*/30 * * * *"
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: gate-check
  cancel-in-progress: false

jobs:
  check:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Get VPN Gate nodes + check via Cloudflare Worker + build page
        run: python vpngate.py
        env:
          # 已部署的检测 Worker (不要改动: 检测统一走这里)
          CHECK_WORKER: "https://check.helei.kdns.fr/check?sstp=vpn:vpn@"
          CHECK_CONCURRENCY: "32"
          CHECK_TIMEOUT: "90"

      - name: Ensure GitHub Pages is enabled (one-time auto setup)
        run: |
          code=$(curl -s -o /dev/null -w '%{http_code}' \
            -H "Authorization: Bearer $GITHUB_TOKEN" \
            "https://api.github.com/repos/$GITHUB_REPOSITORY/pages")
          echo "Pages API status: $code"
          if [ "$code" = "404" ]; then
            echo "首次运行: 启用 GitHub Pages (source = GitHub Actions, 目录 /public)"
            curl -sS -X PUT \
              -H "Authorization: Bearer $GITHUB_TOKEN" \
              -H "Content-Type: application/json" \
              -d '{"build_type":"workflow","source":{"branches":["main"],"path":"/public"}}' \
              "https://api.github.com/repos/$GITHUB_REPOSITORY/pages"
            echo
            sleep 10
          fi
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Configure Pages
        uses: actions/configure-pages@v5

      - name: Upload Pages artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: 'public'

      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4

      - name: Show site URL
        run: 'echo "site: https://jerylihub.github.io/gate/ (deploy outcome: ${{ steps.deployment.outcome }})"'
粘贴完成后，点击页面最下方的 Commit changes 保存文件。

三、步骤三：GitHub Actions 权限与 Pages 静态托管设置
1. 开启 Actions 读写权限
点击仓库顶部的 Settings。
在左侧菜单点击 Actions ➔ General。
向下滑动找到 Workflow permissions 区域，将选项勾选为：
👉 Read and write permissions。
点击 Save 保存设置。
2. 设置 GitHub Pages 静态站点
在仓库的 Settings 页面中，左侧点击 Pages。
在 Build and deployment 区域，确保 Source 选为 github actions。
四、步骤四：获取优选域名并修改核心代码配置
1. 获取最新优选域名
打开优选域名提供网址，筛选出测速优异的 Cloudflare 优选 IP 或优选域名备用：

优选域名提供网址： [https://bestcf.fxxk.dedyn.io/]
2. 在 vpngate.py 中修改三处核心配置
打开仓库根目录下的 vpngate.py 文件，点击右上角的 ✏️ 铅笔图标进行在线编辑，必须严格修改以下三处配置：

位置一：替换 Worker 测速检测端（文件第 52 ~ 55 行左右）
找到定义 WORKER_CHECK_URL 的代码行，将默认域名 check.helei.kdns.fr 替换为你第一步部署完成的 Cloudflare Worker 域名（保留前面的 https:// 以及末尾的 /check?sstp=vpn:vpn@）：

# 原代码第 52-55 行左右：
WORKER_CHECK_URL = os.environ.get("CHECK_WORKER", "https://你的Worker域名/check?sstp=vpn:vpn@")
位置二：替换 Cloudflare 优选域名池（文件第 461 ~ 463 行左右）
找到定义 EDGE_HOSTS 的代码段，这里是 edgetunnel 的入口优选地址池。将双引号内由逗号分隔的默认域名（如 saas.072159.xyz:443,...）替换为你在优选网站获取到的最新域名或 IP，每个地址后必须带上 :443 端口：

# 原代码第 276-284 行左右：
EDGE_HOSTS = [
    h.strip()
    for h in os.environ.get(
        "EDGE_HOSTS",
        "填入优选域名1:443,填入优选域名2:443,填入优选域名3:443",
    ).split(",")
    if h.strip()
]
位置三：必须配置用户自己的 edgetunnel 节点信息（文件第 525 ~ 526 行左右，必做项）
特别注意：代码内预留的 EDT_UUID 和 EDT_DOMAIN 是演示参数。你必须替换为自己实际部署的 edgetunnel 节点域名和对应的 UUID 密钥，否则自动生成的 sub.txt 订阅链接将无法连接使用！

edgetunnel部署代码：【点此跳转】

# 原代码第 338-341 行左右：
EDT_UUID = os.environ.get("EDT_UUID", "填入你自己edgetunnel的UUID")
EDT_DOMAIN = os.environ.get("EDT_DOMAIN", "填入你自己edgetunnel绑定的节点域名")
EDT_FINGERPRINT = os.environ.get("EDT_FINGERPRINT", "chrome")
三处关键参数修改核对无误后，滑动到页面最下方点击 Commit changes 保存提交。

五、步骤五：启动构建并获取两大生成页面
1. 手动运行 Actions 任务
点击仓库顶部的 Actions 选项卡。
在左侧 Workflows 列表点击 VPN Gate Node Check。
点击右侧的 Run workflow 按钮，弹出菜单中再次点击绿色的 Run workflow 触发运行。
等待 1-3 分钟，当工作流出现绿色对号 ✅ 标识，即表示节点检测和页面生成已经完成。
2. 获取两大访问地址
工作流完成后，GitHub Pages 会自动发布生成两个访问页面（将下方链接中的 <username> 替换为你的 GitHub 用户名，<repository> 替换为你的仓库名称）：

🌐 页面一：节点前端展示与下载页面（GitHub Pages 网址）
用于直接在浏览器端实时查看经过测速的节点列表、延迟、带宽，并可直接下载 .ovpn 配置文件：

https://你的GitHub用户名.github.io/仓库名/
📋 页面二：复制内容并粘贴到 EDG 后台的专用网页
打开此页面后，可直接全选复制页面中的节点配置文本，然后粘贴进 EDG 后台系统：

https://你的GitHub用户名.github.io/仓库名/hosts.txt
🎉 部署完成： 后续系统会根据 check.yml 中的定时配置（每 30 分钟自动运行一次）持续抓取、测速并提交最新数据，两个前端网页也会自动无缝同步最新节点。
Github仓库地址：https://github.com/hezhanleiok/gate
