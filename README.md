# 峻爸 AI Transcriber v3.5｜安全發佈／低誤判封裝版

> 核心轉錄功能沿用 v3.4；v3.5 主要重做 Windows 發佈方式。目標是降低自製 PyInstaller 程式被 Chrome / SmartScreen / Defender 誤判的機率，而不是規避或關閉安全檢查。

## v3.5 發佈方式

- **MSI 安裝版（首選）**：標準 Windows Installer。
- **Portable 資料夾版**：PyInstaller onedir，不使用 UPX。
- EXE 內建產品名稱、版本、公司名稱（峻爸）與 `asInvoker` manifest。
- 每次 GitHub Actions 產生 `SHA256SUMS.txt`、`BUILD_INFO.txt` 與建置來源證明。
- 支援可選 Authenticode Code Signing。若 GitHub Secrets 設定 `WINDOWS_CERT_PFX_BASE64` 與 `WINDOWS_CERT_PASSWORD`，EXE/MSI 會自動簽署並加 timestamp。
- **不再預設建置 one-file EXE**，因為 one-file 自解壓行為較容易被安全軟體判定為未知／高風險程式。

### 重要
沒有受信任的程式碼簽章憑證時，Chrome / SmartScreen 仍有可能警告；本專案不使用任何規避安全檢查的技巧，也不建議關閉整體瀏覽器或 Defender 防護。

## GitHub 建置

Workflow 路徑：`.github/workflows/build-windows-v3.5.yml`

完成後會有兩個主要 Artifacts：
1. `Junba-AI-Transcriber-v3.5-MSI-Windows-x64`
2. `Junba-AI-Transcriber-v3.5-Portable-Windows-x64`

---


## v3.5 核心目標

1. **最佳化優先**：一般使用者只需選「最佳化（推薦）」後加入音檔並開始，不必懂 Whisper 模型、OpenVINO、CUDA 或 NPU。
2. **跨電腦自適應**：依 NVIDIA CUDA、Intel GPU/NPU、Intel/AMD CPU、RAM 與本機歷史速度（RTF）選擇路徑；失敗會降級到較穩定裝置。
3. **改善舊款 Intel Iris Xe / UHD 長時間卡住**：自適應初次使用會偏向 CPU INT8，模型也會保守選 small/medium；有實測 RTF 後再依該電腦真正速度調整。
4. **長錄音自動快速切段**：最佳化開啟時，切割設為 0 代表「自動切段」。v3.5 優先採 FFmpeg 無重編碼 stream-copy；一般 M4A 不再先重編碼 AAC，因此切割應遠快於即時播放速度。只有格式不相容才改用 16 kHz mono PCM WAV 相容切割。
5. **音檔相容預檢**：加入檔案後先做 FFmpeg 解碼檢查；讀不到的 M4A/AAC/容器會自動轉為 16 kHz 單聲道 PCM WAV 再處理。OneDrive 雲端佔位檔若尚未下載，會給出明確提示。
6. **錄音核對播放器**：預設輸出 HTML + VTT。開啟 HTML 後可播放原始錄音、搜尋逐字稿、點文字跳到時間點、播放時自動反白目前段落。若搬移檔案導致音訊路徑失效，可在播放器內重新選擇音檔。
7. **混合模式最佳化**：本機 Whisper 先完成有時間軸的逐字稿，再把文字送 Gemini 整理。Gemini 失敗/繁忙不會丟掉本機結果。Word 可同時保留「Gemini 整理後全文」與「時間軸逐字稿（錄音核對用）」。
8. **立即停止一定有結果可查**：切割/轉換階段會立即終止 FFmpeg；已有辨識內容就輸出中止版逐字稿，尚未開始辨識也至少建立可開啟的中止紀錄，不再把使用者停止動作當成程式錯誤。

## 一鍵使用方式

- **最佳化（推薦）**：有 Gemini API Key 時自動使用混合模式；沒有 Key 時自動改用完全離線。模型、硬體與切段都自動。
- **完全離線穩定**：不把音訊或文字送上網路，使用本機自適應 Whisper。
- **線上 Gemini 快速**：直接使用 Gemini 音訊轉錄並自動切段。
- **進階自訂**：需要時才手動指定引擎、模型、硬體、切割分鐘數。

## 核對播放器輸出

完成後會看到例如：

- `會議_逐字稿.docx`
- `會議_逐字稿_字幕.vtt`
- `會議_逐字稿_錄音核對.html`

HTML 頁面直接用瀏覽器開啟即可。點選任一段文字會把錄音跳到該時間；播放錄音時目前段落會自動反白。

## GitHub Actions 建置

Workflow 必須位於：

`.github/workflows/build-windows-v3.5.yml`

Actions → **Build Windows EXE v3.5** → Run workflow。

建議 Intel NPU/GPU / OpenVINO 優先下載：

`Junba-AI-Transcriber-v3.5-Portable-Windows-x64`

單檔版：

`Junba-AI-Transcriber-v3.5-Single-EXE-Windows-x64`

## 注意

- 自適應不是保證每一顆 GPU/NPU 永遠不會有驅動錯誤，而是盡量在同一工作內選擇、降級並記住這台電腦較穩定/較快的路徑。
- 舊款 Intel Iris Xe / UHD 不再預設使用大型 OpenVINO Whisper 模型；先求穩定完成，再依實測效能調整。
- 原始音檔如果在 OneDrive 且只有雲端圖示，請先在檔案總管選「永遠保留在此裝置」。

## v3.5 切割與停止行為

- 切割階段主要使用 CPU + 磁碟 I/O，GPU/NPU 沒負載屬正常現象。
- 執行監看會顯示切割百分比、已耗時與 ETA。
- 第一次按「立即停止並輸出目前結果」後按鈕會停用，避免重複送出停止命令。
- 若停止時還沒有任何逐字稿，會輸出中止紀錄；若已有部分辨識，會輸出中止版 Word/TXT/字幕。
