# このリポジトリを実際のアプリにする手順

`app-template` から作った直後の状態。**この作業は `~/workspace/apps/` 直下のセッションで行う。**
すべて終わったら **このファイルを削除してコミットする。**

## 1. 雛形を埋める

`{{ }}` で囲まれた箇所がプレースホルダ。全ファイルを走査して残りがないことを確認する。

```bash
grep -rn '{{' --exclude-dir=.git .
```

- `CLAUDE.md` — アプリ名・一行説明・固有の制約
- `README.md` — アプリ名・概要・セットアップ手順
- `PLAN.md` — 何を作るか・なぜ作るか（走り書きでよい）
- `HANDOFF.md` — 「現在地」の 3 行だけ埋める。以後も `/apps-workflow:handoff` が現在地だけを書き換える（履歴は積まない）
- `DESIGN.md` — 見た目の決まり。今は上の YAML の名前と一行説明、Overview の「誰が、どんな場面で使うか」「画面の仕事」だけを仮で埋める（3〜5 行）。
  ほかは「未定」のままにし、最初の画面の骨組みが動いたら（見た目を作り込む前に）`/apps-workflow:look` の「A 方向を決める」で埋める

`SPEC.md` `TODO.md` `KNOWLEDGE.md` `logs/decisions.md` は空のまま始めてよい。

## 2. 共通プラグインをこのアプリに入れる

共通のフックと `/apps-workflow:handoff` `/apps-workflow:pr-check` などのスキルは `.claude/settings.json` の
`enabledPlugins` で [apps-workflow プラグイン](https://github.com/n-yoshida-dev/claude-plugins)から読み込む。
**ただし `enabledPlugins` だけでは install されない。アプリごとに project スコープで install が要る**
（install の記録は `~/.claude/plugins/installed_plugins.json` にプロジェクトのパス単位で持たれるため、
テンプレートで入れても新アプリには効かない）。マーケットプレイスの登録だけは PC ごとに初回でよい（今の Windows の PC は登録済み）。

これは `apps-workflow` だけでなく、`enabledPlugins` にある**全プラグイン**（security-guidance・frontend-design・todo-progress など）で同じ。
進捗表を出す todo-progress は公開用のリポジトリ（claude-code-toolkit）にあり、apps-workflow はこれに依存している（入っていないと apps-workflow ごと読み込みに失敗する）。
このアプリのディレクトリで開いた **Claude Code のセッション内**で 1 本ずつ実行する（スコープを聞かれたら project）。

```
/plugin marketplace add n-yoshida-dev/claude-plugins         # マシンごとに初回だけ
/plugin marketplace add n-yoshida-dev/claude-code-toolkit    # マシンごとに初回だけ（todo-progress の置き場）
/plugin install apps-workflow@n-yoshida-dev                  # アプリごとに必要。他のプラグインも同様に 1 本ずつ
/plugin install todo-progress@yoshida-toolkit
```

Claude に代行させる場合は、Claude デスクトップ版に同梱の CLI（`claude.exe`）でまとめて入れられる
（シェルの `claude` は PATH に無いが、フルパスなら Claude が実行できる。デスクトップ版の更新で版のフォルダが変わるので、いちばん新しい版を探す）。

```bash
CLI=$(ls -d ~/AppData/Local/Claude-Data/claude_*/claude-code/*/*/claude.exe | sort -V | tail -1)
for p in $(jq -r '.enabledPlugins | keys[]' .claude/settings.json | tr -d '\r'); do "$CLI" plugin install "$p" --scope project; done
```

`tr -d '\r'` は、Windows の `jq` が行末に付ける改行の印（`\r`）を消すため（残るとプラグインの名前の後ろに混ざり、install が「not found」で失敗する。2026-11-21 に when-to-buy で 3 本が失敗した。Linux や macOS では消す物が無いだけで害は無い）。

どちらも**反映は次のセッションから**。
**`/plugin` の一覧や `claude plugin list` の「enabled」表示は `enabledPlugins` しか見ていないので当てにならない。**
導入済みかどうかは `~/.claude/plugins/installed_plugins.json` にこのアプリのパスの記録があるかで確認する。
起動時のフックの出力はユーザーの画面には出ない。Claude の最初の返答の冒頭に進捗表が貼られていれば todo-progress が動いている。
apps-workflow は、Claude に「起動時に HANDOFF.md を読むよう知らせを受けたか」を尋ねて確かめる。install 忘れは静かに死ぬ
（2026-08-30 の棚卸しで、3 アプリが 8 日間フックなしで動いていたことが判明）。

## 3. プロジェクトを初期化する

必要なほうだけ。CI は `frontend/package.json` と `backend/go.mod` の有無を見て自動でジョブを出し分けるので、
片方だけでも CI は緑になる。

```bash
# フロントエンド
npm create vite@latest frontend -- --template react-ts

# バックエンド
mkdir backend && cd backend && go mod init github.com/n-yoshida-dev/{アプリ名}
```

フロントエンドを作ったら `package.json` に `typecheck` / `lint` / `format:check` / `test` / `build` の
スクリプトを揃える。CI がこの5つを呼ぶ。

frontend を作ったら、手元に `.vscode/settings.json`（JS・JSON・HTML の整形役を Prettier にする。`.vscode/` は `.gitignore` で外してある）と直下の `.prettierignore`（`frontend/` の外は Markdown だけ整形する。`.git/info/exclude` に `/.prettierignore` を足す）を置く。中身の見本は skill-matrix の手元の 2 ファイル（ops の `docs/環境/2026-10-18-VSCode拡張と設定の見直し-2回目-段階1.md` の C17）。

**Vite の見本の見た目を消す。** `npm create vite` の見本は、紫の差し色（`--accent: #aa3bff`）・影・ロゴの画像を持っていて、残すとそのままアプリの見た目になる
（babyfood-check がこの状態のまま。ops の `docs/apps/2026-10-10-apps-デザイン改善の進め方.md` 3-3 節）。作った直後に次をする。

- `frontend/src/App.css` を消し、`frontend/src/App.tsx` から読み込みを外す
- `frontend/src/index.css` の中身を消す。色・文字の大きさ・余白は、見た目の方向を決めてから（`DESIGN.md`）CSS の変数で書く。それまでは書体と行間くらいにとどめる
- 見本の画像（`frontend/src/assets/` の `hero.png`・`react.svg`・`vite.svg` と `frontend/public/icons.svg`）を消し、`App.tsx` を見本の中身の無い最小の画面にする
- ブラウザのタブに出る Vite の印（`frontend/public/favicon.svg`）も消し、`frontend/index.html` の `<link rel="icon" …>` の行を外す。アプリの印は、作ったときに足す
  （印が無くても、撮影の `look.mjs shoot` は画面のエラーに数えない。2026-10-11 に印の無いページで確かめた）
- `frontend/index.html` の `<title>`（`Vite + React + TS`）をアプリの名前にする
- 確かめ：`grep -rnE "aa3bff|App\.css|react\.svg|vite\.svg|hero\.png|icons\.svg|favicon\.svg|Vite \+ React" frontend/src frontend/index.html` が何も出さない

## 4. 開き直す

Claude デスクトップ版で `C:\Users\annin\workspace\apps\{アプリ名}` を作業フォルダにして、新しい会話を開く。

**以降の設計・実装はこの新しい会話で行う。** `~/workspace/apps/` 直下のセッションは閉じてよい。

## 5. このファイルを消す

`main` へ直接コミットせず、作業ブランチ `feat/initial-setup` を切ってコミットし、PR 経由で `main` にマージする。

```bash
git switch -c feat/initial-setup
rm SETUP.md
git status --short   # 自分が変えた・作ったファイルを確かめる
# 下の FILES は、上の一覧のうち自分が変えた物だけにする（作っていない frontend / backend は外す）
FILES="SETUP.md CLAUDE.md README.md PLAN.md HANDOFF.md DESIGN.md frontend backend"
git add -- $FILES
git commit -m "chore: テンプレートから {アプリ名} を初期化" -- $FILES
git push -u origin feat/initial-setup
```

`git add -A` と `commit -a` は使わず、自分が変えたファイルだけをパス指定で add し、コミットにもパスを指定する
（同じリポジトリで別の Claude セッションが同時に作業していることがあるため。決まりは `~/.claude/rules/git-concurrent.md`）。
push のあとの PR の作成からマージまでは `apps-workflow:pr-flow` スキルの手順どおりに進め、CI が通ってからマージする（決まりは `~/workspace/apps/CLAUDE.md` の「Git運用ルール」）。
