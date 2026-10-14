# Radeon Cloud 部署 Bonsai2（llama.cpp PrismML 分支 + PQ2_0 GGUF）流程

- 当前实例：`ssh root@36.150.116.206 -p 33436`（免密）。IP/端口随实例变，重建后从 Profile 页抄新的，替换本文所有出现处。
- 全套已在本实例验证通过（2026-10-14）：/v1/models 返回 200 且 `ftype = PQ2_0`，客户端可连。
- 实例重建（销毁重开）后 /workspace 大概率是空的 → ①→⑧ 全跑（约 15 分钟）。仅重启先 `ls /workspace`：有 `llama.cpp-prism/` + `models/bonsai2.gguf` 就跳过 ①~④，从 ⑤ 开始。
- 本实例 /workspace 是 25G NFS 持久卷；llama-server 启动约 90 秒；全程在 Mac 终端执行。

## ① 启动模型下载（后台，约 2 分钟）

```bash
ssh root@36.150.116.206 -p 33436 'bash -s' <<'REMOTE'
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
df -h /workspace | tail -1
REMOTE
```

说明：huggingface.co 经集群代理可能 503（CONNECT tunnel failed），hf-mirror.com 同路径直接可用。上面自动挑活的，看 `src=` 用的是哪个。

## ② 上传源码包（35 MB，约 1 分钟）

```bash
scp -P 33436 ~/Downloads/llama-prism-src.tar.gz root@36.150.116.206:/workspace/
```

若报 No such file，本机包丢了，发我，我给重下命令。

## ③ 校验哈希 + 解包 + 后台编译（5–10 分钟）

```bash
ssh root@36.150.116.206 -p 33436 'bash -s' <<'REMOTE'
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

## ④ 看进度（可反复跑；两者全绿再往下）

```bash
ssh root@36.150.116.206 -p 33436 'ls -lh /workspace/models/bonsai2.gguf 2>/dev/null; tail -c 120 /workspace/models/dl.log | tr "\r" "\n" | tail -1; tail -1 /workspace/build.log'
```

- 模型：`7.2G`（= 7,646,367,744 字节 / 7.66 GB；curl 尾行 `100 7302M …` 即下完）
- 编译：`[482/482] Linking CXX executable bin/llama-server`

## ⑤ 起服务（复用已有 key，没有才生成）

```bash
ssh root@36.150.116.206 -p 33436 'bash -s' <<'REMOTE'
pkill -x llama-server; sleep 1
test -s /workspace/api.key || openssl rand -hex 24 > /workspace/api.key
nohup /workspace/llama.cpp-prism/build/bin/llama-server -m /workspace/models/bonsai2.gguf -ngl 999 -fa on -c 131072 --parallel 1 --spec-type draft-mtp --spec-draft-n-max 3 --host 127.0.0.1 --port 18180 --alias bonsai2-v2 --api-key "$(cat /workspace/api.key)" --jinja --chat-template-kwargs '{"reasoning_effort":"medium"}' > /workspace/server.log 2>&1 &
echo "srv_pid=$!"
REMOTE
sleep 120
ssh root@36.150.116.206 -p 33436 'tail -3 /workspace/server.log; echo ---KEY---; cat /workspace/api.key'
```

期望：`srv_pid=数字`；约 2 分钟后日志出现 `model loaded` + `listening on http://127.0.0.1:18180`；`---KEY---` 下面是客户端要填的 key。key 只生成一次、之后重启都复用（客户端配置不用动）；想换 key 就删 `/workspace/api.key`（客户端也要同步改）。

## ⑥ 修 sshd 转发开关（ssh -L 的前置，10 秒；实例重启后保险起见重跑）

```bash
ssh root@36.150.116.206 -p 33436 'bash -s' <<'REMOTE'
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

## ⑦ Mac 挂隧道（新终端挂着别关；自带 5 次重试）

```bash
pkill -f "18180:127.0.0.1:18180"; sleep 1; for i in 1 2 3 4 5; do ssh -f -N -o ExitOnForwardFailure=yes -o ConnectTimeout=15 -o ServerAliveInterval=30 -L 18180:127.0.0.1:18180 root@36.150.116.206 -p 33436 && { echo TUNNEL_UP; break; }; echo "retry $i"; sleep 3; done
```

期望：`TUNNEL_UP`。Mac 睡眠/断网后重跑这条。

## ⑧ 客户端 + 端到端验证

```bash
ssh root@36.150.116.206 -p 33436 'cat /workspace/api.key' > ~/.radeon-api-key; chmod 600 ~/.radeon-api-key; wc -c ~/.radeon-api-key
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

- ssh 秒断 `Connection closed by … port 33436` = 平台 SSH 网关闪断（容器没重启，别慌），重试即可；本文长命令自带重试或直接重跑。
- 别 kill PID 1（jupyter-lab，杀掉 = 整实例重启，nohup 的服务全没）。
- 隧道报 administratively prohibited → ⑥ 没做或已被重置，重跑 ⑥。
- 客户端连不上先看 401 来源：Key 框空 / 隧道断了（重跑 ⑦）。
- 长任务一律 `nohup … > log 2>&1 &`；下载断了 `curl -C -` 续传。
- 实例销毁重建：IP/端口、Jupyter token 都变；/workspace 大概率空 → 从 ① 重跑。
