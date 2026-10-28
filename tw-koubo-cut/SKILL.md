---
name: tw-koubo-cut
description: 使用者附上中文口播影片（直式手機影片）並說「剪這支、幫我剪、去 NG」時使用：自動剪掉重講與停頓、降噪、加字幕（強調字換色）、第 0 幀標題、b-roll、音效、原創配樂，輸出 1080x1920 成品與 srt。預設不先問問題。需要開啟 Code execution and file creation。
---

# 中文口播短影音自動剪輯（繁中通用版）

把一支中文口播原片（直式、30 到 60 秒最佳）自動剪成可發布的成品：剪掉重講和停頓、降噪、字幕、標題、b-roll、音效、配樂、響度標準化。整個流程在 Claude 的雲端工作空間執行，使用者不需要安裝任何軟體。

## 開始前先檢查
- 如果沒有程式執行或檔案建立的工具可用，停止，並告訴使用者：「請到 設定 → Capabilities，打開 Code execution and file creation，再重新送一次。」不要硬做。
- 使用者沒附影片就只說「剪這支」：請他附上影片，不要猜。

## 什麼時候用
使用者附上口播影片並說「剪這支、幫我剪、去 NG」之類。**預設不要先問問題，不要要求使用者切模型、下載檔案或做額外設定**：用預設值全自動跑完，再在交付說明裡列出你做過的判斷，讓使用者否決。

## 使用者可以在請求裡自訂的項目（沒說就用預設）
| 使用者說的 | 你怎麼做 |
| --- | --- |
| 指定標題兩行文字 | 寫進 config.hook；說「不要標題」就設 null |
| 不要配樂 | config.music 設 "none" |
| 附了自己有版權的音樂檔 | config.music 設該檔路徑 |
| 指定強調字顏色（色碼） | 改 config.brand.emphasis |
| 附專有名詞、人名、品牌名詞表 | 寫字幕頁時一律用詞表的正確寫法（見下方 pages 規則） |
| 「先給我逐字稿確認再剪」 | prep.sh 跑完後，把逐字稿貼給使用者，**停下來等他回覆修正**，再繼續；這是唯一要先停的情況 |

## 成品規格
1080x1920、30 fps、bt709、AAC 48k 立體聲、-14 LUFS、真峰值 -1.5 dBFS。字幕為白字深色描邊、強調字黃色（預設：強調 #FFD84A、描邊 #1B1B1F、文字 #FFFFFF），字型為思源黑體 Bold（Noto Sans CJK TC Bold），70 px，字幕中心位置 y=1405。要換顏色就改 config.brand 的 hex。

## 流程（每次照順序做）
用 TaskCreate 建立任務清單，最後一項是驗證。

```bash
YJ=/tmp/vid_edit; W=$YJ/work_<片名>; SC=$YJ/scripts
# 0. 若 $SC 下還沒有腳本，先照本檔最後面「腳本」一節逐字寫入（用檔案建立工具，不要改內容）
bash $SC/setup_env.sh $YJ                       # 每個新工作階段一次，約 35 到 60 秒
bash $SC/prep.sh <影片路徑> $W $YJ              # 檢查規格、抽音訊、臉部與抖動分析、中文轉錄，約 25 秒
# 1. 讀轉錄結果，寫 $W/config.json（規則見下節）
python3 -I $SC/stage1_audio.py $W/src48m.wav $YJ/models/dn/dpdfnet2_48khz_hr.onnx $W/transcript_raw.json $W $W/config.json $SC
python3 -I $SC/common.py $W/edit_plan.json $W/config.json $W/storyboard.json
python3 -I $SC/stage2_mix.py $W $SC $W/config.json
python3 -I $SC/stage3_render.py $W $SC $YJ/cv4 "$(cat $YJ/fonts/font.txt)" $W/config.json
python3 -I $SC/finalize.py $W <輸出檔名>
```

之後用 Read 看 `<輸出檔名>_sheet.jpg`（接觸表）做目視檢查：字幕沒蓋到臉、標題沒壓到頭、b-roll 文字沒被切、沒有缺字方塊、強調字是黃色。finalize 的任何 `[FAIL]` 都要處理，`[WARN]` 要在交付說明裡提。

## config.json 決策規則
欄位預設值在 common.py 的 DEFAULT。必填：windows、pages。

**windows（保留哪幾次）**
- 看轉錄：同一句話（3 字以上）在 12 秒內重複出現，只留最後一次完整的；說到一半停下重來的也算重講。
- 每個 window 是保留那一次的 [開頭前約 0.3 秒, 結尾後約 0.4 秒]，不要把前一次的尾音包進來。例：前一次結束在 4.21、這一次從 4.65 開始，window 就從 4.40 起算。
- 單獨的贅詞整段（嗯、呃）不要包含。「那個、就是」常有實際語意，只有單獨出現或連續重複時才剪。
- 結尾只剩重複句時留最後一次，並在交付說明標明「這句是否為結尾我不確定」。

**pages（字幕頁）**
- 依語氣斷頁，每頁約 12 字以內；去掉 。，、 保留 ？！
- **字要用正確的寫法**：語音辨識會有誤字（人名、品牌、專有名詞最常錯）。使用者給了詞表就一律照詞表；沒給詞表，依上下文判斷明顯聽錯的字並改正，在交付說明列出「我改了哪些字」。
- 英文單字、數字寫成一個單位（AI、GPT-4 不要拆開）。
- 強調字用 [] 包起來，每頁最多 2 組，挑名詞、數字、結論。最後一頁可整句強調色當收尾。
- 對不上口白的字，common.py 會印 WARN（改正過的字一定會有，屬正常）。**WARN 的字數超過字幕總字數的 15%**，代表辨識品質差、字幕出現時間可能不準：在交付說明明確寫出，並建議使用者在安靜環境重錄或提供詞表。

**hook（第 0 幀標題）**
兩行：第一行強調色（主題），第二行白色（疑問或承諾），每行 8 字以內。只寫影片實際講到的內容，不要宣稱沒發生的事。頭頂空間不夠時腳本會自動縮小，仍放不下會印 WARNING，請改短。**prep 找不到臉時（印出「no face found」），標題空間會被保守估計而放不下，改成單行短標題或設 null，並在交付說明提。** 不要標題就設 null。

**broll（選用）**
只有口白在列舉 2 到 4 個具體名詞時才用。格式：`{"start":秒,"end":秒,"label":"標籤","chips":[["顯示文字","口白第一個字"],...]}`。start 設在列舉開頭前約 0.3 秒，end 為列舉結束，長度不超過全片約三分之一。「工具、錢、時間」有內建圖示，其他用圓點。片長不到 6 秒或沒有列舉就設 null。

**其他**
- zoom：預設循環 1.0 / 1.07 / 1.0 / 1.06（跳切節奏），一般不用改。
- music："synth"（預設，自己合成的原創配樂，無版權問題）、"none"、或使用者附上的檔案路徑（只用使用者有權使用的）。雲端環境連不到音樂網站，不要嘗試下載。
- threshold_offset_db：預設 15.5。呼吸聲被當成說話保留時調小，輕聲字被切掉時調大。
- brand 的六個欄位：emphasis（強調字）、stroke（字幕描邊）、text（字幕文字）、dark、dark2、accent（b-roll 背景與圖卡）。

## 交付
用 SendUserFile 送成品 mp4 和 srt。說明只寫：
1. 剪掉了哪些（附原片秒數）、不確定的判斷、沒驗證到的項目、你改正了哪些辨識誤字；
2. 完整字幕文字（逐頁列出），請使用者逐句核對錯字；
3. 一定要說：我聽不到成品，請用手機開聲音看，並用時間碼回饋（例如「0:03 圖卡太早」）。
不要複述步驟。

## 修改回合
使用者給時間碼意見時改 config.json：字幕文字改 pages，b-roll 時間改 broll.start/end，保留區間才需要改 windows。字幕時間會自動重算。**從 common.py 開始重跑**即可，不必重新轉錄；只有改了 windows 才要重跑 stage1。回報時只說改了哪些地方，不用重貼全部內容。

## 已知限制與踩坑
- OpenCV 5 沒有 Haar 臉部偵測（CascadeClassifier），所以 setup 另裝 4.11 放在 cv4 資料夾，只給臉部偵測用。
- 語音辨識用 SenseVoice 加 OpenCC 轉繁體，時間戳誤差約 0.1 到 0.2 秒；人名、專有名詞、聲音小或有雜音時容易聽錯，字幕必須請使用者核對。
- 句首輕聲字（例如「那」）音量低於閾值會被切掉，stage1 有補救規則；字幕缺字時回頭檢查切點。
- 臉部安全帶：頭頂以上、y 215 起放標題。頭頂太高時標題自動縮小。
- 字幕位置固定在 y=1405（畫面約 73% 處），沒有針對各平台的介面遮擋做過實測，發布前請使用者自己在平台預覽確認。
- finalize 後的音訊比畫面短約 0.07 秒，是 loudnorm 的正常現象，結尾保留的 0.15 秒靜音不受影響。
- 嘴型同步沒有自動量化。依據是：音訊與剪輯時間軸的位移實測 0 ms，且畫面與音訊由同一原片、同一 30 fps 格線切出。
- 以下情況 prep.sh 只會警告，處理方式未實測，交付說明要標註「未實測」：音畫起點偏移不為 0、HDR（需先 tone-map）、橫式（需先置中裁成 9:16）、非 30 fps（需先轉 30 fps 固定幀率）。
- 未實作：指向動作圖卡、蓋章與計數動畫。
- 渲染約為成品長度的 3 倍時間，60 秒成品約 3 到 4 分鐘；長片請放背景執行，避免 10 分鐘逾時。
- 下載的模型與字型來自 GitHub，一律用 `python3 -I` 執行腳本。

## 腳本（逐字寫入 /tmp/vid_edit/scripts/）

### setup_env.sh
```bash
#!/usr/bin/env bash
# One-time environment setup (idempotent). usage: setup_env.sh [WORK_ROOT=/tmp/vid_edit]
set -e
YJ=${1:-/tmp/vid_edit}
mkdir -p "$YJ"/models "$YJ"/fonts "$YJ"/cv4 "$YJ"/scripts

pip install --break-system-packages -q --default-timeout=200 --retries 8 \
  sherpa-onnx soundfile librosa opencc-python-reimplemented scipy pillow numpy

# OpenCV 5 dropped Haar cascades; keep a 4.x copy in its own folder just for face detection
[ -d "$YJ/cv4/cv2" ] || pip install --break-system-packages -q --target "$YJ/cv4" --no-deps opencv-python-headless==4.11.0.86

GH=https://github.com/k2-fsa/sherpa-onnx/releases/download
if [ ! -f "$YJ/models/sv/model.int8.onnx" ]; then          # SenseVoice: Chinese/English ASR with timestamps
  mkdir -p "$YJ/models/sv"
  curl -sSL --max-time 500 -o "$YJ/models/sv.tar.bz2" "$GH/asr-models/sherpa-onnx-sense-voice-zh-en-ja-ko-yue-int8-2024-07-17.tar.bz2"
  tar --no-same-owner -xjf "$YJ/models/sv.tar.bz2" -C "$YJ/models/sv" --strip-components=1
  rm -f "$YJ/models/sv.tar.bz2"
fi
mkdir -p "$YJ/models/dn" "$YJ/models/vad"
[ -f "$YJ/models/dn/dpdfnet2_48khz_hr.onnx" ] || curl -sSL --max-time 300 -o "$YJ/models/dn/dpdfnet2_48khz_hr.onnx" "$GH/speech-enhancement-models/dpdfnet2_48khz_hr.onnx"
[ -f "$YJ/models/vad/silero_vad.onnx" ] || curl -sSL --max-time 120 -o "$YJ/models/vad/silero_vad.onnx" "$GH/asr-models/silero_vad.onnx"

# Caption font: Source Han Sans / Noto Sans CJK TC Bold (思源黑體 Bold). Use the system copy if present, else download it.
# $YJ/fonts/font.txt holds "path" or "path:index" (index = the Traditional Chinese face inside a .ttc collection).
if [ ! -s "$YJ/fonts/font.txt" ]; then
  python3 - "$YJ" <<'PYEOF'
import glob, os, sys
from PIL import ImageFont
yj = sys.argv[1]
found = None
for p in glob.glob("/usr/share/fonts/**/NotoSansCJK-Bold.ttc", recursive=True):
    for i in range(10):
        try:
            fam, sty = ImageFont.truetype(p, 20, index=i).getname()
        except Exception:
            break
        if "CJK TC" in fam and sty == "Bold":
            found = f"{p}:{i}"
            break
    if found:
        break
if found:
    open(f"{yj}/fonts/font.txt", "w").write(found)
PYEOF
fi
if [ ! -s "$YJ/fonts/font.txt" ]; then
  curl -sSL --max-time 180 -o "$YJ/fonts/NotoSansTC-Bold.otf" "https://raw.githubusercontent.com/notofonts/noto-cjk/main/Sans/SubsetOTF/TC/NotoSansTC-Bold.otf"
  echo "$YJ/fonts/NotoSansTC-Bold.otf:0" > "$YJ/fonts/font.txt"
fi

# sanity check
python3 - "$YJ" <<'EOF2'
import os, sys
yj = sys.argv[1]
for f in ["models/sv/model.int8.onnx", "models/sv/tokens.txt", "models/dn/dpdfnet2_48khz_hr.onnx", "models/vad/silero_vad.onnx"]:
    p = os.path.join(yj, f)
    print(("OK   " if os.path.getsize(p) > 100000 else "BAD  "), f, os.path.getsize(p))
fa = open(os.path.join(yj, "fonts/font.txt")).read().strip()
from PIL import ImageFont
path, _, idx = fa.rpartition(":")
f = ImageFont.truetype(path, 40, index=int(idx))
print("OK   font", fa, f.getname(), "| has 繁體字:", f.getmask("臺灣繁體").getbbox() is not None)
import sherpa_onnx, soundfile, librosa, opencc, scipy, PIL
sys.path.insert(0, os.path.join(yj, "cv4"))
import cv2
print("cv2", cv2.__version__, "Haar:", hasattr(cv2, "CascadeClassifier"))
EOF2
```

### prep.sh
```bash
#!/usr/bin/env bash
# Prepare one clip: copy, inspect, extract audio, analyse face/shake, transcribe.
# usage: prep.sh <video> <workdir> [WORK_ROOT=/tmp/vid_edit]
set -e
V=$1; W=$2; YJ=${3:-/tmp/vid_edit}
mkdir -p "$W"; cp "$V" "$W/src.mov"
echo "== ffprobe =="
ffprobe -v error -show_entries stream=codec_type,codec_name,width,height,r_frame_rate,avg_frame_rate,start_time,duration,color_transfer,sample_rate:stream_side_data=rotation -of compact=p=0 "$W/src.mov"
python3 - "$W" <<'EOF'
import json, subprocess, sys
w = sys.argv[1]
s = json.loads(subprocess.run(["ffprobe", "-v", "error", "-show_entries", "stream=codec_type,start_time,color_transfer,width,height,r_frame_rate",
                               "-of", "json", f"{w}/src.mov"], capture_output=True, text=True).stdout)["streams"]
v = next(x for x in s if x["codec_type"] == "video"); a = next(x for x in s if x["codec_type"] == "audio")
off = float(a.get("start_time", 0) or 0) - float(v.get("start_time", 0) or 0)
open(f"{w}/av_off.txt", "w").write(f"{off:.4f}")
print(f"AV_OFF (audio start - video start) = {off:+.4f}s  -> saved to av_off.txt")
if v.get("color_transfer") in ("arib-std-b67", "smpte2084"):
    print("WARNING: HDR source - tone-map to SDR first (see SKILL.md)")
if int(v["width"]) > int(v["height"]):
    print("WARNING: horizontal source - crop to 9:16 around the face first (see SKILL.md)")
if v["r_frame_rate"] != "30/1":
    print("WARNING: frame rate is", v["r_frame_rate"], "- convert to 30 fps CFR first (see SKILL.md)")
EOF
ffmpeg -y -v error -i "$W/src.mov" -vn -ac 1 -ar 16000 "$W/voice16.wav"
ffmpeg -y -v error -i "$W/src.mov" -vn -ac 1 -ar 48000 "$W/src48m.wav"
echo "== face / shake =="
python3 -I "$YJ/scripts/analyze_video.py" "$W/src.mov" "$W/face.json" "$YJ/cv4"
echo "== transcript (Traditional Chinese, token@seconds) =="
python3 -I "$YJ/scripts/transcribe.py" "$W/voice16.wav" "$YJ/models/sv" "$YJ/models/vad/silero_vad.onnx" "$W/transcript_raw.json"
```

### common.py
```python
"""Shared config loading + storyboard builder (captions, hook, b-roll, SFX times)."""
import json
import re

DEFAULT = {
    "brand": {"emphasis": "FFD84A", "stroke": "1B1B1F", "text": "FFFFFF", "dark": "101215", "dark2": "23272E", "accent": "5A6170"},
    "caption": {"size": 70, "stroke": 9, "y": 1405},
    "grade": "eq=contrast=1.12:brightness=-0.01:saturation=1.12,vignette=PI/8",
    "zoom": [1.00, 1.07, 1.00, 1.06],     # cycled over kept segments (jump-cut rhythm)
    "threshold_offset_db": 15.5,          # active-speech threshold = speech p90 - this
    "windows": [],                        # [[src_start, src_end], ...] takes to keep
    "pages": [],                          # caption pages, [..] marks yellow emphasis
    "hook": None,                         # ["yellow line", "white line"] or null
    "broll": None,                        # {"start","end","label","chips":[[text,key],..]} or null
    "music": "synth",                     # "synth" | "none" | "/path/to/royalty_free.mp3"
}


def _merge(base, over):
    out = json.loads(json.dumps(base))
    for k, v in over.items():
        if isinstance(v, dict) and isinstance(out.get(k), dict):
            out[k] = _merge(out[k], v)
        else:
            out[k] = v
    return out


def load_cfg(path):
    return _merge(DEFAULT, json.load(open(path, encoding="utf-8")))


def hex_rgb(h):
    h = h.lstrip("#")
    return tuple(int(h[i:i + 2], 16) for i in (0, 2, 4))


def colors(cfg):
    return {k: hex_rgb(v) for k, v in cfg["brand"].items()}


SKIP_DISPLAY = set("，。、；：,.;:")


def parse_page(text):
    """'現在透過[AI]可以' -> [('現',False),...,('AI',True),...]; latin/digit runs stay one unit."""
    units = []
    for part in re.split(r"(\[[^\]]*\])", text):
        if not part:
            continue
        emph = part.startswith("[")
        part = part.strip("[]")
        for u in re.findall(r"[A-Za-z0-9]+|.", part):
            if u in SKIP_DISPLAY or u.isspace():
                continue
            units.append((u, emph))
    return units


def _find(toks, j, unit):
    for k in range(j, min(len(toks), j + 8)):
        t = toks[k]["ch"].lower().replace("▁", "")
        if t == unit.lower():
            return k, k
        if unit.isascii() and unit.isalnum():           # latin word split in several tokens
            acc = ""
            for m in range(k, min(len(toks), k + 6)):
                acc += toks[m]["ch"].lower().replace("▁", "")
                if acc == unit.lower():
                    return k, m
                if not unit.lower().startswith(acc):
                    break
    return None


def build_storyboard(plan_path, cfg):
    plan = json.load(open(plan_path, encoding="utf-8"))
    total, toks = plan["total"], plan["chars"]
    warnings = []

    pages, j, last_t1 = [], 0, 0.0
    for text in cfg["pages"]:
        items = []
        for unit, emph in parse_page(text):
            hit = _find(toks, j, unit)
            if hit:
                k, m = hit
                items.append({"ch": unit, "t0": toks[k]["t0"], "t1": toks[m]["t1"], "emph": emph})
                j, last_t1 = m + 1, toks[m]["t1"]
            else:
                if unit not in "？！?!":
                    warnings.append(f"cannot align {unit!r} in {text!r} (timed from previous word)")
                items.append({"ch": unit, "t0": last_t1, "t1": last_t1 + 0.1, "emph": emph})
        if not items:
            continue
        pages.append({"text": text, "items": items})
    for pg in pages:
        pg["show"] = max(0.0, pg["items"][0]["t0"] - 0.12)
    for i, pg in enumerate(pages):
        pg["hide"] = pages[i + 1]["show"] if i + 1 < len(pages) else total

    hook = None
    if cfg.get("hook"):
        hook = {"lines": cfg["hook"], "show": 0.0, "hide": pages[0]["hide"] if pages else min(total, 2.5)}

    broll, sfx = None, []
    if cfg.get("broll"):
        b = cfg["broll"]
        chips = []
        for text, key in b["chips"]:
            t = next((c["t0"] for c in toks if c["ch"] == key[0] and c["t0"] >= b["start"] - 0.3), None)
            if t is None:
                warnings.append(f"chip {text!r}: word not found, spaced evenly")
                t = b["start"] + 0.4 + 1.0 * len(chips)
            chips.append({"text": text, "t": round(t - 0.07, 3)})
        broll = {"start": b["start"], "end": b["end"], "label": b["label"], "chips": chips}
        sfx = [{"kind": "whoosh_in", "t": b["start"] - 0.05}, {"kind": "whoosh_out", "t": b["end"] - 0.12}]
        sfx += [{"kind": "pop", "t": c["t"]} for c in chips]

    z = cfg["zoom"]
    zoom = [z[i % len(z)] for i in range(len(plan["segments"]))]
    return {"total": total, "fps": plan["fps"], "av_off": plan.get("av_off", 0.0), "pages": pages, "hook": hook,
            "broll": broll, "sfx": sfx, "zoom": zoom, "segments": plan["segments"], "warnings": warnings}


if __name__ == "__main__":
    import sys
    plan_path, cfg_path, out = sys.argv[1:4]
    sb = build_storyboard(plan_path, load_cfg(cfg_path))
    json.dump(sb, open(out, "w", encoding="utf-8"), ensure_ascii=False, indent=1)
    for p in sb["pages"]:
        print(f"{p['show']:5.2f}-{p['hide']:5.2f}  {p['text']}")
    if sb["hook"]:
        print("hook", sb["hook"]["show"], "-", sb["hook"]["hide"])
    if sb["broll"]:
        print("broll", sb["broll"]["start"], "-", sb["broll"]["end"], [(c['text'], c['t']) for c in sb["broll"]["chips"]])
    print("sfx", [(s['kind'], round(s['t'], 2)) for s in sb["sfx"]])
    for w in sb["warnings"]:
        print("WARN", w)
```

### transcribe.py
```python
"""SenseVoice (Chinese) + Silero VAD transcription with token timestamps; prints Traditional Chinese.
usage: transcribe.py <voice16k.wav> <sensevoice_dir> <silero_vad.onnx> <out.json>
"""
import json
import sys

import numpy as np
import sherpa_onnx
import soundfile as sf
from opencc import OpenCC

wav_path, sv_dir, vad_path, out_path = sys.argv[1:5]
cc = OpenCC("s2twp")

audio, sr = sf.read(wav_path, dtype="float32")
assert sr == 16000, sr
if audio.ndim > 1:
    audio = audio.mean(axis=1)
audio_n = audio * (0.8 / float(np.max(np.abs(audio)) or 1.0))       # recordings are often very quiet

vcfg = sherpa_onnx.VadModelConfig()
vcfg.silero_vad.model = vad_path
vcfg.silero_vad.threshold = 0.4
vcfg.silero_vad.min_silence_duration = 0.30
vcfg.silero_vad.min_speech_duration = 0.15
vcfg.silero_vad.max_speech_duration = 12.0
vcfg.sample_rate = 16000
vad = sherpa_onnx.VoiceActivityDetector(vcfg, buffer_size_in_seconds=120)
win = vcfg.silero_vad.window_size
for i in range(0, len(audio_n), win):
    vad.accept_waveform(audio_n[i:i + win])
vad.flush()
chunks = []
while not vad.empty():
    seg = vad.front
    chunks.append((seg.start / 16000.0, np.array(seg.samples, dtype=np.float32)))
    vad.pop()

rec = sherpa_onnx.OfflineRecognizer.from_sense_voice(
    model=f"{sv_dir}/model.int8.onnx", tokens=f"{sv_dir}/tokens.txt",
    num_threads=2, use_itn=True, language="zh", debug=False)

segments = []
for start, samples in chunks:
    st = rec.create_stream()
    st.accept_waveform(16000, samples)
    rec.decode_stream(st)
    r = st.result
    ts = list(r.timestamps) if hasattr(r, "timestamps") else []
    segments.append({"start": round(start, 3), "end": round(start + len(samples) / 16000.0, 3), "text": r.text,
                     "tokens": list(r.tokens), "timestamps": [round(float(t) + start, 3) for t in ts]})
json.dump(segments, open(out_path, "w", encoding="utf-8"), ensure_ascii=False, indent=1)

for s in segments:
    print(f"\n[{s['start']:6.2f}-{s['end']:6.2f}] {cc.convert(s['text'])}")
    print("   " + " ".join(f"{cc.convert(t)}@{ts:.2f}" for t, ts in zip(s["tokens"], s["timestamps"])))
```

### analyze_video.py
```python
"""Measure shake and track the face (OpenCV 4.x Haar cascade).
usage: analyze_video.py <video> <out.json> <cv4_dir>
Face boxes are stored in 1080x1920 coordinates for every source frame (interpolated from 10 fps detection).
"""
import json
import sys

video, out_path, cv4_dir = sys.argv[1:4]
sys.path.insert(0, cv4_dir)          # OpenCV 5 removed CascadeClassifier, so use the 4.x copy in cv4_dir
import cv2
import numpy as np

cap = cv2.VideoCapture(video)
W, H = 270, 480
SCALE = 1080 / W
casc = cv2.CascadeClassifier(cv2.data.haarcascades + "haarcascade_frontalface_default.xml")

det, prev, shifts, i = {}, None, [], 0
while True:
    ok, frame = cap.read()
    if not ok:
        break
    if i % 3 == 0 or i % 5 == 0:
        small = cv2.cvtColor(cv2.resize(frame, (W, H)), cv2.COLOR_BGR2GRAY)
        if i % 5 == 0:
            f32 = np.float32(small)
            if prev is not None:
                (dx, dy), _ = cv2.phaseCorrelate(prev, f32)
                shifts.append((dx * 4.0, dy * 4.0))
            prev = f32
        if i % 3 == 0:
            faces = casc.detectMultiScale(cv2.equalizeHist(small), 1.1, 5, minSize=(40, 40))
            if len(faces):
                x, y, w, h = max(faces, key=lambda b: b[2] * b[3])
                det[i] = (x * SCALE, y * SCALE, w * SCALE, h * SCALE)
    i += 1
cap.release()

n = i
sh = np.array(shifts) if shifts else np.zeros((1, 2))
idx = sorted(det)
boxes = []
if idx:
    arr = np.array([det[k] for k in idx])
    boxes = [[float(np.interp(f, idx, arr[:, c])) for c in range(4)] for f in range(n)]
res = {"frames": n, "mean_abs_shift_px": round(float(np.mean(np.abs(sh))), 2),
       "jitter_px": round(float(np.mean(np.abs(np.diff(sh, axis=0)))) if len(sh) > 1 else 0.0, 2),
       "face_detect_rate": round(len(det) / max(1, len(range(0, n, 3))), 3), "boxes": boxes}
json.dump(res, open(out_path, "w"))
print({k: v for k, v in res.items() if k != "boxes"})
if boxes:
    b = np.array(boxes)
    print("head_top min/max:", int((b[:, 1] - 0.34 * b[:, 3]).min()), int((b[:, 1] - 0.34 * b[:, 3]).max()),
          "| chin max:", int((b[:, 1] + 1.05 * b[:, 3]).max()))
    print("(mean shift < ~2 px = steady, skip stabilisation; this clip's value is above)")
else:
    print("WARNING: no face found - cards cannot be placed safely")
```

### stage1_audio.py
```python
"""Stage 1: denoise, polish, find cuts inside the chosen takes, cut the voice, map token times.
usage: stage1_audio.py <src48m.wav> <dpdfnet.onnx> <transcript_raw.json> <workdir> <config.json> <scripts_dir>
"""
import json
import subprocess
import sys

work_args = sys.argv[1:7]
src48, dn_model, transcript_json, outdir, cfg_path, scripts = work_args
sys.path.insert(0, scripts)

import librosa
import numpy as np
import sherpa_onnx as so
import soundfile as sf
from opencc import OpenCC

from common import load_cfg

cfg = load_cfg(cfg_path)
SR, FPS = 48000, 30.0
cc = OpenCC("s2twp")
WINDOWS = [tuple(w) for w in cfg["windows"]]
assert WINDOWS, "config.windows is empty: choose which takes to keep"
try:
    AV_OFF = float(open(f"{outdir}/av_off.txt").read().strip())
except Exception:
    AV_OFF = 0.0

a, sr = sf.read(src48, dtype="float32")
assert sr == SR, sr

# ---------------------------------------------------------------- denoise (48 kHz) + gentle mic polish
dcfg = so.OfflineSpeechDenoiserConfig(model=so.OfflineSpeechDenoiserModelConfig(
    dpdfnet=so.OfflineSpeechDenoiserDpdfNetModelConfig(model=dn_model, attenuation_limit_db=0.0), num_threads=2))
res = so.OfflineSpeechDenoiser(dcfg)(a, sr)
y = np.array(res.samples, dtype=np.float32)
if res.sample_rate != SR:
    y = librosa.resample(y, orig_sr=res.sample_rate, target_sr=SR)
y = y[: len(a)] if len(y) >= len(a) else np.pad(y, (0, len(a) - len(y)))
sf.write(f"{outdir}/denoised.wav", y, SR)
POLISH = ("highpass=f=70,equalizer=f=220:t=q:w=1.4:g=-2,equalizer=f=4200:t=q:w=1.2:g=2.5,"
          "highshelf=f=9500:g=2.5,deesser=i=0.35:m=0.5:f=0.5,"
          "acompressor=threshold=-24dB:ratio=2.5:attack=8:release=90:makeup=2")
subprocess.run(["ffmpeg", "-y", "-v", "error", "-i", f"{outdir}/denoised.wav", "-af", POLISH,
                f"{outdir}/polished.wav"], check=True)
p, _ = sf.read(f"{outdir}/polished.wav", dtype="float32")


def rms_db(x, hop=0.01):
    n = int(SR * hop)
    m = len(x) // n
    d = 20 * np.log10(np.sqrt(np.mean(x[: m * n].reshape(m, n) ** 2, axis=1) + 1e-12) + 1e-9)
    return np.maximum(d, np.append(d[1:], d[-1]))          # 2-frame max filter


db = rms_db(p)
noise, speech = np.percentile(db, 10), np.percentile(db, 90)
thr = speech - cfg["threshold_offset_db"]
print(f"polished loudness p10={noise:.1f} p90={speech:.1f} dB -> active threshold {thr:.1f} dB "
      f"(tune config.threshold_offset_db if breaths are kept or soft words are cut)")

# ---------------------------------------------------------------- active regions inside the kept takes
hop = 0.01
act, runs, i = db > thr, [], 0
while i < len(act):
    if act[i]:
        j = i
        while j < len(act) and act[j]:
            j += 1
        runs.append([i * hop, j * hop])
        i = j
    else:
        i += 1


def win_of(t):
    for k, (w0, w1) in enumerate(WINDOWS):
        if w0 <= t <= w1:
            return k
    return None


runs = [r for r in runs if win_of((r[0] + r[1]) / 2) is not None]
merged = []
for r in runs:
    if merged and r[0] - merged[-1][1] < 0.30 and win_of((merged[-1][0] + merged[-1][1]) / 2) == win_of((r[0] + r[1]) / 2):
        merged[-1][1] = r[1]
    else:
        merged.append(list(r))
merged = [r for r in merged if r[1] - r[0] >= 0.06]
grid = lambda t: round(t * FPS) / FPS
segs = []
for k, (s, e) in enumerate(merged):
    segs.append([grid(max(0.0, s - 0.07)), grid(e + (0.15 if k == len(merged) - 1 else 0.08))])
for k in range(1, len(segs)):
    segs[k][0] = max(segs[k][0], segs[k - 1][1])

# soft words: a recognised token just before a cut point belongs to the next segment
allseg = json.load(open(transcript_json, encoding="utf-8"))
toks_src = []
for sg in allseg:
    for t, ts in zip(sg["tokens"], sg["timestamps"]):
        t = cc.convert(t.replace("▁", "")).strip()
        if t:
            toks_src.append((t, ts))
PUNCT = set("，。？！、：；,.?!")
for k in range(len(segs)):
    s0, prev_end = segs[k][0], (segs[k - 1][1] if k else 0.0)
    cands = [ts for t, ts in toks_src if t not in PUNCT and s0 - 0.35 <= ts < s0 - 0.02 and ts > prev_end + 0.10
             and win_of(ts) is not None]
    if cands:
        new = grid(min(cands) - 0.06)
        print(f"  rescued a soft word: segment {k} start {s0:.2f} -> {new:.2f}")
        segs[k][0] = max(new, prev_end)

# ---------------------------------------------------------------- cut the voice
FADE = int(0.008 * SR)
pieces, o0, plan = [], 0.0, []
for s, e in segs:
    x = p[int(s * SR): int(e * SR)].copy()
    f = min(FADE, len(x) // 2)
    x[:f] *= np.linspace(0, 1, f)
    x[-f:] *= np.linspace(1, 0, f)
    plan.append({"a": s, "b": e, "o0": round(o0, 4)})
    pieces.append(x)
    o0 += len(x) / SR
voice = np.concatenate(pieces)
sf.write(f"{outdir}/voice_cut.wav", voice, SR)
total = len(voice) / SR
print(f"kept {len(segs)} segments, edited length {total:.2f}s, AV_OFF {AV_OFF:+.3f}s")
for s in plan:
    print(f"  src {s['a']:.2f}-{s['b']:.2f}  -> out {s['o0']:.2f}-{s['o0'] + s['b'] - s['a']:.2f}")


def to_out(t):
    for s in plan:
        if s["a"] - 0.03 <= t <= s["b"] + 0.03:
            return s["o0"] + (t - s["a"])
    return None


kept = []
for t, ts in toks_src:
    o = to_out(ts)
    if o is not None:
        kept.append({"ch": t, "src": ts, "t0": round(o, 3), "punct": t in PUNCT})
for k, c in enumerate(kept):
    nxt = kept[k + 1]["t0"] if k + 1 < len(kept) else total
    c["t1"] = round(min(max(nxt, c["t0"] + 0.08), c["t0"] + 0.45), 3)

json.dump({"fps": FPS, "av_off": AV_OFF, "total": round(total, 3), "segments": plan, "chars": kept,
           "threshold_db": round(float(thr), 1), "windows": [list(w) for w in WINDOWS]},
          open(f"{outdir}/edit_plan.json", "w", encoding="utf-8"), ensure_ascii=False, indent=1)
print("kept text:", "".join(c["ch"] for c in kept))
```

### stage2_mix.py
```python
"""Stage 2: backing music (synthesized original, none, or a user-supplied royalty-free file) + SFX, mixed under the voice.
usage: stage2_mix.py <workdir> <scripts_dir> <config.json>
"""
import json
import subprocess
import sys

import numpy as np
import soundfile as sf
from scipy.signal import butter, lfilter

work, scripts, cfg_path = sys.argv[1:4]
sys.path.insert(0, scripts)
from common import load_cfg

cfg = load_cfg(cfg_path)
SR = 48000
rng = np.random.default_rng(11)
sb = json.load(open(f"{work}/storyboard.json", encoding="utf-8"))
voice, _ = sf.read(f"{work}/voice_cut.wav", dtype="float32")
L = len(voice)
TOTAL = L / SR


def mtof(m):
    return 440.0 * 2 ** ((m - 69) / 12)


def tt(dur):
    return np.arange(int(dur * SR)) / SR


def place(buf, x, t, gain=1.0, pan=0.0):
    i = int(t * SR)
    if i >= len(buf):
        return
    x = x[: len(buf) - i]
    buf[i:i + len(x), 0] += x * gain * (1 - max(0, pan))
    buf[i:i + len(x), 1] += x * gain * (1 + min(0, pan))


# ------------------------------------------------------------------ original backing track (C-G-Am-F, 112 BPM)
def pluck(f, dur=0.5):
    t = tt(dur)
    return (np.sin(2 * np.pi * f * t) + 0.35 * np.sin(4 * np.pi * f * t) + 0.15 * np.sin(6 * np.pi * f * t)) \
        * np.exp(-t * 7.5) * np.minimum(1, t / 0.004)


def bass(f, dur=0.5):
    t = tt(dur)
    return (np.sin(2 * np.pi * f * t) + 0.25 * np.sin(4 * np.pi * f * t)) * np.exp(-t * 3.2) * np.minimum(1, t / 0.01)


def kick():
    t = tt(0.28)
    return np.sin(2 * np.pi * np.cumsum(48 + 110 * np.exp(-t * 32)) / SR) * np.exp(-t * 9)


def hat():
    t = tt(0.06)
    return np.diff(rng.standard_normal(len(t) + 1)) * np.exp(-t * 70)


def clap():
    t = tt(0.16)
    b, a = butter(2, [1200 / (SR / 2), 4500 / (SR / 2)], btype="band")
    return lfilter(b, a, rng.standard_normal(len(t))) * (np.exp(-t * 28) + 0.6 * np.exp(-((t - 0.012) ** 2) * 9000))


def pad(freqs, dur):
    t = tt(dur)
    x = sum(np.sin(2 * np.pi * f * t) + 0.5 * np.sin(2 * np.pi * f * 1.004 * t) for f in freqs)
    return x * np.minimum(1, t / 0.25) * np.minimum(1, (dur - t) / 0.3)


def synth_music():
    beat = 60 / 112
    bars = int(np.ceil((TOTAL + 1.0) / (beat * 4)))
    mus = np.zeros((int((bars * beat * 4 + 0.6) * SR), 2), np.float32)
    CH = [([60, 64, 67, 72], 36), ([59, 62, 67, 71], 43), ([57, 60, 64, 69], 45), ([57, 60, 65, 69], 41)]
    ARP = [0, 1, 2, 3, 2, 1, 2, 1]
    for b in range(bars):
        notes, root = CH[b % 4]
        t0 = b * beat * 4
        place(mus, pad([mtof(m - 12) for m in notes[:3]], beat * 4), t0, 0.10)
        for k in range(8):
            place(mus, pluck(mtof(notes[ARP[k]] + 12)), t0 + k * beat / 2, 0.30, pan=0.35 if k % 2 else -0.35)
        for k in range(4):
            place(mus, bass(mtof(root), beat * 0.9), t0 + k * beat, 0.55)
        for k in (0, 2):
            place(mus, kick(), t0 + k * beat, 0.75)
        for k in (1, 3):
            place(mus, clap(), t0 + k * beat, 0.22)
        for k in range(8):
            place(mus, hat(), t0 + k * beat / 2, 0.10 if k % 2 == 0 else 0.06)
    d = int(beat * 0.75 * SR)
    echo = np.zeros_like(mus)
    echo[d:, 0], echo[d:, 1] = mus[:-d, 1] * 0.25, mus[:-d, 0] * 0.25
    mus = mus + echo
    b, a = butter(2, 9500 / (SR / 2))
    mus = lfilter(b, a, mus, axis=0).astype(np.float32)
    return mus * (0.9 / np.max(np.abs(mus)))


mode = cfg["music"]
if mode == "none":
    mus = np.zeros((L, 2), np.float32)
else:
    if mode == "synth":
        sf.write(f"{work}/music_raw.wav", synth_music(), SR)
    else:                                              # user-supplied royalty-free track
        subprocess.run(["ffmpeg", "-y", "-v", "error", "-i", mode, "-ac", "2", "-ar", str(SR), f"{work}/music_raw.wav"],
                       check=True)
    subprocess.run(["ffmpeg", "-y", "-v", "error", "-i", f"{work}/music_raw.wav", "-af",
                    "equalizer=f=2500:t=q:w=1.2:g=-5,equalizer=f=1000:t=q:w=1.0:g=-2", f"{work}/music_eq.wav"], check=True)
    mus, _ = sf.read(f"{work}/music_eq.wav", dtype="float32")
    if mode != "synth":                                # skip a silent intro, loop if too short
        lead = np.argmax(np.abs(mus).max(axis=1) > 0.01)
        mus = mus[lead:]
        if len(mus) < L:
            mus = np.tile(mus, (int(np.ceil(L / len(mus))), 1))


# ------------------------------------------------------------------ SFX
def whoosh(dur=0.42, rising=True):
    t = tt(dur)
    x = rng.standard_normal(len(t))
    a = np.linspace(0.02, 0.45, len(t)) if rising else np.linspace(0.45, 0.02, len(t))
    y = np.zeros_like(x)
    for i in range(1, len(x)):
        y[i] = y[i - 1] + a[i] * (x[i] - y[i - 1])
    slow = np.zeros_like(x)
    for i in range(1, len(x)):
        slow[i] = slow[i - 1] + 0.01 * (y[i] - slow[i - 1])
    return (y - slow) * np.sin(np.pi * np.clip(t / dur, 0, 1)) ** 1.5


def pop():
    t = tt(0.14)
    return np.sin(2 * np.pi * np.cumsum(300 + 380 * np.exp(-t * 40)) / SR) * np.exp(-t * 28) \
        + rng.standard_normal(len(t)) * np.exp(-t * 400) * 0.25


bus = np.zeros((L, 2), np.float32)
for e in sb["sfx"]:
    if e["kind"] == "whoosh_in":
        place(bus, whoosh(0.42, True), max(0, e["t"] - 0.1), 1.0)
    elif e["kind"] == "whoosh_out":
        place(bus, whoosh(0.36, False), e["t"], 0.8)
    elif e["kind"] == "pop":
        place(bus, pop(), e["t"], 1.0)


# ------------------------------------------------------------------ levels
def smooth(x, win):
    return np.convolve(x, np.ones(win) / win, mode="same")


mus = mus[:L] if len(mus) >= L else np.pad(mus, ((0, L - len(mus)), (0, 0)))
hop = int(0.01 * SR)
m = L // hop
vr = np.sqrt(np.mean(voice[: m * hop].reshape(m, hop) ** 2, axis=1) + 1e-12)
env50, env400 = smooth(vr, 5), smooth(vr, 40)
voice_on = env400 > 0.10 * env400.max()
speech_db = 20 * np.log10(float(np.sqrt(np.mean(vr[voice_on] ** 2))))
voice_peak = float(np.max(np.abs(voice)))

env_s = np.pad(np.repeat(env50 / env50.max(), hop)[:L], (0, max(0, L - m * hop)), mode="edge")
bus = bus * (1 - 0.72 * env_s)[:, None]
if np.max(np.abs(bus)) > 0:
    bus *= (0.18 * voice_peak) / np.max(np.abs(bus))                # SFX peak ~ 18 % of the voice peak

musg = np.zeros_like(mus)
if np.max(np.abs(mus)) > 0:
    mr = np.sqrt(np.mean(((mus[: m * hop, 0] + mus[: m * hop, 1]) / 2).reshape(m, hop) ** 2, axis=1) + 1e-12)
    m_db = 20 * np.log10(np.sqrt(smooth(mr ** 2, 40)) + 1e-9)
    target = np.where(voice_on, speech_db - 24.0, speech_db - 19.0)    # ride the level, never one flat gain
    gain = np.repeat(10 ** (smooth(target - m_db, 30) / 20), hop)
    gain = np.pad(gain, (0, L - len(gain)), mode="edge")
    musg = mus * gain[:, None]
    w = int(0.1 * SR)
    cum = np.cumsum(np.pad((musg[:, 0] ** 2 + musg[:, 1] ** 2) / 2, (1, 0)))
    r100 = np.pad(np.sqrt((cum[w:] - cum[:-w]) / w), (0, L - (len(cum) - w)), mode="edge")[:L]
    musg = musg / smooth(np.maximum(r100 / 10 ** ((speech_db - 16.0) / 20), 1.0), 2400)[:, None]   # hard ceiling
    fi = np.minimum(1, np.arange(L) / (0.5 * SR))
    fo = np.minimum(1, (L - np.arange(L)) / (1.5 * SR)) ** 2
    musg = musg * (fi * fo)[:, None]

sf.write(f"{work}/mix_pre.wav", np.stack([voice, voice], axis=1) + bus + musg, SR)

print(f"speaking level {speech_db:.1f} dBFS rms | voice peak {20 * np.log10(voice_peak):.1f} dBFS")
if np.max(np.abs(bus)) > 0:
    print(f"SFX peak / voice peak = {np.max(np.abs(bus)) / voice_peak:.2f}   (target ~0.18)")
blk = int(3 * SR)
for i in range(0, L - SR, blk):
    seg = musg[i:i + blk]
    mdb = 20 * np.log10(np.sqrt(np.mean(((seg[:, 0] + seg[:, 1]) / 2) ** 2)) + 1e-9)
    print(f"  {i / SR:4.1f}-{min(L, i + blk) / SR:4.1f} s: music {speech_db - mdb:5.1f} dB below speech (aim >= ~20)")
```

### stage3_render.py
```python
"""Stage 3: render the picture (jump-cut zoom, grade, captions, hook headline, b-roll) to video_noaudio.mp4.
usage: stage3_render.py <workdir> <scripts_dir> <cv4_dir> <font.ttf> <config.json>
"""
import json
import math
import subprocess
import sys

work, scripts, cv4, FONT_ARG, cfg_path = sys.argv[1:6]
# FONT_ARG is "path" or "path:index" (index picks the Traditional Chinese face inside a .ttc collection)
FONT_PATH, _, _fi = FONT_ARG.rpartition(":") if FONT_ARG.rpartition(":")[0] and FONT_ARG.rpartition(":")[2].isdigit() else (FONT_ARG, "", "0")
FONT_INDEX = int(_fi)
sys.path.insert(0, cv4)
sys.path.insert(0, scripts)
import cv2
import numpy as np
from PIL import Image, ImageDraw, ImageFont

from common import colors, load_cfg

cfg = load_cfg(cfg_path)
C = colors(cfg)
YELLOW, STROKE, WHITE, DARK, DARK2, ACCENT = C["emphasis"], C["stroke"], C["text"], C["dark"], C["dark2"], C["accent"]
W, H, FPS = 1080, 1920, 30
sb = json.load(open(f"{work}/storyboard.json", encoding="utf-8"))
face = json.load(open(f"{work}/face.json"))
boxes = np.array(face["boxes"]) if face["boxes"] else np.array([[400.0, 520.0, 440.0, 440.0]])
segs = sb["segments"]
AV = float(sb.get("av_off", 0.0))
NFR = int(round(sb["total"] * FPS))
CAP_SIZE, CAP_SW, CAP_Y, GAP = cfg["caption"]["size"], cfg["caption"]["stroke"], cfg["caption"]["y"], 4


# ------------------------------------------------------------------ sprite helpers (premultiplied RGBA uint8)
def font(size):
    return ImageFont.truetype(FONT_PATH, size, index=FONT_INDEX)


def premult(img):
    a = np.array(img).astype(np.float32)
    a[..., :3] *= a[..., 3:4] / 255.0
    return np.clip(a + 0.5, 0, 255).astype(np.uint8)


def text_layers(text, size, fill, sw=8, stroke=STROKE):
    f = font(size)
    asc, desc = f.getmetrics()
    pad = sw + 6
    w = int(math.ceil(f.getlength(text))) + 2 * pad
    h = asc + desc + 2 * pad
    st = Image.new("RGBA", (w, h), (0, 0, 0, 0))
    ImageDraw.Draw(st).text((pad, pad), text, font=f, fill=stroke + (255,), stroke_width=sw, stroke_fill=stroke + (255,))
    fl = Image.new("RGBA", (w, h), (0, 0, 0, 0))
    # Noto Sans CJK TC Bold is already heavy, so the fill layer gets no extra stroke
    ImageDraw.Draw(fl).text((pad, pad), text, font=f, fill=fill + (255,))
    return st, fl, pad


def blit(frame, spr, cx, cy, scale=1.0, alpha=1.0):
    if alpha <= 0:
        return
    if scale != 1.0:
        h0, w0 = spr.shape[:2]
        spr = cv2.resize(spr, (max(1, int(round(w0 * scale))), max(1, int(round(h0 * scale)))),
                         interpolation=cv2.INTER_AREA if scale < 1 else cv2.INTER_LINEAR)
    h, w = spr.shape[:2]
    x0, y0 = int(round(cx - w / 2)), int(round(cy - h / 2))
    xa, ya, xb, yb = max(0, x0), max(0, y0), min(W, x0 + w), min(H, y0 + h)
    if xa >= xb or ya >= yb:
        return
    s = spr[ya - y0:yb - y0, xa - x0:xb - x0].astype(np.float32)
    reg = frame[ya:yb, xa:xb].astype(np.float32)
    a = s[..., 3:4] / 255.0 * alpha
    frame[ya:yb, xa:xb] = np.clip(reg * (1 - a) + s[..., :3] * alpha + 0.5, 0, 255).astype(np.uint8)


def ease_out_back(x):
    x = min(max(x, 0.0), 1.0)
    return 1 + 2.70158 * (x - 1) ** 3 + 1.70158 * (x - 1) ** 2


# ------------------------------------------------------------------ captions
pages = []
for pg in sb["pages"]:
    toks, x = [], 0.0
    for it in pg["items"]:
        st, fl, pad = text_layers(it["ch"], CAP_SIZE, YELLOW if it["emph"] else WHITE, CAP_SW)
        adv = font(CAP_SIZE).getlength(it["ch"]) + GAP
        toks.append({"st": premult(st), "fl": premult(fl), "x": x, "adv": adv, "emph": it["emph"], "t0": it["t0"]})
        x += adv
    line_w = x - GAP
    pages.append({"toks": toks, "line_w": line_w, "k": min(1.0, 940.0 / line_w), "show": pg["show"], "hide": pg["hide"]})


def draw_caption(frame, pg, t):
    if not (pg["show"] <= t < pg["hide"]):
        return
    e = min(1.0, (t - pg["show"]) / 0.18)
    ease = 1 - (1 - e) ** 3
    fade = ease
    if pg["hide"] - t < 0.20 and pg is pages[-1]:
        fade *= max(0.0, (pg["hide"] - t) / 0.20)
    k = pg["k"]
    x_start = W / 2 - pg["line_w"] * k / 2
    layers = []
    for tk in pg["toks"]:
        s = k
        if tk["emph"]:
            d = (t - (tk["t0"] - 0.03)) / 0.26
            if 0 <= d <= 1:
                s = k * (1 + 0.16 * math.sin(math.pi * d))
        layers.append((tk, x_start + (tk["x"] + tk["adv"] / 2 - GAP / 2) * k, s))
    cy = CAP_Y + 18 * (1 - ease)
    for tk, cx, s in layers:
        blit(frame, tk["st"], cx, cy, s, fade)
    for tk, cx, s in layers:
        blit(frame, tk["fl"], cx, cy, s, fade)


# ------------------------------------------------------------------ hook headline (fully visible at frame zero)
def seg_frames(s):
    return int(round((s["a"] + AV) * FPS)), int(round((s["b"] + AV) * FPS))


hook_spr = None
if sb["hook"]:
    f0, f1 = seg_frames(segs[0])
    bb = boxes[min(f0, len(boxes) - 1):min(f1, len(boxes))]
    head_top_min = float((bb[:, 1] - 0.34 * bb[:, 3]).min())
    band_top, band_bottom = 215, int(head_top_min - 45)
    l1, l2 = sb["hook"]["lines"]
    for s1, s2 in ((76, 68), (70, 62), (64, 56), (58, 50)):
        a_st, a_fl, _ = text_layers(l1, s1, YELLOW, 8)
        b_st, b_fl, _ = text_layers(l2, s2, WHITE, 8)
        cw, gap = max(a_st.size[0], b_st.size[0]), -6
        ch = a_st.size[1] + b_st.size[1] + gap
        if ch <= band_bottom - band_top:
            break
    cs, cf = Image.new("RGBA", (cw, ch), (0, 0, 0, 0)), Image.new("RGBA", (cw, ch), (0, 0, 0, 0))
    ax, bx, by = (cw - a_st.size[0]) // 2, (cw - b_st.size[0]) // 2, a_st.size[1] + gap
    for cv_, sa, sb_ in ((cs, a_st, b_st), (cf, a_fl, b_fl)):
        cv_.alpha_composite(sa, (ax, 0))
        cv_.alpha_composite(sb_, (bx, by))
    hook_spr = premult(Image.alpha_composite(cs, cf))
    hook_cy = band_top + ch / 2 + max(0, (band_bottom - band_top - ch) / 2)
    print(f"hook band y{band_top}-{band_bottom} (head top min {head_top_min:.0f}), sizes ({s1},{s2}), block h {ch}")
    if ch > band_bottom - band_top:
        print("WARNING: hook does not fit above the head even at the smallest size; shorten the text")


def draw_hook(frame, t):
    h = sb["hook"]
    if hook_spr is None or t < h["show"] or t >= h["hide"]:
        return
    blit(frame, hook_spr, W / 2, hook_cy, 1.0, min(1.0, (h["hide"] - t) / 0.15))


# ------------------------------------------------------------------ b-roll (optional)
BR = sb["broll"]


def make_bg():
    yy, xx = np.mgrid[0:H, 0:W].astype(np.float32)
    g = (yy / H)[..., None]
    base = np.array(DARK, np.float32) * (1 - g) + np.array(DARK2, np.float32) * 0.9 * g
    d = np.sqrt(((xx - 540) / 760) ** 2 + ((yy - 800) / 760) ** 2)
    base = base + (np.clip(1 - d, 0, 1)[..., None] ** 1.8) * (np.array(ACCENT, np.float32) * 0.55)
    d2 = np.sqrt(((xx - 540) / 520) ** 2 + ((yy - 330) / 200) ** 2)
    base = base + (np.clip(1 - d2, 0, 1)[..., None] ** 2) * (np.array(YELLOW, np.float32) * 0.10)
    return np.clip(base, 0, 255).astype(np.uint8)


def icon(kind, size=120):
    S = 4
    img = Image.new("RGBA", (size * S, size * S), (0, 0, 0, 0))
    d = ImageDraw.Draw(img)
    c, Y = size * S / 2, YELLOW + (255,)
    if kind == "gear":
        for i in range(8):
            ang = i * math.pi / 4
            ux, uy, vx, vy = math.cos(ang), math.sin(ang), -math.sin(ang), math.cos(ang)
            r0, r1, hw = 30 * S, 49 * S, 9.5 * S
            d.polygon([(c + ux * r0 + vx * hw, c + uy * r0 + vy * hw), (c + ux * r1 + vx * hw * .8, c + uy * r1 + vy * hw * .8),
                       (c + ux * r1 - vx * hw * .8, c + uy * r1 - vy * hw * .8), (c + ux * r0 - vx * hw, c + uy * r0 - vy * hw)], fill=Y)
        d.ellipse([c - 38 * S, c - 38 * S, c + 38 * S, c + 38 * S], fill=Y)
        d.ellipse([c - 16 * S, c - 16 * S, c + 16 * S, c + 16 * S], fill=(0, 0, 0, 0))
    elif kind == "coin":
        d.ellipse([c - 50 * S, c - 50 * S, c + 50 * S, c + 50 * S], fill=Y)
        d.ellipse([c - 39 * S, c - 39 * S, c + 39 * S, c + 39 * S], outline=(0xC9, 0xB4, 0x45, 255), width=4 * S)
        d.text((c, c), "$", font=ImageFont.truetype(FONT_PATH, 58 * S, index=FONT_INDEX), fill=DARK2 + (255,), anchor="mm")
    elif kind == "clock":
        d.ellipse([c - 48 * S, c - 48 * S, c + 48 * S, c + 48 * S], outline=Y, width=9 * S)
        d.line([(c, c), (c, c - 28 * S)], fill=Y, width=8 * S)
        d.line([(c, c), (c + 22 * S, c + 6 * S)], fill=Y, width=8 * S)
        d.ellipse([c - 6 * S, c - 6 * S, c + 6 * S, c + 6 * S], fill=Y)
    else:                                                    # plain dot for any other chip
        d.ellipse([c - 40 * S, c - 40 * S, c + 40 * S, c + 40 * S], fill=Y)
    return img.resize((size, size), Image.LANCZOS)


ICON_OF = {"工具": "gear", "錢": "coin", "費用": "coin", "價格": "coin", "時間": "clock", "時數": "clock"}


def make_chip(text):
    cw, ch = 800, 200
    img = Image.new("RGBA", (cw, ch), (0, 0, 0, 0))
    d = ImageDraw.Draw(img)
    d.rounded_rectangle([4, 4, cw - 4, ch - 4], radius=56, fill=DARK2 + (238,), outline=ACCENT + (255,), width=6)
    img.alpha_composite(icon(ICON_OF.get(text, "dot"), 120), (46, 40))
    st, fl, _ = text_layers(text, 96, WHITE, 8)
    ty = (ch - st.size[1]) // 2
    img.alpha_composite(st, (214, ty))
    img.alpha_composite(fl, (214, ty))
    return premult(img)


if BR:
    BG = make_bg()
    chip_spr = [make_chip(c["text"]) for c in BR["chips"]]
    chip_y = [640, 900, 1160, 1420][:len(chip_spr)]
    l_st, l_fl, _ = text_layers(BR["label"], 66, YELLOW, 8)
    label_spr = premult(Image.alpha_composite(l_st, l_fl))


def draw_broll(frame, t):
    if not BR or not (BR["start"] <= t < BR["end"]):
        return frame
    a = min(1.0, (t - BR["start"]) / 0.10) * min(1.0, (BR["end"] - t) / 0.10)
    b = BG.copy()
    le = min(1.0, (t - BR["start"]) / 0.25)
    blit(b, label_spr, W / 2, 330 + 14 * (1 - le), 1.0, le)
    for spr, c, y in zip(chip_spr, BR["chips"], chip_y):
        d = t - c["t"]
        if d >= 0:
            blit(b, spr, W / 2 - 40 * (1 - min(1.0, d / 0.2)), y, 0.86 + 0.14 * ease_out_back(d / 0.24), min(1.0, d / 0.10))
    return (frame.astype(np.float32) * (1 - a) + b.astype(np.float32) * a + 0.5).astype(np.uint8)


# ------------------------------------------------------------------ per-segment zoom about the face centre
seg_info, k0 = [], 0
for i, s in enumerate(segs):
    f0, f1 = seg_frames(s)
    bx = boxes[min(f0, len(boxes) - 1):max(min(f0, len(boxes) - 1) + 1, min(f1, len(boxes)))]
    seg_info.append({"f0": f0, "f1": f1, "n": f1 - f0, "k0": k0, "z": sb["zoom"][i],
                     "cx": float((bx[:, 0] + bx[:, 2] / 2).mean()), "cy": float((bx[:, 1] + bx[:, 3] / 2).mean())})
    k0 += f1 - f0
assert k0 == NFR, (k0, NFR)

sel = "+".join(f"between(n\\,{s['f0']}\\,{s['f1'] - 1})" for s in seg_info)
dec = subprocess.Popen(
    ["ffmpeg", "-v", "error", "-i", f"{work}/src.mov", "-vf",
     f"select='{sel}',setpts=N/({FPS}*TB),{cfg['grade']},format=rgb24", "-fps_mode", "passthrough",
     "-f", "rawvideo", "-pix_fmt", "rgb24", "pipe:1"], stdout=subprocess.PIPE)
enc = subprocess.Popen(
    ["ffmpeg", "-y", "-v", "error", "-f", "rawvideo", "-pix_fmt", "rgb24", "-s", f"{W}x{H}", "-r", str(FPS), "-i", "-",
     "-vf", "scale=out_color_matrix=bt709:out_range=tv,format=yuv420p", "-c:v", "libx264", "-preset", "medium", "-crf", "17",
     "-colorspace", "bt709", "-color_primaries", "bt709", "-color_trc", "bt709", "-movflags", "+faststart",
     f"{work}/video_noaudio.mp4"], stdin=subprocess.PIPE)

fsz, k = W * H * 3, 0
while k < NFR:
    buf = dec.stdout.read(fsz)
    if len(buf) < fsz:
        raise SystemExit(f"decoder ended early at frame {k}/{NFR}")
    frame = np.frombuffer(buf, np.uint8).reshape(H, W, 3).copy()
    t = k / FPS
    s = next(x for x in seg_info if x["k0"] <= k < x["k0"] + x["n"])
    if s["z"] != 1.0:
        z = s["z"]
        M = np.float32([[z, 0, (1 - z) * s["cx"]], [0, z, (1 - z) * s["cy"]]])
        frame = cv2.warpAffine(frame, M, (W, H), flags=cv2.INTER_LINEAR, borderMode=cv2.BORDER_REFLECT)
    frame = draw_broll(frame, t)
    draw_hook(frame, t)
    for pg in pages:
        draw_caption(frame, pg, t)
    enc.stdin.write(frame.tobytes())
    k += 1
enc.stdin.close()
enc.wait()
dec.stdout.close()
dec.wait()
print("rendered", k, "frames")
```

### finalize.py
```python
"""Finalize: two-pass loudness, mux, captions.srt, contact sheet and automatic QA.
usage: finalize.py <workdir> <output_basename>
"""
import json
import re
import subprocess
import sys

import numpy as np
import soundfile as sf
from scipy.signal import fftconvolve

work, name = sys.argv[1:3]
sb = json.load(open(f"{work}/storyboard.json", encoding="utf-8"))
total = sb["total"]
SR = 48000
out = f"{work}/{name}.mp4"


def run(cmd, **kw):
    return subprocess.run(cmd, capture_output=True, text=True, **kw)


# ---- loudness: measure, then apply (target -14 LUFS, -1.5 dBTP); apad keeps the length equal to the picture
r = run(["ffmpeg", "-hide_banner", "-i", f"{work}/mix_pre.wav", "-af",
         "loudnorm=I=-14:TP=-1.5:LRA=11:print_format=json", "-f", "null", "-"])
m = json.loads(re.search(r"\{.*?\}", r.stderr, re.S).group(0))
af = (f"loudnorm=I=-14:TP=-1.5:LRA=11:measured_I={m['input_i']}:measured_TP={m['input_tp']}:measured_LRA={m['input_lra']}"
      f":measured_thresh={m['input_thresh']}:offset={m['target_offset']}:linear=true,apad=whole_dur={total}")
subprocess.run(["ffmpeg", "-y", "-v", "error", "-i", f"{work}/video_noaudio.mp4", "-i", f"{work}/mix_pre.wav",
                "-map", "0:v", "-map", "1:a", "-c:v", "copy", "-af", af, "-c:a", "aac", "-b:a", "160k", "-ac", "2",
                "-ar", "48000", "-t", f"{total}", "-movflags", "+faststart", out], check=True)

# ---- captions.srt
def ts(t):
    ms = int(round(t * 1000))
    h, ms = divmod(ms, 3600000)
    mi, ms = divmod(ms, 60000)
    s, ms = divmod(ms, 1000)
    return f"{h:02d}:{mi:02d}:{s:02d},{ms:03d}"


lines = []
for i, p in enumerate(sb["pages"], 1):
    lines += [str(i), f"{ts(p['show'])} --> {ts(p['hide'])}", p["text"].replace("[", "").replace("]", ""), ""]
open(f"{work}/{name}.srt", "w", encoding="utf-8").write("\n".join(lines))

# ---- contact sheet for the visual check (look at it before delivering)
subprocess.run(["ffmpeg", "-y", "-v", "error", "-i", out, "-vf", f"fps=12/{total},scale=270:-1,tile=6x2",
                "-frames:v", "1", f"{work}/{name}_sheet.jpg"], check=True)

# ---- QA
ok = True


def check(label, cond, detail):
    global ok
    ok &= bool(cond)
    print(f"[{'PASS' if cond else 'FAIL'}] {label}: {detail}")


pr = run(["ffprobe", "-v", "error", "-show_entries", "stream=codec_type,width,height,r_frame_rate,color_space,sample_rate,channels",
          "-of", "json", out])
streams = json.loads(pr.stdout)["streams"]
v = next(s for s in streams if s["codec_type"] == "video")
a = next(s for s in streams if s["codec_type"] == "audio")
check("spec", (v["width"], v["height"], v["r_frame_rate"]) == (1080, 1920, "30/1") and v.get("color_space") == "bt709"
      and a["sample_rate"] == "48000" and a["channels"] == 2,
      f"{v['width']}x{v['height']} {v['r_frame_rate']} {v.get('color_space')} | audio {a['sample_rate']} Hz x{a['channels']}")

e = run(["ffmpeg", "-hide_banner", "-i", out, "-af", "ebur128=peak=true", "-f", "null", "-"]).stderr
I = float(re.findall(r"I:\s+(-?\d+\.\d)\s+LUFS", e)[-1])
TP = float(re.findall(r"Peak:\s+(-?\d+\.\d)\s+dBFS", e)[-1])
check("loudness", abs(I + 14) <= 1.0 and TP <= -1.0, f"{I} LUFS, true peak {TP} dBFS (target -14 / <= -1.5)")

dec = np.frombuffer(subprocess.run(["ffmpeg", "-v", "error", "-i", out, "-vn", "-ac", "1", "-ar", "48000", "-f", "f32le", "-"],
                                   capture_output=True).stdout, np.float32)
ref, _ = sf.read(f"{work}/voice_cut.wav", dtype="float32")
n = min(len(ref), len(dec))
x, y = ref[:n] - ref[:n].mean(), dec[:n] - dec[:n].mean()
lag_ms = 1000.0 * (int(np.argmax(fftconvolve(y, x[::-1], mode="full"))) - (n - 1)) / SR
check("audio/picture offset", abs(lag_ms) <= 10, f"{lag_ms:+.1f} ms between delivered audio and the edit timeline")
check("length", abs(len(dec) / SR - total) < 0.12, f"audio {len(dec) / SR:.3f}s vs picture {total:.3f}s")

env = np.sqrt(np.mean(dec[: (len(dec) // 1600) * 1600].reshape(-1, 1600) ** 2, axis=1))
last = int(np.where(env > 0.1 * env.max())[0].max())
check("tail", total - (last + 1) / 30 < 0.5, f"video ends {total - (last + 1) / 30:.2f}s after the last speech")
for w in sb.get("warnings", []):
    print("[WARN]", w)
print("ALL PASS" if ok else "SOME CHECKS FAILED")
print(f"output: {out}\nsrt: {work}/{name}.srt\nsheet: {work}/{name}_sheet.jpg")
```

## 範例 config.json
```json
{
  "windows": [[4.15, 10.20], [10.90, 14.45], [14.75, 18.30]],
  "pages": ["大家好，今天教你用[克勞德]剪影片", "不用開[剪輯軟體]", "只要[一句話]", "第一，訂閱[專業版]", "第二，打開[程式碼執行]", "第三，上傳[技能]", "然後說[剪這支]"],
  "hook": ["用Claude剪片", "不開剪輯軟體"],
  "broll": null
}
```
