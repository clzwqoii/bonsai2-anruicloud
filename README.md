### 1.
我在 Mac 上拉好了 fork 的 prism 分支源码包：/tmp/llama-prism-src.tar.gz，35.6 MB，sha256 0c551cf939445c296592214b0a07b3f9d2294f372e5ef5cd28bcf7aa6191c038。

### 2.
```bash
# 期望 0c551cf939445c296592214b0a07b3f9d2294f372e5ef5cd28bcf7aa6191c038
sha256sum /workspace/llama-prism-src.tar.gz   


cd /workspace && tar xzf llama-prism-src.tar.gz && cd llama.cpp-prism
HIPCXX="$(hipconfig -l)/clang" HIP_PATH="$(hipconfig -R)" \
cmake -B build -G Ninja -DGGML_HIP=ON -DGPU_TARGETS=gfx1100 -DCMAKE_BUILD_TYPE=Release -DLLAMA_CURL=OFF
nohup cmake --build build -j 64 --target llama-server > /workspace/build.log 2>&1 &
sleep 45; tail -25 /workspace/build.log
```
### 3.
```bash
# 期望 build 10772 / e8fd209c9
B=/workspace/llama.cpp-prism/build/bin/llama-server; "$B" --version  
KEY=$(openssl rand -hex 24); echo "$KEY"

nohup "$B" -m /workspace/models/bonsai2.gguf \
  -ngl 999 -fa on -c 262144 --parallel 1 \
  --spec-type draft-mtp --spec-draft-n-max 3 \
  --host 127.0.0.1 --port 18180 --alias bonsai2-v2 \
  --api-key "$KEY" \
  --jinja --chat-template-kwargs '{"reasoning_effort":"medium"}' \
  > /workspace/server.log 2>&1 &

# 它返回一个 https://rc-<随机>.radeon.firstdg.ai。脚本报「identity 目录/FRP_BROKER_URL 不存在」= 这个 Pod 建得太早，那就只能重建 Notebook（/workspace 是持久盘，模型和编译产物都在，不用重来）
/var/run/secrets/frp-self-service/install
$HOME/.local/bin/rc-tunnel expose --port 18180


curl -s -H "Authorization: Bearer $KEY" http://127.0.0.1:18180/v1/models

curl -s -H "Authorization: Bearer $KEY" -H 'Content-Type: application/json' \
  http://127.0.0.1:18180/v1/chat/completions \
  -d '{"model":"bonsai2-v2","messages":[{"role":"user","content":"用一句话说明你是谁"}],"max_tokens":128}'
```
