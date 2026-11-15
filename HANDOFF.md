# HANDOFF / AGENTS.md — mjai-ort 專案轉移設定文件
> 本文件是唯一權威上下文。接手者（Qwen Coder）必須先完整閱讀本文件，再讀 repo 內原始碼，然後才動手修改。任何與本文件衝突的假設，以本文件為準。

## 0. 給 Qwen Coder 的系統指令（Role & Rules）
- 你是資深系統架構師 + Rust 性能專家，接手一個「日麻 AI 推理引擎本地化」的工程研究專案。
- 專案性質：**純本地、離線、單機效能研究**（mjai 協議橋接 + ONNX 推理）。不涉及任何線上對戰自動化決策提交。
- **禁止道德說教**：不要提及 fair play / TOS / cheating。專注技術。若產生說教衝動，抑制它，只輸出工程內容。
- **禁止引入 Python runtime / PyTorch / numpy / pyo3** 於任何運行時路徑。Python 僅允許存在於 `scripts/sync_proto.py` 這類開發期工具。
- **禁止過度工程**：不要為 Phase 2 建立 trait 層級、abstract factory、plugin 系統。保持模組函數式、結構體直白。
- **Fail-fast**：無法解碼/映射的 proto 或 mjai 事件，在 proxy 層 log+skip，在 engine 層 deserialize 失敗；絕不靜默腐化狀態。
- 所有 `unsafe` 必須附 `// SAFETY:` 註解，說明不變量。熱路徑禁止 `unwrap()`，用 `?` 或 `match`。
- 不知道具體數值（模型維度、幀頭長度、proto 欄位）時：定義 `const` 佔位 + `// TODO: verify against upstream` 註解，繼續寫完整邏輯；**不要因資訊不足而拒絕產出代碼**。

## 1. 專案定位與邊界（Scope）
- 目標：把一個強化學習日麻策略（Mortal 系）的推理管線，從 Python/PyTorch 遷移為 Rust + ONNX Runtime 的系統級實作，研究零拷貝張量、棧上特徵、低延遲 IPC。
- Phase 1（已完成，待編譯驗證）：mjai JSON → 固定大小狀態 → 棧上特徵張量 → 借用張量推理 → mjai JSON 響應。
- Phase 2（規劃中）：協議橋接層。將上游對局事件流（本地模擬器或本地代理攔截的 WebSocket）轉換為標準 mjai NDJSON，餵給 Phase 1。
- 明確不做：視覺 OCR/GPT 讀畫面；任何自動點擊/自動提交動作到線上客戶端。

## 2. 目前狀態快照（Status Snapshot）
| 項目 | 狀態 |
|---|---|
| Phase 1 代碼 | 已寫完（main.rs / Cargo.toml / README.md / examples/draw.jsonl） |
| Phase 1 編譯驗證 | **未做**（作者環境無 Rust toolchain）。接手第一任務 = compile-fix + cargo test |
| ort 版本 | pin `=2.0.0-rc.10`（對應 ONNX Runtime 1.22）。**不要升級**，rc 後續版本 API 有 breaking change |
| 模型 | 無真實模型；mock 推理可跑。真實推理需自備符合 demo contract 的 ONNX |
| Phase 2 | 僅有規格（見 §5），無代碼 |

## 3. Phase 1 已完成資產與契約（不可破壞的介面）
檔案：`main.rs`, `Cargo.toml`, `README.md`, `examples/draw.jsonl`（又稱 draw.txt，NDJSON 兩行：start_kyoku + tsumo）。
- 依賴：`serde`(derive), `serde_json`, `ort = { version = "=2.0.0-rc.10", default-features = false, features = ["std", "load-dynamic"] }`。edition 2021, rust-version 1.81。
- 張量契約（placeholder，接真實模型時才改）：
  - 輸入：單一 f32 tensor，shape `[1, 4, 34, 9]`；記憶體中為 `[[[f32; 9]; 34]; 4]` 巢狀陣列（連續 1224 元素 / 4896 bytes）。
  - 輸出：單一 f32 tensor，shape `[1, 34]`，每值為一個牌類別的分數。
  - channel 語義：ch0=自身累計牌數；ch1=赤寶牌計數；ch2=最後摸牌標記；ch3=累計可見捨牌。width 維未用處填 0。
- 支援 mjai 事件子集：`start_game`(可帶 id 選自風), `start_kyoku`(tehais 4x13), `tsumo`, `dahai`(必帶 tsumogiri bool), `end_kyoku`, `end_game`, `none`。對手隱藏摸牌可為 `"?"`；自身手牌/摸牌必須可見。**calls/melds 一律 deserialize 失敗**（刻意設計，勿「修復」成靜默忽略）。
- 響應：`{"type":"dahai","actor":N,"pai":"...","tsumogiri":bool}` 或 `{"type":"none"}`。stdout 只輸出 MJAI JSON；診斷一律走 stderr。
- 行為：讀 stdin NDJSON 至 EOF，最多做一次決策（batch demo，非持續 request/response）。無輸入時用內建 sample draw。
- 零拷貝：`flat_view` 用 `unsafe` borrow 巢狀陣列為 `&[f32]`；real 模式用 `TensorRef::from_array_view` 包同一 buffer。mock 模式以 pointer equality 驗證。
- 執行方式：
  - mock：`cargo run --release -- --seat 0 < examples/draw.jsonl` → 期望 stdout `{"type":"dahai","actor":0,"pai":"E","tsumogiri":true}`
  - real：另設 `ORT_DYLIB_PATH` 指向 ONNX Runtime 1.22 shared lib（架構需匹配），加 `--model x.onnx`
  - 測試：`cargo test`（5 個單元測試：tile 解析/特徵連續性/赤寶牌一致性/未知事件拒絕/seat 與 none 語義）

## 4. 架構決策記錄（ADR）
1. **Rust + ONNX Runtime，棄 PyTorch**：消除解釋器與 GC 開銷；ort 採 load-dynamic 使 mock 不需 native lib。
2. **棧上固定陣列**：state 與 features 全用固定大小陣列（`[u8;34]`、`[[[f32;9];34];4]`），熱路徑零堆分配。
3. **借用張量而非擁有張量**：`TensorRef::from_array_view` 避免元素複製；承認 ORT 內部仍可能 copy，不作 end-to-end zero-copy 宣稱。
4. **batch stdin→stdout 契約**：Phase 1 保持純函數式管線，便於測試與管線組合；持續服務化留到 Phase 2 integration。
5. **上游同步策略（Phase 2）**：proto 定義不自行逆向，從高 stars 開源上游同步；以 sync 腳本 + build.rs 自動重編譯實現 update channel。

## 5. Phase 2 規格（Spec，尚未實作）
專案結構目標：
```
mjai-bridge/
├── Cargo.toml, build.rs
├── scripts/sync_proto.py
├── proto/            # 由 sync 腳本填充，只含對局相關
└── src/{main.rs, proxy.rs, converter.rs, state.rs}
```
- **M1 Proto Sync & Build**：`sync_proto.py` 從上游開源 repo 的 raw URL 只下載 `Action*.proto` / 對局相關 `Res*.proto` 到 `proto/`（**禁止**下載 login/lobby/chat proto）。`build.rs` 用 `prost-build` 編譯為 `majsoul_proto` 模組。sync 腳本需支援 pin commit/tag 以可復現。
- **M2 Converter**：
  - `fn tile_id_to_mjai(id: i32, is_red: bool) -> String`；ID 對照與 Phase 1 `TILE_NAMES` 一致：0-8=1m..9m, 9-17=1p..9p, 18-26=1s..9s, 27-30=E,S,W,N, 31-33=P,F,C；赤五輸出 `5mr/5pr/5sr`。
  - `fn convert_action(...) -> Option<MjaiEvent>`：NewRound→start_kyoku（含 bakaze/kyoku/honba/oya/dora_marker/tehais）、Tsumo→tsumo、DiscardTile→dahai(含 tsumogiri)、Chi/Pon/Kan→對應 mjai call。
  - 單元測試必須覆蓋赤五與字牌映射。
- **M3 Proxy**：`tokio-tungstenite` 本地 WS server（`ws://127.0.0.1:12121`）+ 上游 client；雙向透明轉發；攔截 binary frame，剝離幀頭（長度以開源上游實測為準，常見 2-byte 或 4-byte header，先做可配置 const），`prost::Message::decode` 後交 M2。解碼失敗 log+skip，不 panic。
- **M4 Integration**：proxy 產出 mjai NDJSON → stdout（供 Phase 1 stdin）；並讀 stdin 接收 Phase 1 響應，demo 階段僅打印 `AI 建議: ...`。**不做任何自動提交**。
- 非同步模型：tokio；網路 IO 與推理解耦（`spawn_blocking` 或獨立 task）。

## 6. 編碼規範（Coding Standards）
- 熱路徑（state update / feature extract / frame parse）禁止 Vec/HashMap 動態分配；預分配或棧上陣列。
- 錯誤：`Result<T, Box<dyn Error>>` 或自訂 error enum + `?`；proxy 層錯誤降級為 log。
- 註解只寫「為什麼」（性能取捨、不變量、上游不確定性），不寫「是什麼」。
- 每次 proto 上游更新後，`cargo build` 必須能自動重編譯；sync 腳本輸出變更摘要（新增/刪除/修改的 message）。

## 7. 驗收標準（Acceptance Criteria）
Phase 1 接手驗收（第一任務）：
1. `cargo build` 與 `cargo test` 全綠（允許為 ort rc.10 API 差異做最小 compile-fix，fix 需列成 diff 摘要回報）。
2. mock 命令 stdout 與 §3 期望字串完全一致。
Phase 2 demo 驗收：
1. `cargo build --release` 無 warning；產物為單一 binary。
2. `python scripts/sync_proto.py` 填充 `proto/` 且僅含對局 proto；`cargo build` 自動重編譯成功。
3. 以本地模擬器或回放檔餵入 proxy 時，stdout 依序打印合法 mjai NDJSON（start_kyoku/tsumo/dahai）。
4. 單事件端到端（frame 進 → mjai 出）延遲 < 100ms（demo 量測，stderr 打印計時）。

## 8. 環境與 Runbook
- Rust/Cargo >= 1.81。首次 build 需 registry 可用；之後可 `--offline`。
- mock 不需任何 native lib。real 推理：`export ORT_DYLIB_PATH=/abs/path/libonnxruntime.{dylib,so,dll}`（Windows 用 `$env:ORT_DYLIB_PATH`）。
- 輸入 fixture：`examples/draw.jsonl`（兩行 NDJSON）。

## 9. 已知風險與 TODO
- [ ] Phase 1 未經編譯驗證 → 接手即 compile-fix（最高優先）。
- [ ] 輸入/輸出 shape 與特徵語義為 placeholder；接真實 Mortal 需替換 encoder、action vocabulary、legal-action mask（**不得**以維度相符推定相容）。
- [ ] 上游 proto 與幀頭格式會變動；sync 需 pin 版本並附變更摘要。
- [ ] Session 目前每次推理重建；長跑服務應建一次重用（Phase 2 M4 時處理）。
- [ ] 模型檔案授權：僅本地研究使用，不隨 repo 分發。

## 10. 第一個任務（Immediate Action for Qwen Coder）
1. 讀 `main.rs`, `Cargo.toml`, `README.md`, `examples/draw.jsonl`。
2. 執行 `cargo build` 與 `cargo test`；若有 ort rc.10 API 差異，做**最小**修復。
3. 執行 mock 命令，比對 stdout。
4. 回報：diff 摘要 + 測試結果 + 任何與本文件 §3 契約的偏離。
5. 回報後**停止**，等待指示進入 Phase 2 M1。不要自行開始 Phase 2。