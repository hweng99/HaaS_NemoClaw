# HANDOFF.md

## Goal

讓 NemoClaw / OpenClaw 可以打開 browser，操作 HaaS NemoClaw Ops Console，完整跑完 demo 流程，讓團隊可以螢幕錄影。

錄影要呈現的感覺：

- NemoClaw 像長程代理營運員一樣在瀏覽器裡操作。
- 它能審核 scout lead、執行 policy guardrails、產生任務草稿、核准、驗收、提出 reward proposal。
- 它能拒絕 unsafe request。
- 它能把金融高風險需求改寫成一般教育型任務。
- 右側 audit log 會持續更新。
- pause / reload / resume 能展示持久性。

目前不要只跑 terminal demo，因為 terminal 很難錄出「代理在瀏覽器維運」的感覺。

## Current State

專案路徑：

```text
/Users/user/Desktop/程式碼/2026Nvidia/Haas-Nvidia (1)
```

已新增/整理的核心檔案：

```text
main.py
README.md
SETUP.md
DEMO.md
ARCHITECTURE.md
SAFETY.md
openclaw-ops.html
NEMOCLAW_RUN_REPORT.md
```

其中：

- `main.py` 是離線 Long Agent terminal runtime。
- `openclaw-ops.html` 是瀏覽器操作版 Ops Console，這才是錄影主入口。
- `nemoclaw-demo.html` 是舊的 scripted Next Operation demo，可當備案。

目前觀察到的問題：

- 使用者把 prompt 丟給 OpenClaw 後，畫面像卡在 `You / Tool / Assistant`。
- 可能是 OpenClaw browser tool 沒成功開頁、無法連到 `host.docker.internal:8877`、或者 OpenClaw 正在等工具回傳。
- 需要下一位工程師確認 NemoClaw sandbox 的瀏覽器是否能打開本機 server。

## Desired Browser Demo URL

Mac 本機瀏覽器：

```text
http://127.0.0.1:8877/openclaw-ops.html
```

NemoClaw / OpenClaw container 內瀏覽器通常使用：

```text
http://host.docker.internal:8877/openclaw-ops.html
```

如果 `host.docker.internal` 不能用，改測 Mac 的 LAN IP，例如：

```text
http://192.168.x.x:8877/openclaw-ops.html
```

啟 server 時要 bind 到 `0.0.0.0` 才比較容易被 container 連到：

```bash
cd "/Users/user/Desktop/程式碼/2026Nvidia/Haas-Nvidia (1)"
python3 -m http.server 8877 --bind 0.0.0.0
```

## NemoClaw Sandbox Lifecycle

先 onboard：

```bash
nemoclaw onboard
```

重建 sandbox：

```bash
nemoclaw nemoclaw rebuild --yes
```

銷毀 sandbox：

```bash
nemoclaw nemoclaw destroy --force
```

取得 dashboard / OpenClaw URL：

```bash
nemoclaw nemoclaw dashboard-url --quiet
```

用該 URL 開啟 OpenClaw，完成 token 認證。

## Identify Container

在 Mac 終端機執行：

```bash
docker ps | grep nemoclaw
```

容器名格式類似：

```text
openshell-nemoclaw-6fd8bd69-76e6-42b1-b763-e8e677b40872
```

以下用變數表示，工程師請替換：

```bash
CONTAINER=openshell-nemoclaw-6fd8bd69-76e6-42b1-b763-e8e677b40872
```

## Connect Docker Network

如果需要讓 container 有一般 Docker bridge network：

```bash
docker network create nemoclaw-net
docker network connect nemoclaw-net "$CONTAINER"
```

如果 `network create` 顯示 already exists，可以忽略。

注意：這些 `docker ...` 指令要在 Mac 終端機跑，不是在 container 的 `#` shell 裡跑。

## Test Container Internet

```bash
docker exec "$CONTAINER" \
  python3 -c "import urllib.request; print(urllib.request.urlopen('https://example.com', timeout=5).status)"
```

預期：

```text
200
```

如果失敗，代表 container 外網不通。錄影 demo 不一定需要外網，但若工程師要測外部網站或 connector，需先處理 Docker / NemoClaw network policy。

## Test Container Can Reach Browser Demo Server

先在 Mac 端啟動 server：

```bash
cd "/Users/user/Desktop/程式碼/2026Nvidia/Haas-Nvidia (1)"
python3 -m http.server 8877 --bind 0.0.0.0
```

另開一個 Mac 終端機，測 container 能否打到頁面：

```bash
docker exec "$CONTAINER" \
  python3 -c "import urllib.request; r=urllib.request.urlopen('http://host.docker.internal:8877/openclaw-ops.html', timeout=5); print(r.status, r.headers.get('content-type'))"
```

預期：

```text
200 text/html
```

如果這裡失敗：

1. 確認 server 還在跑。
2. 確認 port 是 `8877`，不是被舊 server 佔用的 `8765`。
3. 用 Mac 本機瀏覽器打開 `http://127.0.0.1:8877/openclaw-ops.html`。
4. 若 container 不能解析 `host.docker.internal`，改用 Mac LAN IP。
5. 檢查 Docker Desktop 是否允許 host networking / host gateway。

## Copy Project Into Container

雖然瀏覽器 demo 可以直接由 Mac server 提供，但 terminal runtime 或 container 內驗證仍可 copy 專案。

```bash
docker exec "$CONTAINER" mkdir -p /workspace/haas-nemoclaw
```

替換為本地專案路徑：

```bash
docker cp "/Users/user/Desktop/程式碼/2026Nvidia/Haas-Nvidia (1)/." \
  "$CONTAINER":/workspace/haas-nemoclaw
```

進 container：

```bash
docker exec -it "$CONTAINER" sh
```

容器內：

```bash
cd /workspace/haas-nemoclaw
python3 main.py --mode demo
```

重置後跑完整 terminal demo：

```bash
python3 main.py --mode reset
python3 main.py --mode demo
python3 main.py --mode policy-test
```

注意：terminal demo 不是錄影主線，只是確認核心 Long Agent logic 可用。

## Browser Recording Flow

錄影主入口：

```text
http://host.docker.internal:8877/openclaw-ops.html
```

給 OpenClaw / NemoClaw 的建議 prompt：

```text
Open http://host.docker.internal:8877/openclaw-ops.html.
Operate the HaaS NemoClaw Ops Console as a long-agent system admin.
Review sig-001, run policy, draft quest, approve, verify, and propose reward.
Then review sig-002 and refuse it because it violates privacy/confrontation policy.
Then review sig-003 and rewrite it into a general educational market-risk task.
Show the audit log and pause/resume persistence.
```

人工/工程師可手動確認流程：

1. 點左側 `sig-001`。
2. 點 `Run Policy`。
3. 點 `Draft Quest`。
4. 點 `Approve Draft`。
5. 點 `Verify Submission`。
6. 點 `Propose Reward`。
7. 點左側 `sig-002`。
8. 點 `Run Policy`。
9. 點 `Refuse Unsafe`。
10. 點左側 `sig-003`。
11. 點 `Run Next` 直到完成 rewrite / draft / approve / verify / reward。
12. 點 `Pause`。
13. 重新整理頁面。
14. 點 `Reload State` 或 `Resume`，展示 localStorage 持久性。
15. 右側 audit log 應該顯示每一步。

## If OpenClaw Still Looks Stuck

排查順序：

1. 先用 Mac 本機瀏覽器打開：

```text
http://127.0.0.1:8877/openclaw-ops.html
```

確認頁面本身可互動。

2. 用 container 測 URL：

```bash
docker exec "$CONTAINER" \
  python3 -c "import urllib.request; print(urllib.request.urlopen('http://host.docker.internal:8877/openclaw-ops.html', timeout=5).status)"
```

3. 若測得 `200`，但 OpenClaw 仍卡住，問題多半在 OpenClaw browser/tool loop，不在 demo 頁。

4. 改把 URL 換成 Mac LAN IP：

```text
http://<MAC_LAN_IP>:8877/openclaw-ops.html
```

Mac LAN IP 可用：

```bash
ipconfig getifaddr en0
```

或：

```bash
ipconfig getifaddr en1
```

5. 如果 OpenClaw 可以開外網但不能開 host URL，可以暫時用 ngrok 作為錄影 workaround：

```bash
ngrok http 8877
```

但這需要外網，而且和「離線可跑」的競賽主張不同。只建議用於錄影應急。

## Important Notes

- `docker exec ...`、`docker cp ...`、`docker network ...` 都在 Mac 終端機跑。
- 進入 container 後看到 `#`，只能跑容器內指令，例如 `cd`、`python3`、`ls`。
- `docker ps` 顯示 `(unhealthy)` 不一定代表 demo 不能跑。之前已確認 container 內 `python3 main.py --mode demo` 可以正常執行。
- `8765` 曾經被舊 server 佔用且出現 empty reply，建議使用 `8877`。
- `openclaw-ops.html` 是錄影主線，`main.py` 是 terminal logic proof，兩者都不需要外網。

## Next Engineer Tasks

1. 重新啟動或確認 Mac server：

```bash
cd "/Users/user/Desktop/程式碼/2026Nvidia/Haas-Nvidia (1)"
python3 -m http.server 8877 --bind 0.0.0.0
```

2. 確認 Mac 本機可開：

```text
http://127.0.0.1:8877/openclaw-ops.html
```

3. 確認 container 可讀：

```bash
docker exec "$CONTAINER" \
  python3 -c "import urllib.request; print(urllib.request.urlopen('http://host.docker.internal:8877/openclaw-ops.html', timeout=5).status)"
```

4. 打開 OpenClaw dashboard：

```bash
nemoclaw nemoclaw dashboard-url --quiet
```

5. 在 OpenClaw 內輸入 browser demo prompt。

6. 如果 OpenClaw 卡住，先改 URL 為 Mac LAN IP，再試一次。

7. 若仍卡住，請檢查 OpenClaw 是否真的有 browser/control tool 啟用；如果沒有，就需要用人工 browser 操作錄影，或改用 OpenClaw 提供的瀏覽器工具設定。
