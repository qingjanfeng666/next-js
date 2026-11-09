# zero-to-tech-4-5 · Next.js 版（模块 4.5 配套代码）

4.5 的终点成品：把 4.4 的 React 项目整体搬到 Next.js。网站长相不变；变的是路由层（手搓 `useRoute` → 文件夹路由）和入口（`index.html` + `main.jsx` + `App.jsx` → `app/`）。

## 跑起来

```bash
npm install
npm run dev          # http://localhost:3000   （Next 默认端口是 3000）
```

## 把你的 4.4 项目（`zero-to-tech`）迁成这样：整包替换

1. 把 `~/zero-to-tech` 里**除隐藏的 `.git` 外的所有文件删掉**。
2. 把本 demo 下的所有文件拷进去（`node_modules`、`.next` 不用拷）。
3. 跑起来确认还是那个网站：
   ```bash
   npm install
   npm run dev          # http://localhost:3000
   ```
4. 确认无误后 `git add` / `commit` / `push`。

## 项目结构

```
app/                     ← 文件夹 = 路由
  layout.jsx             ← 全站外壳（页面包裹 + import 8 个 css）
  page.jsx               ← /         → 渲染 <HomeView />
  text-lab/page.jsx      ← /text-lab → 渲染 <TextLabView />
components/
  Nav.jsx                ← <Link> + usePathname（"use client"）
  HomeView.jsx           ← 4.4 的 HomePage 改名
  TextLabView.jsx        ← 4.4 的 TextLabPage 改名
  PageHeading.jsx        ← 同 4.4
  InputCard.jsx          ← 同 4.4（"use client"）
  ResultCard.jsx         ← 同 4.4（"use client"）
  AnimatedCardGrid.jsx   ← 同 4.4（"use client"）
css/                     ← 8 个 css，同 4.4
data/site.js             ← 同 4.4
next.config.mjs          ← 空 {}
```

相比 4.4，没了 `index.html`、`src/main.jsx`、`src/App.jsx`、`src/router/useRoute.js`——这一坨被 `app/` 取代了。


## 部署到服务器

前提：服务器上已装好 Python 3.10+、Node.js 18+ 和 Nginx。

**1. 拉取代码**

```bash
cd ~/zero-to-tech
git pull
```

**2. 前端：装依赖、写配置、构建**

```bash
npm install
cp .env.example .env.production   # 按下面「配置说明」填好（线上留空）
npm run build                     # 产物进 out/，由 Nginx 提供服务
```

**3. 后端：建环境、装依赖、写配置**

```bash
cd backend
python3 -m venv --prompt=zero-to-tech .venv   # 首次部署才需要
source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env              # 按下面「配置说明」填好
```

**4. 后端：委托给 systemd**（首次部署才需要）

新建 `/etc/systemd/system/zero-backend.service`：

```ini
[Unit]
Description=zero-to-tech backend

[Service]
User=ubuntu
WorkingDirectory=/home/ubuntu/zero-to-tech/backend
ExecStart=/home/ubuntu/zero-to-tech/backend/.venv/bin/fastapi run --host 127.0.0.1
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl start zero-backend     # 启动
sudo systemctl enable zero-backend    # 开机自启
sudo systemctl status zero-backend    # 看到 active (running) 就对了
```

**5. Nginx：静态文件 ＋ `/api/` 反向代理**（首次部署才需要）

编辑 `/etc/nginx/sites-enabled/default`：

```nginx
server {
    listen 80 default_server;
    server_name _;

    root /home/ubuntu/zero-to-tech/out;
    index index.html;

    location / {
        try_files $uri $uri.html $uri/ =404;
    }

    location /api/ {
        proxy_pass http://127.0.0.1:8000;     # 末尾不要加斜杠

        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

```bash
sudo nginx -t && sudo systemctl reload nginx
```

**6. 安全组 / 防火墙**

只放行 **22（SSH）** 和 **80（HTTP）**。**8000 不要放行**——后端只监听 `127.0.0.1`，由 Nginx 转发进去。

**7. 验证**

```bash
curl localhost/api/profile        # 服务器本机，注意不带端口
```

浏览器访问 `http://服务器IP`，做一次分析、看一眼历史记录。换个无痕窗口再试，两边的历史应该互相看不到。

## 日常发布

```bash
cd ~/zero-to-tech
git pull

npm run build                          # 前端改了才需要
sudo systemctl restart zero-backend    # 后端改了才需要
```

## 常用运维命令

```bash
sudo systemctl status zero-backend     # 看状态
sudo systemctl restart zero-backend    # 重启
sudo systemctl stop zero-backend       # 停掉
journalctl -u zero-backend -f          # 实时看日志（Ctrl+C 退出）
journalctl -u zero-backend -n 50       # 看最后 50 行

sudo nginx -t                          # 改完 Nginx 配置先验语法
sudo systemctl reload nginx            # 再让它重读
```

