# Collab Studio — AITuber対談コラボ用 MCP + Webアプリ

- 要求確定日: 2026-09-23
- 状態: **要求FIX（ADR未起票）**
- 実装状態: 初期文書のみ。以下は実装済み機能や検証済み保証を示すものではない。
- リポジトリ: https://github.com/FYuki/aituber-collaboration

## 1. 目的・公開範囲

複数AITuberの対談コラボ配信を実現するWebアプリケーション。各キャラクター（AI人格）がMCP clientとして接続し、Collab StudioがMCP serverとWebアプリを提供する。スタジオがアバター描画・音声ミックス・合成出力を担う。

本addonのみを独立したpublicリポジトリで公開し、他者がセルフホストできる構成とする。キャラクターの人格Core・記憶・TTS・モデル資産をこのリポジトリへ移管するものではない。

同時最大4名を想定し、v0の検証対象は2名。初期運用は身内招待制とし、ソフトウェアの公開範囲とセッションへの参加権限を分ける。

## 2. 採用方式: セッション限定フェッチ + 中央レンダリング

参加者はjoin時に `model_url`（HTTPS、署名付きURL）を提示する。presigned URLを推奨する。

スタジオページがセッション中だけモデルをfetchして描画する。**取得したモデル資産を永続ストレージへ一切書かないことを設計上の不変条件とする。** モデルはメモリ上だけで扱う。

資産のcustodyは参加者側に残し、スタジオは「レンダラーを貸す」役割に限定する。これは資産の権利移転や、スタジオ側での恒久保管を意味しない。

`.motion3.json`等のモーション実体データはモデルフォルダに同梱され、スタジオが実行する。キャラクター側が持つのは「いつ・どのモーションを出すか」という意味レベルの判断のみ。

emotionタグからモデル固有のモーション名・表情名へのマッピングは、参加者が提示するコラボプロファイルmanifestに含める。

### 棄却した案

| 案 | 棄却理由・扱い |
| --- | --- |
| 各参加者から透過アバター映像を伝送 | alphaの扱いに依存する経路を採らず、中央レンダリングを採用する。packed-matte / chroma-key方式も対象外。 |
| モデルをスタジオへ永続格納 | ライセンス・保管責任・削除対応の恒久コストを回避する。 |

## 3. アーキテクチャと責務

- **制御面: MCP / Streamable HTTP**。session管理・発話報告・モーション指示・サーバーからのイベント通知（SSE）を扱う。
- **メディア面: LiveKit**。参加者の音声trackのみを扱い、アバター映像trackは存在しない。
- **出力: OBS Browser Source用フルスクリーンページ**。モデル描画、音声ミックス、字幕、レイアウトをスタジオページへ集約する。

```text
キャラクター                              Collab Studio
┌─────────────────────┐   MCP（制御）   ┌────────────────────────────┐
│ Core（例: FastAPI）  │◄──────────────►│ MCP server                 │
│  ├ MCP client       │                │  session / transcript / 通知│
│  ├ TTS音声          │──LiveKit音声───►│ Webアプリ                  │
│  │  audio track     │                │  ・モデルfetch（使い捨て） │
│  └ model_url提供    │                │  ・Live2D描画 / リップシンク│
└─────────────────────┘                │  ・音声mix / 字幕 / layout │
          │ モデル配信元                └─────────────┬──────────────┘
          └──── HTTPS fetchの応答 ─────────────────►│
                                                      ▼
                                      OBS Browser Source → 配信
```

| 主体 | 担当 | 担当しないこと |
| --- | --- | --- |
| 参加キャラクター | 人格・会話判断、TTS生成と音声publish、発話テキスト報告、意味レベルのモーション指示、モデルURLとprofileの提供 | 他参加者の音声subscribe、STT、相手アバターの描画 |
| MCP server | セッション管理、認証・認可、メモリ上のtranscript、イベント配送 | 人格Coreの代行、モデルやtranscriptの永続保管 |
| スタジオページ | モデル取得・検証・描画、参加者音声の受信・ミックス、リップシンク、字幕、レイアウト | キャラクターの会話内容・人格・長期記憶の決定 |
| ホスト | Web UIでの操作、OBSへのページ設定と配信 | 参加キャラクターごとのローカルアバター合成 |

図中のFastAPIは参加キャラクターCoreの例であり、Collab StudioのMCP server実装言語・フレームワークを確定するものではない。スタジオページの実行場所と「中央レンダリング」の物理的な配置は、設計確認事項で明確にする。

## 4. 確定要求

| ID | 内容 |
| --- | --- |
| A | 同時最大4名想定・v0は2名検証・身内招待制。ソフト自体は公開し、他者セルフホスト可。専用リポジトリ `FYuki/aituber-collaboration` を使用する。 |
| B | v0対応はLive2D（Cubism 4系）と静止画PNG。`model_url`は署名付きHTTPS、モデルはメモリのみ。読込失敗時はプレースホルダーで会話を継続する。形式・サイズ上限を検証する。光織のモデルは未所持のため、開発はCubismサンプルと静止画で進める。 |
| C | 参加キャラクターの音声はLiveKit publish-onlyとし、他者音声をsubscribeしない。相手発話はMCPテキスト通知で受け取り、STTは使用しない。スタジオ側は参加者別の音声解析からリップシンクを行い、`utterance.report`のテキストから字幕を生成する。 |
| D | モーション指示はemotionタグ・モーション名の意味レベル。瞬き・呼吸・アイドル・物理はスタジオ既定。ターン制御は自由会話ベースとし、v0は発話通知のみでfloor管理を導入しない。 |
| E | 出力はOBS Browser Source用ページ。v0は2-up固定、背景、ネームプレート、字幕ON/OFFに対応する。ホスト操作はWeb UIで行う。 |
| F | 認証はセッションスコープのBearer token。SNSログインと運営者ダッシュボードは将来課題。外部参加者向けにはLiveKit TLS/TURN整備を要件差分として扱う。 |
| G | transcriptはメモリのみで非永続化。`model_url`を他参加者・配信・ログへ露出させない。光織側の発話履歴は`collab` provenance付きで既存memory policyに従う。これはスタジオ側のtranscript永続化を許可するものではない。 |

## 5. MCPツール面（たたき台・API未確定）

以下はユーザー提示のたたき台であり、確定したwire schemaや認証情報の受け渡し方法ではない。

```text
collab.join(session_token, {name, model:{type,url}, profile})
    → {livekit_url, livekit_token, slot, session_state}
collab.leave()
collab.utterance.report(text, duration_ms, emotion?)  # transcript・字幕の正本
collab.context.tail(n)                               # 直近発話列
collab.avatar.motion.play(name)                      # 自キャラのみ
collab.avatar.expression.set(name)                   # 自キャラのみ
collab.react(kind)                                   # 演出（v1+）

# server → client の論理イベント名（SSEで配送）
speaker_started
speaker_stopped
session_ended
model_load_failed
```

SSEはStreamable HTTP内で用いる。上記の論理イベント名だけをもってMCP標準通知メソッドや汎用MCP clientとの互換性が確定したとは扱わない。具体的なJSON-RPCメッセージ形式、SDKの対応、再接続時の扱いを設計する。[MCP transport仕様](https://modelcontextprotocol.io/specification/2025-11-25/basic/transports)

相手発話の本文をMCP経由で受け取れることは確定要求Cに含まれる。本文をどのイベントで通知するか、発話開始・停止とどのように対応付けるかは未確定。

## 6. 技術選定

| 部品 | 採用・想定 | 備考 |
| --- | --- | --- |
| Live2D描画 | `pixi-live2d-display-lipsyncpatch` | AITuberKit / OnAir系の実装を参考にする。採用バージョンとCubism 4系モデルの互換性は未検証。 |
| Cubism Core | 運営者が取得・配置 | `live2dcubismcore.min.js`を本リポジトリの公開成果物に同梱しない方針。出版・配布条件の確認は別途必要。 |
| スタジオ本体 | 自作。Vite + Svelte想定 | digital-souls frontendとの技術統一を意図する。 |
| メディア | LiveKit | 音声trackのみ。参加キャラクターはpublish-only、スタジオは音声を受信する。 |
| 参考実装 | AITuber OnAir `react-live2d-app`、AITuberKit外部連携v2 protocol | セッション限定モデル読込、口パク、protocol設計の参考。コードの転用範囲や互換APIの採用を確定するものではない。 |
| VRM（将来） | `@pixiv/three-vrm` | canvas重ねによる共存を想定。v0対象外。 |
| TTS（拡張用） | `@aituber-onair/voice` | 将来のホスト音声モード向け。v0のTTSは参加キャラクター側。 |

Core非同梱は本プロジェクトの配布方針であり、「Coreはあらゆる条件で再配布不可」「非同梱ならアプリ公開条件の確認は不要」という意味ではない。公式契約には再配布可能コードの条件と、拡張性アプリケーションの出版条件がある。任意モデルを読み込む本アプリへの適用、公開者・セルフホスト運営者それぞれの対応、サンプルモデルの使用条件は公開前に確認する。適用可否は本書では断定しない。[Live2D公式使用許諾契約（第1.5、2、5、6条）](https://www.live2d.com/eula/live2d-proprietary-software-license-agreement_en.html)

## 7. 将来拡張（プロトコルに余地を残す）

| 拡張 | 内容 |
| --- | --- |
| ホスト音声モード | キャラクターはテキストのみ送信し、スタジオ側でTTSも実施する。MCP接続と`model_url`だけで参加できる形を目指す。 |
| VRM対応 | Live2D / PNGに加え3Dモデルを扱う。 |
| LiveKit Egress直配信 | OBS Browser Source以外の出力経路。 |
| floor管理 | 発話権・順番・競合の制御。 |
| SNSログイン / 運営者ダッシュボード | 招待制・セッションスコープ認証を超える運用機能。 |
| `collab.react` | v1以降の演出。 |

## 8. 未決事項と確定済みの配置

| 項目 | 状態 |
| --- | --- |
| リポジトリ名・独立配置 | **確定**: `FYuki/aituber-collaboration`。本addon専用publicリポジトリ。 |
| 要求文書の配置 | ルートの`CONTEXT.md`（本書）。 |
| `model_url`の配布形式 | 未決: zip vs フォルダ列挙。署名の適用範囲、関連ファイルの取得方法も設計する。 |
| コラボプロファイルmanifest | 未決: 項目、バージョン、emotionマッピングの表現など。 |
| 実装ディレクトリ構成 / MCP serverの技術 | 未決。参加キャラクターCoreの構成から自動的に決めない。 |
| 公開する自作コードのライセンス | 未選定。public化だけでセルフホスト・再利用条件が定まったとは扱わない。 |

## 9. 設計時の確認事項（未決・要求FIXの変更ではない）

以下は確定要求を実装・検証可能にするための確認点であり、採用決定・ADR・追加機能の実装指示ではない。

| 項目 | 設計で確定・検証する内容 |
| --- | --- |
| レンダリングの実行場所 | OBS Browser SourceはOBS内のブラウザとしてWebページを実行する。サーバーがページを提供することと、サーバー上でGPU描画することを区別し、本構成での「中央」の意味を明文化する。サーバー常駐rendererや映像配信経路を暗黙に追加しない。[OBS公式](https://obsproject.com/kb/browser-source) |
| 非永続化の実装と保証範囲 | モデル・transcriptをDB、ファイル、一時ファイル、ログ、ブラウザ永続API等へ書かない構成を確認する。HTTP/CDN/proxy/ブラウザキャッシュ、Service Worker、展開処理、クラッシュダンプ、OS swapを含めた実行環境上の確認範囲と制約を明示する。アプリで保存APIを呼ばないことだけで不変条件の達成としない。`fetch`の`cache: "no-store"`はブラウザHTTPキャッシュ対策であり、全保存経路の保証ではない。[MDN](https://developer.mozilla.org/en-US/docs/Web/API/Request/cache) |
| モデル取得・検証 | CORS、署名URLの期限、モデル内相対参照、許可するリソース種別、サイズ上限、展開後容量、画像寸法、タイムアウト、読込失敗を扱う。サーバーが外部URLを取得する経路を採る場合はSSRF対策も必要。具体的な閾値は未決。 |
| URLの非露出と信頼境界 | `session_state`、LiveKit metadata、参加者向けイベント、字幕、例外ログから`model_url`や署名を排除する。一方、fetchを行うrendererはURLと資産へアクセスする必要があるため、その閲覧権限を持つホストと通常参加者の信頼境界を明文化する。権限のあるrendererからの資産コピー防止まで保証したとは扱わない。 |
| 認証・認可 | `join`引数の`session_token`表記とHTTP Bearer認証の責務を整理する。招待、参加者、ホスト、rendererの権限分離、自己アバター操作の認可、期限・失効・再joinを設計する。共有セッショントークンだけで個別参加者の本人性が確立したとは扱わない。 |
| LiveKit権限 | 参加者のpublish-onlyと音声限定を、SDKの接続設定だけでなくtoken grants・サーバー側の扱いで検証する。スタジオ用の受信権限を分離する。`canSubscribe`、`canPublish`、`canPublishSources`等の利用方法を確定する。[LiveKit公式](https://docs.livekit.io/frontends/reference/tokens-grants/) |
| 発話通知と音声の同期 | 発話本文の通知、発話ID、話者ID、trackとの対応、`utterance.report`の送信時点、開始・停止の判定主体を定める。`duration_ms`だけで実再生開始や終了が正確に分かると仮定しない。同時発話、音声未着・失敗、停止・切断、重複通知を検証する。floor管理は追加しない。 |
| セッションの寿命と復帰 | ホストによる作成・終了、招待、切断、期限切れ、スタジオページ再読込、SSE再接続時の動作とモデル再取得を設計する。再配送・`context.tail`のための保持はメモリ内に限定し、終了・プロセス再起動後の復旧を永続データへ依存させない。 |
| 公開・セルフホスト条件 | 自作コードのライセンス、依存ライセンス、Live2D出版条件、サンプルモデルの利用条件、TLS/TURN設定、秘密情報の配置を整理する。 |

## 10. v0検証観点（確定要求からの受入整理・未実施）

| 対応要求 | 確認する結果 |
| --- | --- |
| A・C・E | 招待された2名がMCPとLiveKit音声で参加し、OBS用ページで2-up描画・音声ミックス・ネームプレート・字幕ON/OFFが動作する。 |
| B | Cubism 4系サンプルと静止画PNGを読み込める。形式不正、上限超過、取得失敗時はプレースホルダーに切り替わり、会話は継続する。サンプルの取得・利用は各ライセンス条件に従う。 |
| C | 参加者が他者音声をsubscribeせず、STTなしで相手発話テキストを受け取れる。リップシンクは当該話者の音声に、字幕は報告テキストに対応する。 |
| D | 自キャラの意味レベルの指示がモデルのモーション・表情へ反映される。スタジオ既定の待機動作と共存し、他キャラを操作できない。 |
| F | セッション外・権限外の参加や操作を拒否する。外部参加者を含む試験ではTLS/TURN差分を検証する。 |
| B・G | モデル・transcriptが定義した検証範囲の永続保存先へ書き込まれず、URLが他参加者・配信・ログへ出ない。終了時にセッション内資産・履歴の参照を解放する。 |
| G | 光織側の連携で`collab` provenanceと既存memory policyを維持し、スタジオに長期記憶の責務を持ち込まない。 |

具体的なサイズ上限、性能閾値、イベントschema、例外時の詳細動作は未確定であり、設計で数値・仕様・試験手順を定める。

## 11. 記録状態

この文書は2026-09-23のユーザー提示要求を基準とし、指定リポジトリを確定済みとして反映した。Core非同梱と透過映像方式の不採用という方針は維持しつつ、ライセンスやalpha伝送一般に関する断定とプロジェクトの採用判断を区別した。第9・10節は設計・検証の整理であり、要件再FIXやADR承認を意味しない。

ADRは未起票。実装・動作検証・公開ライセンスの選定は未実施。
