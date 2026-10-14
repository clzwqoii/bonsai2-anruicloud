1.
我在 Mac 上拉好了 fork 的 prism 分支源码包：/tmp/llama-prism-src.tar.gz，35.6 MB，sha256 0c551cf939445c296592214b0a07b3f9d2294f372e5ef5cd28bcf7aa6191c038。

2.
# 期望 0c551cf939445c296592214b0a07b3f9d2294f372e5ef5cd28bcf7aa6191c038
sha256sum /workspace/llama-prism-src.tar.gz   


cd /workspace && tar xzf llama-prism-src.tar.gz && cd llama.cpp-prism
HIPCXX="$(hipconfig -l)/clang" HIP_PATH="$(hipconfig -R)" \
cmake -B build -G Ninja -DGGML_HIP=ON -DGPU_TARGETS=gfx1100 -DCMAKE_BUILD_TYPE=Release -DLLAMA_CURL=OFF
nohup cmake --build build -j 64 --target llama-server > /workspace/build.log 2>&1 &
sleep 45; tail -25 /workspace/build.log

3.
# 期望 build 10772 / e8fd209c9
B=/workspace/llama.cpp-prism/build/bin/llama-server; "$B" --version   

nohup "$B" -m /workspace/models/bonsai2.gguf -ngl 999 -fa on -c 65536 --parallel 1 \
  -ctk q8_0 -ctv q8_0 --spec-type draft-mtp --spec-draft-n-max 3 \
  --host 127.0.0.1 --port 18180 --alias bonsai2-v2 --jinja \
  --chat-template-kwargs '{"reasoning_effort":"medium"}' \
  --temp 1.0 --top-p 0.95 --top-k 20 > /workspace/server.log 2>&1 &
sleep 90; tail -20 /workspace/server.log; curl -s http://127.0.0.1:18180/v1/models
