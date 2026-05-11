---
title: 'AIボット初日、3つのエラーで詰んだ話【デイトレボット開発記 #4】'
emoji: "🔥"
type: "tech"
topics: ["python", "finance", "trading", "ai"]
published: true
---

## 動かしてみたら、全然動かなかった

「PCを起動してkabu STATIONを開くだけでいい」と思っていました。

結果：**ボットは動かず、LINEも来ず、気づいたらPCがスリープしていました。**

今回はボット初日に遭遇した3つのエラーと、その解決方法を記録します。

---

## エラー① タスクスケジューラーがボットを起動できなかった

### 症状

朝8:45に自動起動するよう設定したはずなのに、何も起きない。

### 原因を調査

Windowsイベントログを確認したところ、以下の時系列が判明しました。

```
07:28  Windowsにログイン
08:48  タスクスケジューラーが起動しようとした
       → エラーコード 0x80070002（ファイルが見つからない）
09:00  System Idle でスリープ
```

タスクに `python.exe` とだけ登録していたため、スケジューラーがPythonのフルパスを見つけられなかったのが原因です。

さらに電源設定を見ると：

```
Power Management: No Start On Batteries
```

バッテリー動作時は起動しない設定になっていました。

### 修正方法

Pythonのフルパスを確認してタスクを再登録。

```powershell
# Pythonのフルパスを確認
(Get-Command python).Source
# → C:\Users\kuren\AppData\Local\Microsoft\WindowsApps\python.exe
```

XMLでタスクを定義して登録しました（PowerShellのコマンドレットより確実）。

```xml
<Task>
  <Settings>
    <DisallowStartIfOnBatteries>false</DisallowStartIfOnBatteries>
    <StopIfGoingOnBatteries>false</StopIfGoingOnBatteries>
    <WakeToRun>true</WakeToRun>
  </Settings>
  <Actions>
    <Exec>
      <!-- フルパスで指定 -->
      <Command>C:\Users\kuren\AppData\Local\Microsoft\WindowsApps\python.exe</Command>
      <Arguments>C:\Users\kuren\predict_stock\main.py</Arguments>
      <WorkingDirectory>C:\Users\kuren\predict_stock</WorkingDirectory>
    </Exec>
  </Actions>
</Task>
```

あわせてスリープも無効化：

```powershell
powercfg /change standby-timeout-ac 0   # AC電源でスリープしない
powercfg /change standby-timeout-dc 0   # バッテリーでもスリープしない
```

:::message
タスクスケジューラーに登録するコマンドは**フルパス必須**です。`python`だけでは環境によって見つからないことがあります。
:::

---

## エラー② 🧠絵文字でボットがクラッシュ

### 症状

LINEにこんな通知が届きました。

```
⚠️ BOTエラー: 'cp932' codec can't encode character
'\U0001f9e0' in position 0: illegal multibyte sequence
```

### 原因

Windowsの標準出力エンコードは `cp932`（Shift-JIS）です。

反省会の出力に含まれる 🧠（U+1F9E0）がcp932で表現できず、print文がクラッシュ。そのエラーメッセージ自体がLINEに送られていました。

### 修正方法

`main.py`の先頭でstdoutをUTF-8に強制変換：

```python
import sys, io

if sys.stdout.encoding != 'utf-8':
    sys.stdout = io.TextIOWrapper(
        sys.stdout.buffer,
        encoding='utf-8',
        errors='replace'
    )
if sys.stderr.encoding != 'utf-8':
    sys.stderr = io.TextIOWrapper(
        sys.stderr.buffer,
        encoding='utf-8',
        errors='replace'
    )
```

:::message alert
Windowsで日本語・絵文字を含むPythonスクリプトを動かす場合、**必ずstdoutをUTF-8に変換**しておくのが安全です。タスクスケジューラー経由の実行は特に注意が必要です。
:::

---

## エラー③ 反省会のLINEが3回届いた

### 症状

16時頃、反省会の通知が3通届きました。

### 原因

本番のボットループ（1回）＋動作確認のテスト実行（2回）が重なっただけです。

ボット本体には `reflected_today` フラグがあり、1日1回しか実行されません：

```python
if self._is_time("16:00", margin=2) and not self.reflected_today:
    self.do_reflection()
    self.reflected_today = True  # ← 二重実行を防ぐ
```

今日はテスト中にこのフラグを通さず直接呼んでしまいました。通常運用では1回だけ届きます。

---

## 最終的に動作確認できた内容

エラーを全部潰したあと、手動でスクリーニングを実行。

```
🔍 本日の監視銘柄 (10銘柄)
  9501 [★★★☆☆] 上昇:7/10点
         前日比:+3.2% ATR:3.8%
         理由: 引け強(92%), 3日上昇(+6.6%), 下ひげ反発, RS優
  7211 [★★★☆☆] 上昇:6/10点
         前日比:+6.3% ATR:3.6%
         理由: 引け強(90%), 3日上昇(+7.4%), 買い集め
  ...
```

LINEに届いて、テスト売買も正常に記録されることを確認しました。

---

## 今日の教訓まとめ

| エラー | 原因 | 対処 |
|--------|------|------|
| タスクが起動しない | python.exeのフルパス未指定 | XMLで再登録 |
| 絵文字でクラッシュ | Windowsのcp932エンコード | stdout をUTF-8に変換 |
| LINE通知が複数届く | テスト実行と本番が重複 | 通常運用では1日1回のみ |

---

## 明日から本格稼働

設定は完了しました。

明日の朝：
1. PCを起動してWindowsにログイン
2. kabu STATIONを起動

これだけで8:45に自動起動、8:50にLINEに銘柄リストが届くはずです。

うまくいったかどうかも記録します。

> ⚠️ 投資は自己責任です。この記事は特定の投資を推奨するものではありません。

---

## 前の記事

https://zenn.dev/ryoya_aitech/articles/daytrade-bot-02-system

---

*📝 このシリーズは毎週更新予定です。*
*💬 感想・質問はコメントでどうぞ。*
