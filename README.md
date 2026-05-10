# macmini16gb_turboquant
🚀 M4 Mac Mini: Qwen 3.5 TurboQuant 極速部署手冊
本文件記錄了如何在 16GB M4 Mac Mini 上，利用 TurboQuant 技術實現 100+ tokens/s 的本機 LLM 推論環境，並完美串接 VS Code 開發工作流。

🛠 一、伺服器端：Mac Mini 從零開始
1. 環境準備
確保系統已安裝 Xcode Command Line Tools 及 CMake：

Bash
xcode-select --install
brew install cmake
2. 編譯 TurboQuant 專屬內核
使用 TheTom/llama-cpp-turboquant 分叉版，這對 M4 的 AMX 及 4-bit KV Cache 有極致優化。

Bash
git clone https://github.com/TheTom/llama-cpp-turboquant
cd llama-cpp-turboquant
mkdir build && cd build

# 開啟 Metal 硬體加速與原生架構優化
cmake .. -DGGML_METAL=ON -DGGML_NATIVE=ON

# 針對 M4 進行並行編譯
cmake --build . --config Release -j 10
3. 模型下載
建議使用 Qwen 3.5 9B 的 Q4_K_M 量化版本，兼顧智力與速度。

來源：Hugging Face (推薦 Unsloth 或 Bartowski 倉庫)

路徑建議：~/models/Qwen3.5-9B-Q4_K_M.gguf

4. 一鍵啟動腳本
建立 start_server.sh，鎖定 32K 上下文並開啟 Turbo 模式：

Bash
./bin/llama-server \
  -m ../../models/Qwen3.5-9B-Q4_K_M.gguf \
  -np 1 \
  --port 8080 \
  --host 0.0.0.0 \
  --ctx-size 32768 \
  --cache-type-k turbo4 \
  --cache-type-v turbo4 \
  --flash-attn on
注意：使用 turbo4 參數後，32K 的 KV Cache 僅佔用約 272MB，極大緩解 16GB 記憶體壓力。

💻 二、客戶端：IDE (Continue) 配置
在開發機的 VS Code 中，修改 config.yaml（或 config.json）。為了對抗模型「廢話」導致的解析紅叉，必須進行格式壓制。

1. YAML 完整配置
YAML
models:
  - name: M4-Qwen3.5-Turbo
    provider: openai
    model: Qwen3.5-9B-Q4_k_M
    apiBase: http://192.168.0.21:8080/v1 # 替換為 Mac Mini 實際 IP
    apiKey: EMPTY
    contextLength: 32768
    # --- 關鍵優化：奪回首字輸出權 ---
    systemPrompt: |
      You are a specialized code generator for the NoteDaily project.
      - DO NOT provide explanations, lists, or conversational text.
      - Start your output immediately with the opening markdown code block (e.g., ```html).
      - Provide the FULL content of the file.
      - End immediately after the closing markdown tag (```).
    completionOptions:
      temperature: 0.0      # 設為 0 以獲得極致穩定性
      maxTokens: 8192
      stop:                 # 強制結束標籤，解決 Generating 卡死
        - "<|im_end|>"
        - "```\n"
        - "```"
    roles:
      - chat
      - edit
      - apply
📊 三、性能指標與運作筆記 (HAL 工程師日誌)
1. 記憶體統計 (Memory Breakdown)
在 16GB M4 上，運行 32K 上下文時：

總體壓力：約 14.62GB (黃色壓力)，屬於高效運行區。

Swap 使用量：應接近 0。如果 Swap 過高，請重啟伺服器。

模型佔用：約 5.4GB / KV Cache：約 322MB。

2. 速度基準 (Benchmarks)
Prefill (PP)：~170+ tokens/s (快取命中時可達瞬間完成)。

Decode (TG)：~15-20 tokens/s (長文本下依然穩定)。

總體效能：曾實測達到 97.5 tokens/s (總時長/總 Token)。

3. 常見問題解決 (Troubleshooting)
紅叉報錯：通常是模型輸出了廢話。解決方案：在對話中加入「直接給代碼」。

卡在 Generating：伺服器顯示 200 OK 但 IDE 不動，代表緩衝區排隊中。不要點 Stop，等 10-20 秒讓渲染完成。

連線失敗：檢查啟動命令是否包含 --host 0.0.0.0。

最後更新：2026-05-10
