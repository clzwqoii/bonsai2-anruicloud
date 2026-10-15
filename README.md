# Radeon Cloud 部署 Bonsai2（llama.cpp PrismML 分支 + PQ2_0 GGUF）流程

- 当前实例：`ssh root@36.150.116.206 -p 31548`（免密）。IP/端口随实例变，重建后从 Profile 页抄新的，替换本文所有出现处。
- 全套已在本实例验证通过（2026-10-14；2026-10-15 追加 256K 上下文与图片输入 --mmproj）：/v1/models 返回 200 且 `ftype = PQ2_0`，客户端可连。
- 实例重建（销毁重开）后 /workspace 大概率是空的 → ①→⑧ 全跑（约 15 分钟）。**实例重启（非销毁）**：/workspace 数据保留，但 nohup 的服务全没、⑥ 的 sshd 配置被平台重置回 no、**SSH 端口和主机名也会变** → 只需重跑 ⑤⑥⑦（几分钟）。判断法：先 `ls /workspace`，有 `llama.cpp-prism/` + `models/bonsai2.gguf` + `models/bonsai2-mmproj.gguf` 就跳过 ①~④，从 ⑤ 开始；从 Profile 页抄新端口，替换本文所有出现处。
- 本实例 /workspace 是 25G NFS 持久卷；llama-server 启动约 90 秒；全程在 Mac 终端执行。

## ① 启动模型下载（后台，约 2 分钟；主模型 7.2G + mmproj 629M 一起下）

```bash
ssh root@36.150.116.206 -p 31548 'bash -s' <<'REMOTE'
export HTTP_PROXY=http://notebook-egress-proxy.amd-oneclick-lablab.svc.cluster.local:3128
export HTTPS_PROXY=$HTTP_PROXY
export NO_PROXY="localhost,127.0.0.1,.svc"
mkdir -p /workspace/models && cd /workspace/models
C1=$(curl -s -o /dev/null -m 15 -w "%{http_code}" https://huggingface.co/)
C2=$(curl -s -o /dev/null -m 15 -w "%{http_code}" https://hf-mirror.com/)
echo "probe hf=$C1 mirror=$C2"
B="https://hf-mirror.com/BoldingBuilds/Ternary-Bonsai-2-27B-Abliterated-v2-PQ2_0-MTP-GGUF/resolve/main/Ternary-Bonsai-2-27B-Abliterated-v2-PQ2_0-MTP.gguf"
[ "$C2" = "200" ] || B="https://huggingface.co/BoldingBuilds/Ternary-Bonsai-2-27B-Abliterated-v2-PQ2_0-MTP-GGUF/resolve/main/Ternary-Bonsai-2-27B-Abliterated-v2-PQ2_0-MTP.gguf"
nohup curl -L -C - --retry 5 --retry-delay 5 -o bonsai2.gguf "$B" > dl.log 2>&1 &
echo "dl_pid=$! src=$B"
M="https://hf-mirror.com/prism-ml/Ternary-Bonsai-2-27B-gguf/resolve/main"
[ "$C2" = "200" ] || M="https://huggingface.co/prism-ml/Ternary-Bonsai-2-27B-gguf/resolve/main"
nohup curl -L -C - --retry 5 --retry-delay 5 -o bonsai2-mmproj.gguf "$M/Ternary-Bonsai-2-27B-mmproj-Q8_0.gguf" > mmproj-dl.log 2>&1 &
echo "mmproj_pid=$! src=$M"
df -h /workspace | tail -1
REMOTE
```

说明：huggingface.co 经集群代理可能 503（CONNECT tunnel failed），hf-mirror.com 同路径直接可用。上面自动挑活的，看 `src=` 用的是哪个。mmproj 是视觉投影文件（图片输入用，来自上游 `prism-ml/Ternary-Bonsai-2-27B-gguf` 仓库；Q8_0 版官方 629,246,976 字节）；不需要图片功能可以不下，⑤ 里删掉 `--mmproj` 即可。

## ② 上传源码包（35 MB，约 1 分钟）

```bash
scp -P 31548 ~/Downloads/llama-prism-src.tar.gz root@36.150.116.206:/workspace/
```

若报 No such file，本机包丢了，发我，我给重下命令。

## ③ 校验哈希 + 解包 + 后台编译（5–10 分钟）

```bash
ssh root@36.150.116.206 -p 31548 'bash -s' <<'REMOTE'
cd /workspace
echo "0c551cf939445c296592214b0a07b3f9d2294f372e5ef5cd28bcf7aa6191c038 llama-prism-src.tar.gz" | sha256sum -c - || exit 1
mkdir -p /workspace/src && tar -xzf llama-prism-src.tar.gz -C /workspace/src
mv /workspace/src/* /workspace/llama.cpp-prism
cd /workspace/llama.cpp-prism
export PATH=$PATH:/opt/rocm/bin
export HIPCXX="$(hipconfig -l)/clang" HIP_PATH="$(hipconfig -R)"
nohup bash -c 'cmake -B build -G Ninja -DGGML_HIP=ON -DGPU_TARGETS=gfx1100 -DCMAKE_BUILD_TYPE=Release -DLLAMA_CURL=OFF && cmake --build build -j 64 --target llama-server' > /workspace/build.log 2>&1 &
echo BUILD_STARTED
REMOTE
```

哈希不匹配会停在第一步（FAILED），别继续，发我。

## ④ 看进度（可反复跑；全部完成再往下）

```bash
ssh root@36.150.116.206 -p 31548 'ls -lh /workspace/models/bonsai2*.gguf 2>/dev/null; tail -c 120 /workspace/models/dl.log | tr "\r" "\n" | tail -1; tail -c 120 /workspace/models/mmproj-dl.log | tr "\r" "\n" | tail -1; tail -1 /workspace/build.log'
```

- 模型：`7.2G`（= 7,646,367,744 字节 / 7.66 GB；curl 尾行 `100 7302M …` 即下完）
- mmproj：`629M`（= 629,246,976 字节；curl 尾行 `100 600M …` 即下完；sha256 官方值 `6807ede61d570bb86ba34b756a0fa109edc33668604de867c6ea6d8f1d631903`）
- 编译：`[482/482] Linking CXX executable bin/llama-server`

## ⑤ 起服务（复用已有 key，没有才生成）

```bash
ssh root@36.150.116.206 -p 31548 'bash -s' <<'REMOTE'
pkill -x llama-server; sleep 1
test -s /workspace/api.key || openssl rand -hex 24 > /workspace/api.key
nohup /workspace/llama.cpp-prism/build/bin/llama-server -m /workspace/models/bonsai2.gguf --mmproj /workspace/models/bonsai2-mmproj.gguf -ngl 999 -fa on -c 262144 --parallel 1 --spec-type draft-mtp --spec-draft-n-max 3 --host 127.0.0.1 --port 18180 --alias bonsai2-v2 --api-key "$(cat /workspace/api.key)" --jinja --chat-template-kwargs '{"reasoning_effort":"medium"}' > /workspace/server.log 2>&1 &
echo "srv_pid=$!"
REMOTE
sleep 120
ssh root@36.150.116.206 -p 31548 'tail -3 /workspace/server.log; echo ---KEY---; cat /workspace/api.key'
```

期望：`srv_pid=数字`；约 2 分钟后日志出现 `loaded multimodal model` + `model loaded` + `listening on http://127.0.0.1:18180`；`---KEY---` 下面是客户端要填的 key。key 只生成一次、之后重启都复用（客户端配置不用动）；想换 key 就删 `/workspace/api.key`（客户端也要同步改）。

`-c 262144` = 256K 上下文（模型训练上限）；实测显存占用 ≈26G/48G。

`--mmproj /workspace/models/bonsai2-mmproj.gguf` = 视觉投影（让模型能看图）；启动日志出现 `loaded multimodal model, '...mmproj.gguf'` 才算加载成功。不要图片就删掉这个参数，文本不受影响。

## ⑥ 修 sshd 转发开关（ssh -L 的前置，10 秒；实例重启后保险起见重跑）

```bash
ssh root@36.150.116.206 -p 31548 'bash -s' <<'REMOTE'
mkdir -p /run/sshd
cp -a /tmp/amd-oneclick-sshd_config /tmp/amd-oneclick-sshd_config.bak 2>/dev/null
sed -i "s/^AllowTcpForwarding no/AllowTcpForwarding local/" /tmp/amd-oneclick-sshd_config
grep -n "^AllowTcpForwarding" /tmp/amd-oneclick-sshd_config
P=$(ps -eo pid,args | grep "[s]shd -f /tmp/amd-oneclick-sshd_config" | awk '{print $1; exit}')
echo "master=$P"
sshd -t -f /tmp/amd-oneclick-sshd_config && kill -HUP "$P" && echo RELOADED
REMOTE
```

期望：`20:AllowTcpForwarding local` → `master=数字` → `RELOADED`。平台 sshd 读的是 `/tmp/amd-oneclick-sshd_config`（不是 /etc/ssh/sshd_config），默认 `AllowTcpForwarding no`，不改这步隧道会被 administratively prohibited 拒。命令幂等，重跑无害。

## ⑦ Mac 挂隧道（常驻自动重连，挂一次不用管）

```bash
pkill -f "18180:127.0.0.1:181[80]"; sleep 1
nohup bash -c 'while :; do ssh -N -o ExitOnForwardFailure=yes -o ConnectTimeout=15 -o ServerAliveInterval=30 -o ServerAliveCountMax=3 -L 18180:127.0.0.1:18180 root@36.150.116.206 -p 31548; sleep 5; done' > /tmp/bonsai-tunnel.log 2>&1 &
sleep 4; curl -s -o /dev/null -w "HTTP %{http_code}\n" -H "Authorization: Bearer $(cat ~/.radeon-api-key)" http://127.0.0.1:18180/v1/models
```

期望：`HTTP 200`（401 = ⑧ 的 `~/.radeon-api-key` 没准备好，先跑 ⑧ 第一条）。断网/Mac 睡眠恢复、实例重启后 ssh 断开，都会**每 5 秒自动重连**，不用手动重跑。想停隧道：`pkill -f "18180:127.0.0.1:181[80]"`；Mac 重启后重新跑本节；实例换了端口就改命令里的 `-p 31548`。日志在 `/tmp/bonsai-tunnel.log`。

## ⑧ 客户端 + 端到端验证

```bash
ssh root@36.150.116.206 -p 31548 'cat /workspace/api.key' > ~/.radeon-api-key; chmod 600 ~/.radeon-api-key; wc -c ~/.radeon-api-key
curl -s -w '\nHTTP %{http_code}\n' -H "Authorization: Bearer $(cat ~/.radeon-api-key)" http://127.0.0.1:18180/v1/models
```

`wc -c` 应为 `49`（打 0 = ssh 被网关闪断，重跑）；curl 期望 `HTTP 200` + `"ftype":"PQ2_0 - 2.13 bpw (group 128)"`。

客户端填：

- Base URL：`http://127.0.0.1:18180/v1`
- API Key：`cat ~/.radeon-api-key` 的内容（保存时不能留空——改过 Base URL 后保存会把空 key 存进去 → 401）
- Model：`bonsai2-v2`

可选冒烟（真出一次字）：

```bash
curl -s -H "Authorization: Bearer $(cat ~/.radeon-api-key)" -H "Content-Type: application/json" -d '{"model":"bonsai2-v2","messages":[{"role":"user","content":"你好"}],"max_tokens":200}' http://127.0.0.1:18180/v1/chat/completions | head -c 600
```

## 小抄

- ssh 秒断 `Connection closed by … port 31548` = 平台 SSH 网关闪断（容器没重启，别慌），重试即可；本文长命令自带重试或直接重跑。
- 别 kill PID 1（jupyter-lab，杀掉 = 整实例重启，nohup 的服务全没）。
- 隧道报 administratively prohibited → ⑥ 没做或已被重置，重跑 ⑥。
- 客户端连不上先看 401 来源：Key 框空 / 隧道断了（重跑 ⑦）。
- 发图报 `image input is not supported` → 启动命令没带 `--mmproj`（或实例没下 mmproj 文件），见 ⑤。
- 每次重启 llama-server 会清空前缀缓存：老会话的下一条消息要全量预填充（10 万 token 级会话约 5-7 分钟），属正常，不是卡死。
- 长任务一律 `nohup … > log 2>&1 &`；下载断了 `curl -C -` 续传。
- **长任务别放在客户端/交互连接里跑**：平台 SSH 网关会掐长时间不动的连接（实测约半小时），客户端断开 → llama-server 收到断连就把任务取消（日志 `cancel task`）→ 「跑到一半停了」。服务端长活（下载、编译）必须 nohup；客户端长对话被掐就等隧道自动重连后重发。
- 实例重启（非销毁）也会**换 SSH 端口/主机名**，但 /workspace 和 api.key 都在：抄新端口 → ⑤⑥⑦ 三步恢复；`~/.radeon-api-key` 内容不变不用重建（key 来自 /workspace/api.key）。
- 实例销毁重建：IP/端口、Jupyter token 都变；/workspace 大概率空 → 从 ① 重跑。
