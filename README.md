# tw-koubo-cut：用 Claude 剪中文口播短影音

不開剪輯軟體，把一支中文直式口播影片丟給 Claude，自動剪掉重講與停頓、降噪、加字幕（強調字換色）、第 0 幀標題、b-roll、音效、原創配樂，輸出 1080x1920 成品和 srt 字幕檔。

## 你需要什麼
1. Claude 付費方案（Pro 以上，依 Anthropic 支援文章；方案內容請以官網為準）
2. 在 Settings → Capabilities 打開 **Code execution and file creation**
3. 不需要 MCP，不需要終端機，不需要安裝任何軟體

## 安裝（一次就好）
1. 下載 [`tw-koubo-cut.zip`](./tw-koubo-cut.zip)
2. Claude：Settings → Capabilities → Skills → **Upload skill**，選這個 zip
3. 確認清單裡出現 `tw-koubo-cut`，開關是開的

## 使用
新開對話，附上直式口播影片（30 到 60 秒最佳），輸入：

```
剪這支。
```

進階（沒有的行刪掉）：

```
剪這支。
標題：第一行「____」，第二行「____」
強調字顏色：#FFD84A
專有名詞：Claude、____、____
不要配樂
```

也可以說「不要標題」、「先給我逐字稿確認再剪」，或附上自己有版權的音樂檔。

收到成品後，**Claude 聽不到聲音**，請用手機開聲音看、逐句核對字幕錯字，並用時間碼回覆修改，例如：

```
0:03 字卡太早，延後半秒
0:12 字幕「克勞德」改成「Claude」
```

## 預設規格
1080x1920、30 fps、AAC 48k、-14 LUFS。字幕為思源黑體 Bold（Noto Sans CJK TC Bold），白字、深色描邊、強調字黃色 #FFD84A，字幕中心位置 y=1405。

## 時間
環境安裝約 1 分鐘，分析約 25 秒，渲染約為成品長度的 3 倍。

## 檔案
- `tw-koubo-cut/SKILL.md`：skill 本體（規則加 9 支腳本）
- `tw-koubo-cut.zip`：可直接上傳的壓縮檔
