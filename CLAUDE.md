# {{APP_NAME}}

{{一行で「何をするアプリか」。SETUP.md の手順で埋めること}}

## セッション開始時にすること

1. **`HANDOFF.md` を読む** — 現在地と次の一手だけがある（決定事項や履歴は持たない）
2. **設計・方針を提案する前に `logs/decisions.md` を読む** — ユーザーと合意済みの判断。蒸し返さない
3. {{個人データを扱うアプリなら「実データが必要な作業なら `PRIVATE.md` を読む（.gitignore 対象）」。扱わないなら、この行ごと削除する}}

未完タスクは SessionStart フックが `TODO.md` から自動で提示する。
引き継ぎは、区切りごとに Claude が `apps-workflow:handoff` スキルをユーザーに聞かずに呼んで書く（作業の PR があるときは、受け入れレビューの前にその PR へ入れる）。
ユーザーに実行を頼まない。呼ぶ場面は `../CLAUDE.md`「開発ドキュメント」と同じ。

## ドキュメントの役割

| ファイル | 役割 | 読み手 |
|---|---|---|
| `HANDOFF.md` | 現在地と次の一手だけ。履歴は持たない | AI |
| `logs/decisions.md` | ユーザーと合意した判断の台帳。`/decide` で追記 | AI |
| `PLAN.md` | 何を作るか・なぜ作るか | AI |
| `SPEC.md` | 確定仕様。実装が参照する正本 | AI |
| `TODO.md` | タスクと進捗 | AI |
| `KNOWLEDGE.md` | 技術的な設計判断・ハマりどころ | AI |
| `DESIGN.md` | 見た目の決まり（Google の DESIGN.md の書式）。画面のファイルを変える前に読む。値の正本は CSS | AI |
| `docs/` | 要件定義・アーキテクチャの解説 | 人間 |

**同じ事実を AI 用と人間用の両方に書かない。** AI 用から人間用へリンクする。

## 守ること

{{このアプリ固有の制約をここに書く。以下は雛形の例。該当しないものは消す}}

- **秘密情報・個人の実数値をリポジトリに入れない。** サンプル・プリセットはすべてダミー値。
  実データは `PRIVATE.md` と gitignore 対象の `*.local.json` で扱う
- **`{{ドメインロジックのディレクトリ}}` は純粋関数のみ。** React・DOM・localStorage を import しない
- **外部由来の数値（税率・料金表・制度の閾値）をコードに直書きしない。**
  JSON に分離し、`source` と `checkedAt` を記録する
- ロジックを変更したら、対応する単体テストを同時に更新する

## Claude Code の設定

共通のフック・スキル・エージェントは [apps-workflow プラグイン](https://github.com/n-yoshida-dev/claude-plugins)から来る（`.claude/settings.json` の `enabledPlugins`）。
部品の一覧はプラグインの [README](https://github.com/n-yoshida-dev/claude-plugins/blob/main/plugins/apps-workflow/README.md) にあり、ここには写さない（写すと古くなる）。
`enabledPlugins` に書くだけでは入らず、アプリごとに project スコープの install が要る（手順は `../app-template/SETUP.md` §2）。

このアプリ固有のフック・スキルは `.claude/` に置く。固有のルールが増えたら `.claude/rules/` に切り出し、CLAUDE.md からはリンクだけにする。

## 共通ルール

自作プロダクト共通の開発ルール（技術スタック・Git運用・コーディング規則）は `../CLAUDE.md` にある。
上位ディレクトリの CLAUDE.md は自動でロードされる。
