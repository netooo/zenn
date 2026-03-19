---
title: "タスク管理もミーティングメモも Obsidian に自動で集まる仕組みを作った"
emoji: "📅"
type: "tech"
topics: ["obsidian", "pkm", "idea", "google_calendar"]
publication_name: fukurou_labo
published: false
---

こんにちは、フクロウラボの満江です。

かなり乗り遅れた感はあるんですが、最近 Obsidian に入門して「タスク・議事録・日報・メモ」等々の管理を行っています。  
所感としては、自分好みにカスタマイズ出来る点が非常に良いですね。最高です。  
とはいえ、「毎日 Daily Note を作成して、前日のタスクをコピーしつつ Google Calendar の予定も反映して...」だったり、  
「ミーティングの度に Note を作成して、Google Meet の Gemini メモから要点を転記して...」という作業が地味に面倒でした。

上記の地味な作業を改善するために色々と自動化を進めてきたので、今回はその内容を紹介したいと思います!!

# できること

本記事で紹介する仕組みを導入すると、以下が自動化されます。

- 前日の未完了タスクを当日の Daily Note に **自動引き継ぎ**
- Google Calendar の予定を Daily Tasks に **自動追加**
- Google Meet の Gemini メモを Obsidian に **自動取り込み**


なお、私の Daily Note は下記のような構成となっており、積んでいるタスク(Tasks Left)と当日タスク(Daily Tasks)の他に、割り込みタスク(Cut In Tasks)、日報(Daily Report)、メモ(Memo) のセクションがあります。  
Tasks Left と Daily Tasks に関しては、未完了のタスクのみ翌日に引き継がれるようになっています。  
また Daily Tasks には Google Calendar の予定を取得して自動で追加しています。  
Gemini メモの取り込みは 5_Work/ 配下に格納しており、詳しくは後述します。

![daily_note_overview](/images/obsidian_google_calendar/daily_note_overview.png)

(右側にはターミナルを配置しており、この記事も Obsidian 上で動く Claude Code に下書きを作成してもらいました)

# 構成

ディレクトリ構成はシンプルで、用途ごとにフォルダを分けています。

```
obsidian/
├── 1_News/   # 日次テックニュース(自動追加)
├── 2_Daily/  # Daily Note
├── 3_Weekly/ # Weekly Note
├── 4_People/ # 人物ノート
├── 5_Work/   # 雑多メモ(Google Meet Gemini メモを自動同期)
└── Scripts/  # 自動化スクリプト
    ├── sync_gdrive/       # Google Drive → Obsidian 同期
    ├── fetch_tech_news/   # テックニュース取得
    └── ...                # その他スクリプト
```

今回は `2_Daily/` と `5_Work/` に関わる自動化を紹介します。

# 2_Daily/ の自動化

## 動作イメージ

Daily Note を新規作成すると、直近の未完了タスクが Tasks Left と Daily Tasks に自動で引き継がれます。  
更に Daily Tasks には Google Calendar の予定が自動で追加されます。こんな感じ。

| Before | After |
|-|-|
| ![daily_note_before](/images/obsidian_google_calendar/daily_note_before.png) | ![daily_note_after](/images/obsidian_google_calendar/daily_note_after.png) |

## Daily Note のテンプレート構成

私は Daily Note のテンプレートを `2_Daily/0_Template.md` として用意しており、以下のセクションで構成しています。

```markdown
---
# 📌 Tasks Left    ← バックログのような扱い。前日の未完了タスクを自動で引き継ぐ。
- [ ] TaskA
---
# 🗓️ Daily Tasks   ← 当日行うタスク群。前日の未完了タスクを自動で引き継ぐ + Google Calendar から自動で追加。
- [ ] TaskB
---
# 🚨 Cut In Tasks  ← 当日中に発生した割り込みタスクを手動で管理。
---
# 🗣️ Daily Report
## 【やったこと】
## 【所感】
---
# 📝 Memo
```

`📌 Tasks Left` と `🗓️ Daily Tasks` が自動入力されるセクションで、残りは手動で記入します。

## プラグイン設定

### Daily Notes コアプラグイン
Daily Notes コアプラグインでは「日付の書式」「新規ファイルの場所」「テンプレートファイルの場所」を設定しておいてください。

![daily_notes_settings](/images/obsidian_google_calendar/daily_notes_settings.png)

### Templater コミュニティプラグイン

[Templater](https://github.com/SilentVoid13/Templater) は Obsidian のコミュニティプラグインで、JavaScript を使った動的なテンプレートを作成できます。  
ノート作成時にスクリプトを実行し、ファイルの読み込みや外部データの取得が可能です。  
テンプレートノートにスクリプトを記述して自動化を実現するために有効化しておきましょう。

### ICS コミュニティプラグイン

Google Calendar との連携には [ICS](https://github.com/muness/obsidian-ics) プラグインを使用しています。  
ICS プラグインは Google Calendar の iCal フィードを読み込み、Templater からイベントデータを取得できるようにしてくれます。

#### 1. iCal 形式の非公開URL を取得

Google Calendar の「マイカレンダー」から「設定と共有」→「カレンダーの統合」を開き、「iCal 形式の非公開 URL」をコピーします。

![google_calendar_ical_url](/images/obsidian_google_calendar/ical_secret_url.png)

:::message
「iCal 形式の非公開 URL」の取り扱いには十分注意してください。
:::

#### 2. プラグイン設定

ICS プラグインをインストールし、「Calendar URL」に上記で取得した URL を登録して設定は完了です。

![ics_plugin_settings](/images/obsidian_google_calendar/ics_plugin_settings.png)

## Tasks Left への自動追加

`📌 Tasks Left` セクションはバックログとして機能させており、数日間に渡って未完了のタスクが残ることがあります。  
毎日毎日、新しい Daily Note を作成しては前日の未完了タスクをコピーして...というのは地味に面倒なので、ここでは直近の Daily Note から未完了タスク（`- [ ]`）を自動的に引き継ぐ設定をしています。  
土日・祝日を考慮して、前日だけでなく最大7日前まで遡って最新の Daily Note を探す工夫をしています。  
Daily Note テンプレートファイルの Tasks Left セクション直下に下記のスクリプトを配置することでこれを実現します。

```javascript
<%*
// 前日の未完了タスクを引き継ぐ
const TASKS_LEFT_MAX_LOOKBACK_DAYS = 7;
let tasks = "";

for (let offset = 1; offset <= TASKS_LEFT_MAX_LOOKBACK_DAYS; offset++) {
    const date = tp.date.now("YYYY-MM-DD", -offset);
    const file = tp.file.find_tfile(`2_Daily/${date}`);
    if (!file) continue;

    const content = await app.vault.read(file);
    const sectionMatch = content.match(/# 📌 Tasks Left([\s\S]*?)(?=\n---|\n#|$)/);
    if (sectionMatch && sectionMatch[1]) {
        const unfinished = sectionMatch[1]
            .split("\n")
            .filter(line => line.includes("- [ ]") && line.trim() !== "- [ ]");
        if (unfinished.length > 0) tasks = unfinished.join("\n");
    }
    break;
}
-%>
<% tasks %>
```

ポイントは以下です。

- `tp.date.now("YYYY-MM-DD", -offset)` で対象日付を計算
- `tp.file.find_tfile(`2_Daily/${date}`)` でノートを検索し、存在しない日（休日など）はスキップ
- 未完了タスクは正規表現でセクション内容を抽出し、`- [ ]` で始まる行のみを取得
- `line.trim() !== "- [ ]"` で空のチェックボックス（`- [ ]` のみの行）は引き継ぎ対象から除外

これで直近の未完了タスクが自動的に引き継がれるようになりました!!

## Daily Tasks への自動追加

`# 🗓️ Daily Tasks` セクションは当日行うタスクを管理しています。  
ここでも Tasks Left と同様に前日の未完了タスクを引き継いでいますが、これに加えて Google Calendar の予定も自動で追加するようにしています。  
ただし定常的なミーティングやランチといった予定はタスク管理の対象外にしたいので、ここはスクリプトで一工夫しています。  
上記で扱った Daily Note テンプレートファイルの Daily Tasks セクション直下に下記のスクリプトを配置します。

```javascript
<%*
// タスク化する必要のない定常的な予定は対象外
const SKIP_LIST = ['デイリーMTG', 'Lunch'];

// 前日の未完了タスクを引き継ぐ
const DAILY_TASKS_MAX_LOOKBACK_DAYS = 7;
let dailyTasks = "";

for (let offset = 1; offset <= DAILY_TASKS_MAX_LOOKBACK_DAYS; offset++) {
    const date = tp.date.now("YYYY-MM-DD", -offset);
    const file = tp.file.find_tfile(`2_Daily/${date}`);
    if (!file) continue;

    const content = await app.vault.read(file);
    const sectionMatch = content.match(/# 🗓️ Daily Tasks([\s\S]*?)(?=\n---|\n#|$)/);
    if (sectionMatch && sectionMatch[1]) {
        const unfinished = sectionMatch[1]
            .split("\n")
            .filter(line => line.includes("- [ ]") && line.trim() !== "- [ ]");
        if (unfinished.length > 0) dailyTasks += unfinished.join("\n") + "\n";
    }
    break;
}

// 今日のカレンダーイベントを追加（スキップリスト除外）
const allEvents = await app.plugins.getPlugin('ics').getEvents(
    moment().startOf('day'), moment().endOf('day')
);
allEvents
    .filter(event => !SKIP_LIST.includes(event.summary))
    .sort((a, b) => a.utime - b.utime)
    .forEach(event => { dailyTasks += `- [ ] ${event.summary}\n`; });
-%>
<% dailyTasks %>
```

ポイントは以下です。

- 前日の未完了タスクの引き継ぎは Tasks Left と同様のロジック(正規表現のみ異なる)
- `app.plugins.getPlugin('ics').getEvents()` で Google Calendar から当日のイベントを取得
- `.filter(event => !SKIP_LIST.includes(event.summary))` で `SKIP_LIST` に含まれるイベントを除外

これで Google Calendar の予定も Daily Tasks に自動で追加されるようになりました!!


# 5_Work/ の自動化

## 概要
5_Work/ は雑多なメモフォルダとして活用しています。  
ミーティングのメモや議事録もここに保存するようにしており、プレーンな Note 以外にも Canvas Note なども存在します。  
しばらく運用していると「ミーティングのメモや議事録を毎回 Obsidian の Note に起こすのが面倒だなぁ」と感じるようになったので、半自動化することにしました。  
私が所属しているフクロウラボでは Google Meet でミーティングを実施することが多く、更に [Gemini による自動メモ生成機能](https://workspace.google.com/intl/en/solutions/ai/ai-note-taking/) を使用しているため、会議終了後には要約されたメモが Google Drive に保存されます。  
この Gemini メモを Obsidian に自動で同期(正確には Vault で指定したディレクトリ配下に複製) する仕組みを作ることで、ミーティングの度に Note を起こす手間を削減しています。

## 仕組み

```
Google Meet
  ↓ (自動)
Gemini によるメモ (.docx)
  ↓ (自動) 
Google Drive
  ↑ (10分毎 / launchd)
rclone で取得
pandoc で Markdown に変換
  ↓
「まとめ」「詳細」セクションを抽出
  ↓
5_Work/ に保存
```

やることはシンプルで、Google Drive に保存された Gemini メモを定期的に `{vault_path}/5_Work/` 配下に複製するだけです。  
Obsidian のプラグインを用いてどうのこうの...という選択肢も検討したんですが、最終的には Python スクリプトを定期実行する形にしました。

:::details 検討した選択肢
- [Google Drive Sync プラグイン](https://github.com/RichardX366/Obsidian-Google-Drive): Google Drive と Obsidian を双方向で同期できる。
- [Drive for Desktop](https://support.google.com/drive/answer/16631477?sjid=5538544712847723294-NC): Google Drive をローカルドライブとしてマウントできる。

共に双方向の同期が可能で便利そうだったんですが、今回は「Google Drive → Obsidian の一方向」かつ「Gemini メモのみを対象にしたい」という要件であるため、オーバースペックという判断をしました。
:::


## 実装

### 事前準備

スクリプトの実行には rclone と pandoc が必要です。  
rclone は Google Drive からファイルを取得するために使用します。  
pandoc は Gemini メモの `.docx` を Markdown に変換するために使用します。

```bash
brew install rclone pandoc
```

rclone の Google Drive 設定は以下のコマンドから対話的に行います。

```bash
rclone config
# → n（新規作成）→ 名前: gdrive → ストレージ: Google Drive
```

### Google Drive から Gemini メモを取得

Google Drive から Gemini メモを取得し、5_Work/ 配下に保存する Python スクリプトを用意しました。

:::details Script
```python
#!/usr/bin/env python3
"""
Google Drive の指定フォルダを監視し、新しい .md ファイルを
Obsidian の 5_Work/ へリネームしてコピーするスクリプト。

リネームルール:
  YYYY／MM／DD HH:MM JST に開始した会議 - Gemini によるメモ.md
    → 2026-02-20_19-15_会議.md

  1on1 - YYYY／MM／DD HH:MM JST - Gemini によるメモ.md
    → 2026-01-29_11-28_1on1.md
"""

import json
import re
import os
import subprocess
import tempfile
from datetime import datetime, date

REMOTE      = "gdrive:"
FOLDER_ID   = "{FOLDER_ID}"  # Gemini メモが保存される Google Drive のフォルダID を指定
DEST        = "{vault_path}/5_Work" # Gemini メモの保存先
LOG         = "{log_path}/sync_gdrive.log" # ログの出力先
RCLONE      = "/opt/homebrew/bin/rclone" # rclone のパスを指定
PANDOC      = "/opt/homebrew/bin/pandoc" # pandoc のパスを指定


# ── ログ ────────────────────────────────────────────────────────────────────

def log(message: str) -> None:
    line = f"{datetime.now().strftime('%Y-%m-%d %H:%M:%S')}  {message}\n"
    with open(LOG, "a") as f:
        f.write(line)


# ── リネーム ─────────────────────────────────────────────────────────────────

# 全角スラッシュを使った日付パターン: YYYY／MM／DD HH:MM
DATE_PATTERN = re.compile(r"(\d{4})／(\d{2})／(\d{2})\s+(\d{2}):(\d{2})")


def round_to_15min(hour: int, minute: int) -> str:
    """時刻を15分単位に四捨五入して HHmm 形式の文字列で返す（例: 16:58 → '1700'）"""
    total = hour * 60 + minute
    rounded = round(total / 15) * 15 % (24 * 60)  # 24:00 は 00:00 に折り返す
    h, m = divmod(rounded, 60)
    return f"{h:02d}{m:02d}"


def rename_md(filename: str) -> str:
    """
    Google Meet / Gemini のファイル名を yyyy-MM-dd_HHmm_title.md 形式に変換する。
    時刻は15分単位で四捨五入（例: 16:58 → 1700、10:14 → 1015）。
    日付が見つからない場合は「Gemini によるメモ」だけ除去して返す。
    拡張子は問わず、出力は常に .md。
    """
    base = os.path.splitext(filename)[0]

    # 「- Gemini によるメモ」「Gemini によるメモ」を除去
    base = re.sub(r"\s*[-－]?\s*Gemini\s*によるメモ", "", base).strip()

    m = DATE_PATTERN.search(base)
    if not m:
        return f"{base.strip(' -　')}.md"

    year, month, day, hour, minute = m.groups()
    time_str = round_to_15min(int(hour), int(minute))
    date_str = f"{year}-{month}-{day}_{time_str}"

    # 日付部分と前後の JST・に開始した・会議 などを整理してタイトルを抽出
    title = DATE_PATTERN.sub("", base)
    title = re.sub(r"\s*JST\s*", " ", title)
    title = re.sub(r"に開始した", "", title)
    title = title.strip(" -　")

    return f"{date_str}_{title}.md" if title else f"{date_str}.md"


# ── セクション抽出 ───────────────────────────────────────────────────────────────

EXTRACT_SECTIONS = ["まとめ", "詳細"]

def extract_sections(content: str) -> str:
    """「まとめ」と「詳細」セクションのみ抽出して返す"""
    lines = content.splitlines()
    result = []
    in_target = False
    target_level = 0

    for line in lines:
        m = re.match(r'^(#{1,6})\s+(.+)', line)
        if m:
            level = len(m.group(1))
            title = m.group(2).strip()
            is_target = any(name in title for name in EXTRACT_SECTIONS)

            if is_target:
                if result:
                    result.append("")  # セクション間の空行
                in_target = True
                target_level = level
                result.append(line)
            elif in_target and level <= target_level:
                in_target = False
            elif in_target:
                result.append(line)
        elif in_target:
            result.append(line)

    return "\n".join(result).strip()


# ── rclone 操作 ───────────────────────────────────────────────────────────────

def is_gemini_memo(filename: str) -> bool:
    """ファイル名（拡張子なし）が「Gemini によるメモ」で終わっているか判定する"""
    base = os.path.splitext(filename)[0]
    return bool(re.search(r"Gemini\s*によるメモ$", base))


def list_remote_files() -> list[str]:
    """リモートフォルダのファイルのうち、当日更新かつ「Gemini によるメモ」で終わるものを返す"""
    result = subprocess.run(
        [RCLONE, "lsjson", REMOTE,
         "--drive-root-folder-id", FOLDER_ID,
         "--files-only"],
        capture_output=True, text=True
    )
    if result.returncode != 0:
        log(f"ERROR rclone lsjson: {result.stderr.strip()}")
        return []

    try:
        files = json.loads(result.stdout)
    except json.JSONDecodeError:
        log("ERROR: JSON parse failed")
        return []

    today = date.today()
    result_files = []
    for f in files:
        try:
            # ModTime は UTC（末尾 Z）→ ローカルタイムゾーンに変換して日付比較
            mod_time = datetime.fromisoformat(f["ModTime"].replace("Z", "+00:00"))
            if mod_time.astimezone().date() == today and is_gemini_memo(f["Name"]):
                result_files.append(f["Name"])
        except (KeyError, ValueError):
            pass

    return result_files


def copy_file(remote_name: str, local_name: str) -> bool:
    """リモートファイルをダウンロードし、Markdown に変換して保存する"""
    ext = os.path.splitext(remote_name)[1].lower()

    with tempfile.TemporaryDirectory() as tmpdir:
        tmp_src = os.path.join(tmpdir, remote_name)

        # 1. Google Drive からダウンロード
        dl = subprocess.run(
            [RCLONE, "copyto",
             f"{REMOTE}{remote_name}",
             tmp_src,
             "--drive-root-folder-id", FOLDER_ID],
            capture_output=True, text=True
        )
        if dl.returncode != 0:
            log(f"ERROR download: {dl.stderr.strip()}")
            return False

        dest_path = os.path.join(DEST, local_name)

        # 2. .docx は pandoc で Markdown に変換 → セクション抽出、それ以外はそのままコピー
        if ext == ".docx":
            tmp_md = os.path.join(tmpdir, "converted.md")
            cv = subprocess.run(
                [PANDOC, tmp_src, "-f", "docx", "-t", "gfm",
                 "--wrap=none", "-o", tmp_md],
                capture_output=True, text=True
            )
            if cv.returncode != 0:
                log(f"ERROR pandoc: {cv.stderr.strip()}")
                return False
            with open(tmp_md, encoding="utf-8") as f:
                extracted = extract_sections(f.read())
            with open(dest_path, "w", encoding="utf-8") as f:
                f.write(extracted + "\n")
        else:
            import shutil
            shutil.copy2(tmp_src, dest_path)

    return True


# ── メイン ────────────────────────────────────────────────────────────────────

def main() -> None:
    log("─── 同期開始 ───────────────────────────")

    remote_files = list_remote_files()
    if not remote_files:
        log("転送対象の .md ファイルなし")
        log("─── 同期完了 ───────────────────────────\n")
        return

    copied = skipped = errors = 0

    for remote_file in remote_files:
        new_name  = rename_md(remote_file)
        dest_path = os.path.join(DEST, new_name)

        if os.path.exists(dest_path):
            log(f"SKIP  {remote_file!r:50s} → {new_name}")
            skipped += 1
        else:
            if copy_file(remote_file, new_name):
                log(f"COPY  {remote_file!r:50s} → {new_name}")
                copied += 1
            else:
                log(f"ERROR {remote_file!r:50s} → {new_name}")
                errors += 1

    log(f"結果: コピー {copied} 件 / スキップ {skipped} 件 / エラー {errors} 件")
    log("─── 同期完了 ───────────────────────────\n")


if __name__ == "__main__":
    main()
```
:::

はい。いかにもな(AIに生成させた)コードですね。  
Obsidian 内に常駐させている Claude Code から指示するとシームレスで捗ります。  
正直この辺は個人利用のスクリプトということもあり、人間が頑張って実装するメリットはほぼないですよね。  

とはいえ、こだわっているポイントもあるので解説しておきます。

- Gemini メモ以外のファイルは不要なので、「Gemini によるメモ」で終わるファイルのみを対象にしています。
- Gemini メモは「{タイトル} - {Google Meet 開始時刻} - Gemini によるメモ.docx」という名前で保存されているため、見やすいように「YYYY-MM-DD_HHMM_{タイトル}.md」形式に変換しています。
  - {Google Meet 開始時刻} をそのまま適用すると、Google カレンダーの予定時刻と数分ずれて気持ち悪かったので、15分単位で四捨五入して整形しています。
  - 「- Gemini によるメモ」は不要なので除去しています。
- Gemini メモは .docx 形式で保存されるため、そのまま複製すると文字化けします。ここは pandoc を用いて Markdown に変換しています。
- Gemini メモは「招待済み」や「添付ファイル」といった情報も含まれているため、「まとめ」と「詳細」セクションのみを抽出しています。
- 重複を避けるため、既に同名のファイルが存在する場合はスキップするようにしています。

### launchd によるスケジュール実行

私の端末は macOS なので 10分おきの定期実行には `launchd` を用いています。  
`~/Library/LaunchAgents/com.obsidian.gdrive-sync.plist` を作成します。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
    <dict>
        <key>Label</key>
        <string>com.obsidian.gdrive-sync</string>

        <key>ProgramArguments</key>
        <array>
            <string>{python_command_path}</string>
            <string>{sync_gdrive_script_path}</string>
        </array>

        <key>StartInterval</key>
        <integer>600</integer>

        <key>RunAtLoad</key>
        <true/>
    </dict>
</plist>
```

作成後は `launchctl load` で登録します。

```bash
launchctl load ~/Library/LaunchAgents/com.obsidian.gdrive-sync.plist
```

登録状態の確認は `launchctl list` で行えます。

```bash
launchctl list | grep gdrive
# -   0   com.obsidian.gdrive-sync
```

作成した plist ファイルやスクリプトファイルの実行権限が問題なければ設定は完了です!!

## 動作イメージ

会議が終わって Gemini メモが作成されると、少しして `5_Work/` 配下にメモが追加されます。  
良い感じですね!!

![gemini_memo](/images/obsidian_google_calendar/gemini_memo.png)

ログでは同期状況を確認できます。

```
2026-03-16 12:42:00  ─── 同期開始 ───────────────────────────
2026-03-16 12:42:01  COPY  '1on1 - 2026／03／16 11:28 JST - Gemini によるメモ.docx' → 2026-03-16_1130_1on1.md
2026-03-16 12:42:01  結果: コピー 1 件 / スキップ 0 件 / エラー 0 件
2026-03-16 12:42:01  ─── 同期完了 ───────────────────────────
```

# 最後に

まず Daily Note の引き継ぎと Google Calendar の予定自動追加は誰でもサクッと導入できるので、是非試してみてください！  
議事録(Gemini メモ) の自動取り込みなんですが、二つ欠点があるので紹介しておきます。  
一つ目は、Slack のハドルミーティングや Zoom などには対応していない点です :)  
私は現状、圧倒的に Google Meet が多いので特に困ることはないんですが、フクロウラボでは Slack のハドルミーティングも活用しているので、いつかそちらも対応したいなぁと思ったり。。。  
二つ目は、Google Meet が終了しないと Gemini メモが生成されない点です。  
つまりミーティング中に Obsidian に直接メモを取ると、Note が二重で作成されちゃいます。  
私は Raycast Notes を愛用しているので、一時的に Raycast Notes にメモを取って、ミーティング終了後に Gemini メモと統合する運用にしています。  
人によっては、先に Obsidian に Note を作成して後から Gemini メモを統合したいケースもあると思うので、その場合はスクリプトを改造してもらえればと思います！  
本記事の設定はあくまで自分好みにカスタマイズした一例なので、これを参考にしつつ皆さんも自分好みにアレンジしてもらえればと思います！  

それでは、良い Obsidian ライフを!!
