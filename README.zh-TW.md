# DSH 第二助手工具組

[English](README.md) · **繁體中文**

這套工具組替 macOS 上的 DeepSeek Harness 0.1.5-rc.2 補上幾項可以分別檢查的執行限制、記憶隔離的修正，以及一個獨立的文件轉檔工具。本專案採 MIT 授權，不會改變 DSH Desktop 本身的授權條款。

## 提供的功能

- **操作批准**：宿主原本做出的拒絕或詢問決定都會保留。此外，瀏覽器寫入、接管既有分頁，以及常見的發布與刪除命令，都會多一道確認。
- **macOS 程序限制**：沿用官方 Seatbelt provider 的檔案政策，另外替受限的 shell 程序擋下 IP 網路與 Apple Events。本機計算與工作區內的寫入照常可用。Unix socket 會保留，讓 LibreOffice 等應用程式能在本機互相通訊。
- **Engram 0.7.6 驗證**：用未修改的上游版本驗證工作區隔離與明確指定的記憶操作，並提供唯讀的升級檢查，以及真實 SQLite 遷移行為的重現。0.7.5 的舊補丁只保留作為歷史紀錄。
- **Office 預覽**：使用另外安裝的 LibreOffice 與 macOS 系統字型，把文件轉成 PDF 和逐頁 PNG，避免中文字型缺失卻沒人發現。
- **相容性修正紀錄**：包含 Verified Search 的 patch，以及一份沒有通過實機驗收的 computer-use 實驗 patch。後者不會由本套件安裝或啟用。

在同一輪對話中，Mac 輸入只要連續兩次遇到已知的介面錯誤，或累計達到 20 次輸入，工具層就會拒絕這一輪後續的輸入。觀察畫面仍然可以，但只有新的使用者輪次才會重置計數。重新觀察或改用別的輸入工具，都不能清除失敗次數。這項限制也不等於模型費用的上限。

## 安裝防護插件

需要 macOS 14 以上、Node 24、DSH 0.1.5-rc.2，以及既有的 workspace-write／ask 權限模式。請先在獨立的測試 profile 中驗證。

```sh
git clone https://github.com/f0909172434/dsh-second-agent-kit.git
cd dsh-second-agent-kit
npm ci
npm test
npm pack
mkdir -p "$HOME/.dsh/packages/dsh-second-agent-kit-0.1.4"
tar -xzf dsh-second-agent-kit-0.1.4.tgz -C "$HOME/.dsh/packages/dsh-second-agent-kit-0.1.4"
dsh plugin --profile YOUR_TEST_PROFILE add "file:$HOME/.dsh/packages/dsh-second-agent-kit-0.1.4/package"
```

套件會停用上游的 `sandbox` 入口，並加入 `second-agent-sandbox` 與 `second-agent-approval`。前者繼承上游的檔案政策，並不是改用沒有限制的 shell。如果本機已經裝了 `daily-approval`，請在確認新插件正常之後，再停用這個重複的入口。DSH 自己的批准插件要保留。

遇到不支援的沙箱執行器時，套件會明確失敗，不會偷偷改成沒有限制的執行方式。此版本只支援 macOS。Linux CI 只驗證可攜的邏輯，不代表 Linux 部署已經受到支援。不要永久啟用 `danger-full-access`，因為這個模式本來就會繞過宿主的程序限制。網路命令仍需要宿主既有的逐次提權與使用者批准。

## 記憶驗證與升級檢查

已驗證的離線設定使用原版 **Engram 0.7.6**，並關閉自動攝取與查詢改寫。啟動升級後的插件之前，請先檢查既有的記憶目錄。只要回傳非零，就停止升級：

```sh
npm run check:engram-upgrade -- --db-dir /absolute/memory-directory --workspace /absolute/workspace
npm run verify:engram
npm run verify:engram:host
npm run test:migration
```

這項檢查只讀取檔名與 Git 識別資訊，不會開啟記憶內容，也不會搬移或合併資料。記憶驗收使用真實的插件與 SQLite，透過 HonestCI 要求六項測試全部執行，不能跳過任何一項。固定的 0.7.6 基準仍然有舊資料庫歸屬不明的問題，能重現這個問題並不代表遷移是安全的。

上游的 [PR #4](https://github.com/kenz1117/dsh-engram/pull/4) 已在 2026-09-21 合併，並隨 [0.7.7](https://github.com/kenz1117/dsh-engram/releases/tag/v0.7.7) 發布，新增了 conservative 選項與遷移歸屬的 sidecar。本套件還沒有驗證或升級到這個版本。細節請看[升級指引與限制](docs/ENGRAM-UPGRADE.md)。

`scripts/patch-engram.py` 與 `tests/memory-runtime.mjs` 是 0.7.5 時期的歷史實驗，不是新版的預設安裝步驟。自動攝取政策、向量檢索品質與原生桌面介面，都還沒有納入這一輪驗收。

## 文件轉檔

從 [LibreOffice 官方網站](https://www.libreoffice.org/) 安裝應用程式，再到獨立的 Python 環境安裝 PyMuPDF：

```sh
python scripts/render-office.py input.docx new-preview-directory --png
```

支援的格式有 DOCX、XLSX、PPTX、ODT、ODS 與 ODP。工具會使用隔離的暫存 profile，保留來源文件，並拒絕覆蓋已經存在的 PDF。省略 `--png` 時不需要 PyMuPDF。`--soffice` 可以指定 LibreOffice 的執行檔。

每一頁 PNG 都要實際看過，不能只抽出文字就說版面正常。LibreOffice 的排版不保證和 Microsoft Word 完全一致，試算表的重新計算與簡報動畫也仍然要在目標應用程式中確認。

## Computer-use 與搜尋 patch

`patches/` 裡保存了這些修正、它們對應的上游 commit，以及原本的授權。computer-use patch 必須套用在記錄下來的原始碼版本上，並執行上游的 `pnpm run build`，讓 JavaScript 與原生 helper 一起重建。

**目前的 computer-use patch 沒有解決 Word 檔案面板的問題。** 它只是一份失敗實驗的紀錄，不應該當成正式修復來安裝。編譯與單元測試成功，不等於在 Mac 畫面上的操作會成功。

## 驗證與限制

`npm test` 驗證批准政策、宿主設定的組合、與上游 provider 的整合，也確認 macOS 核心確實允許工作區內的寫入、拒絕越界寫入與 TCP 連線，並允許本機的 Unix socket。

這不是所有工具共用的完整安全邊界。命令樣式比對只是輔助手段。瀏覽器、MCP、GUI 與其他插件各自有權限設定。本機 IPC 可能連到擁有額外權限的服務或代理，因此不能宣稱已經完全阻止所有對外的副作用。允許寫入的檔案，也可能被其他程序讀取。

完整的實測結果、失敗紀錄與尚未驗證的事項，請看 [VALIDATION.md](VALIDATION.md)。公開的 repository 不得包含 API 金鑰、使用者文件、對話紀錄或正式的記憶資料庫。

桌面版 0.15.6 會把本機壓縮包誤認成遺失的資料夾，並在啟動時刪除它，所以請保留上面步驟中解壓出來的目錄。

## 回復

先停止 DSH，從目標 profile 的套件與 bundles 中移除本插件，確認上游的 `sandbox` 已經恢復、而且只有一個有效的 provider，再重新啟動。如果先前停用了重複的本機批准插件，核對無誤後可以把它恢復。不要只停用新的 provider，卻讓原本的 provider 保持停用。

移除插件後，額外的 IP 網路與 Apple Events 限制也會一起消失。記憶 patch、文件技能與 provider 憑證都是獨立的設定，不會因為移除防護套件而自動還原。

## 授權

工具組採用 [MIT](LICENSE) 授權。各上游 patch 的原始授權另外存放在 `patches/`。本專案不是 DeepSeek 的官方產品，也不改變其他軟體的授權或服務條款。

[技術案例：驗證上游功能、退役重複補丁，並保留遷移限制](docs/ENGRAM-CASE-STUDY.md)
