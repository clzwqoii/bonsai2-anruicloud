SSH 通了、/workspace 是空的 → 要重做一遍：下模型（约 3 分钟）+ 传源码（1 分钟）+ 编译（5–10 分钟），然后这个实例就永久可用了。下面 6 步按顺序粘，全程在 Mac 终端跑。

① 启动模型下载（后台，约 3 分钟下完）

code.sh
ssh root@36.150.116.206 -p 33436 'bash -s' <<'REMOTE'
export HTTP_PROXY=http://notebook-egress-proxy.amd-oneclick-lablab.svc.cluster.local:3128
export HTTPS_PROXY=$HTTP_PROXY
export NO_PROXY="localhost,127.0.0.1,.svc"
mkdir -p /workspace/models && cd /workspace/models
nohup curl -L -C - -o bonsai2.gguf "https://huggingface.co/BoldingBuilds/Ternary-Bonsai-2-27B-Abliterated-v2-PQ2_0-MTP-GGUF/resolve/main/Ternary-Bonsai-2-27B-Abliterated-v2-PQ2_0-MTP.gguf" > dl.log 2>&1 &
df -h /workspace | tail -1
echo DOWNLOAD_STARTED
REMOTE
② 上传源码包（35 MB，约 1 分钟）

code.sh
scp -P 33436 ~/Downloads/llama-prism-src.tar.gz root@36.150.116.206:/workspace/
若报 No such file，说明本机包丢了，告诉我，我给重下命令。

③ 校验哈希 + 解包 + 后台编译

code.sh
ssh root@36.150.116.206 -p 33436 'bash -s' <<'REMOTE'
cd /workspace
echo "0c551cf939445c296592214b0a07b3f9d2294f372e5ef5cd28bcf7aa6191c038  llama-prism-src.tar.gz" | sha256sum -c - || exit 1
mkdir -p /workspace/src && tar -xzf llama-prism-src.tar.gz -C /workspace/src
mv /workspace/src/* /workspace/llama.cpp-prism
cd /workspace/llama.cpp-prism
export PATH=$PATH:/opt/rocm/bin
export HIPCXX="$(hipconfig -l)/clang" HIP_PATH="$(hipconfig -R)"
nohup bash -c 'cmake -B build -G Ninja -DGGML_HIP=ON -DGPU_TARGETS=gfx1100 -DCMAKE_BUILD_TYPE=Release -DLLAMA_CURL=OFF && cmake --build build -j 64 --target llama-server' > /workspace/build.log 2>&1 &
echo BUILD_STARTED
REMOTE
哈希不匹配会停在第一步（FAILED），那就别继续，发我。

④ 隔几分钟看进度（可反复跑）

code.sh
ssh root@36.150.116.206 -p 33436 'ls -lh /workspace/models/bonsai2.gguf 2>/dev/null; tail -1 /workspace/build.log'
模型到 7.7G、build.log 出现 [482/482] Linking CXX executable bin/llama-server 就是都好了。

⑤ 起服务（等 ④ 全绿后跑）

code.sh
ssh root@36.150.116.206 -p 33436 'bash -s' <<'REMOTE'
KEY=$(openssl rand -hex 24); echo "$KEY" > /workspace/api.key
nohup /workspace/llama.cpp-prism/build/bin/llama-server -m /workspace/models/bonsai2.gguf -ngl 999 -fa on -c 131072 --parallel 1 --spec-type draft-mtp --spec-draft-n-max 3 --host 127.0.0.1 --port 18180 --alias bonsai2-v2 --api-key "$(cat /workspace/api.key)" --jinja --chat-template-kwargs '{"reasoning_effort":"medium"}' > /workspace/server.log 2>&1 &
echo SERVER_STARTED
REMOTE
sleep 120
ssh root@36.150.116.206 -p 33436 'tail -3 /workspace/server.log; echo ---KEY---; cat /workspace/api.key'
server is listening 出现即成功（加载约 90 秒），---KEY--- 下面是客户端要填的 key。

⑥ Mac 开隧道（新终端挂着，别关）+ 客户端配置

code.sh
ssh -N -L 18180:127.0.0.1:18180 root@36.150.116.206 -p 33436
客户端：Base URL http://127.0.0.1:18180/v1，API Key 填 ⑤ 打出的那串，Model ID bonsai2-v2。

如果秒断可能是正常的, 多试几次
如果还是秒断 Connection closed，用这条进（绕过登录配置文件）

code.sh
ssh -t root@36.150.116.206 -p 33436 'bash --norc'
还不行就把 ssh -v root@36.150.116.206 -p 33436 2>&1 | tail -15 的输出发我。

3. 存个别名，之后一条命令搞定（在 Mac 上跑一次）

code.sh
cat >> ~/.ssh/config <<'EOF'
 
Host radeon
  HostName 36.150.116.206
  Port 33436
  User root
  ServerAliveInterval 30
EOF
之后：开 shell = ssh radeon；传文件 = scp ~/Downloads/llama-prism-src.tar.gz radeon:/workspace/；开隧道 = ssh -N -L 18180:127.0.0.1:18180 radeon。

4. 在 SSH 会话里，直接粘这些（不用再包 ssh ... <<REMOTE 外壳了）
