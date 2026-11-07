# video/ 版本清单 —— 当前发布版 **v3**（2026-11-07）

## 1 文件（远端 `video/` 与本地同名）

| 文件 | 字节 | SHA-256 | 源 |
|---|---|---|---|
| plugin-intro-16x9.mp4 | 16219820 | AC84767F0922130464616FBFAE90A38E59AAB0CBBDE85F7F35165B917B8203FB | plugin-intro-v3-16x9.mp4 |
| plugin-intro-9x16.mp4 | 14674590 | 166107AD89D0C934C935194602A6489077888F24BD01E4F91DF715030C33B0D4 | plugin-intro-v3-9x16.mp4 |
| plugin-intro-16x9-subbed.mp4 | 15905630 | A71C268953FA07BDB4CC6EB866B4D94B7EDA15E9464CBBFCF45B1F748DE9414E | v3 16:9 叠本文 srt（本仓库现场生成） |
| plugin-intro.srt | 2592 | C1789BEA7BFE410B23B7F3E2A2B3D4398C4CF8E4FFA2F9D5B36FC9E806C4CA9F | 未变（v2 时期同 blob） |
| plugin-intro-contact.png | 517604 | 620270C04F44D6D7FBBA46B58AEF788EFE41C9FC4186504A1EBF6AB569D81B5F | 未变（v2 时期同 blob，画面未改） |

## 2 质检（`engine/qa.mjs`，正片与烧字幕版各自实测）

| 版本 | 判据 | r（音画包络） | n / bin | t | 置换检验 | 趋势 |
|---|---|---|---|---|---|---|
| v3 16:9 | 10/10 PASS | **0.7533** | 151 / 0.5s | 13.9813 | 0/2000 (p=0.0005) | 同相 |
| v3 9:16 | 10/10 PASS | **0.7815** | 151 / 0.5s | 15.2899 | 0/2000 (p=0.0005) | 同相 |
| v3 16:9-subbed | 10/10 PASS | 0.369 | 151 / 0.5s | 4.8462 | 0/2000 (p=0.0005) | 同相 |
| v2（已替换） | 9/10 | −0.1278 | 151 / 0.5s | — | — | 反相 |

烧字幕版 r 下降是叠字改变亮度包络所致（画面变化包络均值 0.09 → 0.24），不是音轨问题。

## 3 复现（v3 = v4 字节级）

```powershell
# 音频母带（13/13 PASS，两跑同 sha 7350AB4C…）
python tools\make-audio.py --spec engine\scenes\plugin-intro-v3.json --out out\bed1.wav
# 跟画重音：注意**不带 --scan**（扫描落点 0.31 与交付口径不同，成片更差）
python out\plugin-intro-v3-av-audio.py --bed out\bed1.wav `
       --env out\plugin-intro-v2-16x9-silent-envelope.json --pulse-at 0.125 --iters 6 `
       --peak-db 8.0 --target-lufs -15.3 --out out\audio.wav     # 期望 5EAF16D5…
ffmpeg -y -i out\plugin-intro-v2-16x9-silent.mp4 -i out\audio.wav -c:v copy -c:a aac -b:a 224k -shortest out\16x9.mp4
node engine\qa.mjs --video out\16x9.mp4 --expect-duration 76 --expect-size 1920x1080   # 期望 r=0.7533 / 10 of 10
```

烧字幕版（本仓库现场生成，工作目录内的 `pi.srt` 即上面的 `plugin-intro.srt`）：

```powershell
ffmpeg -y -i plugin-intro-16x9.mp4 -vf "subtitles=pi.srt:force_style='FontName=Microsoft YaHei,FontSize=22,MarginV=40'" `
       -c:v libx264 -preset medium -b:v 1464k -pix_fmt yuv420p -c:a aac -b:a 224k -movflags +faststart plugin-intro-16x9-subbed.mp4
```

## 4 v2 → v3 变更范围

画面与字幕**未改**（静音源、contact.png、srt 三者 blob 不变）；变更只在**音轨**：
v2 音轨反相（r=−0.1278），v3 音轨同相（r=0.7533/0.7815）。`plugin-intro-v3-qa-*.json` 为成片质检原始报告。
