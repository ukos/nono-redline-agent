# REDLINE AGENT — 試作 BUILD-NOTES（2026-11-23、v0／v1／v1.1／v1.2／v1.3／v1.4／v1.5）

v1 は 2026-11-23 に別途指示で公開済み。**v1.1・v1.2 はローカル検証まで（公開は別途）**。v1.2 は後に公開済み（2026-11-23）。v1.3・v1.4 も後に公開済み（2026-11-23）。**v1.5 はローカル検証まで（未公開）**。

---

## v1.5 (2026-11-23) — 停止中も横スワイプ可／移動中は武器をレディ保持／射撃ポーズ

Built locally, verified, **not published**. Version label: "REDLINE AGENT prototype v1.5".
実機（v1.4）フィードバック3件への対応。

### Change 1 — 横スワイプは「移動中」必須を廃止、全状態で有効に

- 停止中・中央レーン: 単発 ←/→ = その側の最寄りの物陰へ移動して入る（探索窓: 同側・|Δz| ≤ 7m、前方優先・なければ最も近い後方。新関数 `nearestCoverStopped`）。移動中と同じ transit／衝突解決（v1.2）を通る。範囲内に物陰がなければ「近くに物陰がない」を表示してその場に留まる（停止中はサイドステップ回避にフォールバックしない。回避は移動中のみ従来通り）。
- 停止中: ダブル ←←/→→ = 物陰に入ってから、ロックオン可能な対象がいれば射撃（いなければ物陰に入るだけ）。既存の A 方式（1回目で `enterCover` が `S.cover` を即時設定し `S.lastSide` を記録、350ms 以内の2回目が物陰内分岐で `shoot()`）をそのまま使う。物陰内でのダブル＝射撃は従来通り。
- 移動中／物陰内／レーン内の既存挙動はすべて不変。
- 文言: ヘルプ「移動中に ←→ 物陰へ」→「←→ 物陰へ（停止中も可）」。チュートリアルバナー「移動中に左右スワイプで物陰へ」→「左右スワイプで物陰へ」（開始カットイン後・1ラン1回、`S.hintShown`）。

### Change 2 — 移動中は武器を体の前にレディ保持（ready フレーム）

歩行2相ぶんの ready 版を衣装ごとに生成（各相の現行歩行フレームを参照）。パイプラインは v1.4 と同一（`process_v15_frames.py` → `normalize_v15_frames.py` で idle の図形高／足元基準線／キャンバス高に統一）。使用箇所は移動中のみ（歩行・ダッシュ・バックステップ・レーン／物陰 transit。transit とバックステップも「移動」に数え、後退時はサイクル逆順）。停止時と物陰アイドルは従来のリラックスした idle のまま。

生成の要点: 「体の前に保持」は後ろ姿では「両前腕が胴の側面を回って前方へ、手は背中に見えない、武器は銃身の先／ストックの尻だけが肩口・腰横に覗く」と描くのが正解だった（試行1は全滅: ライフルが背中に斜め掛けのまま／ピストルが横向きプロファイル）。

監査結果（武器が前に読めるか・武器種・髪型・脚相・足開き ≤0.45×hipW）:

| フレーム | 試行 | 判定 | 備考 |
|---|---|---|---|
| o1-ready-a | 2 | 合格 | 背中の斜めライフル消失。銃身が右肩の先、ストックが右腰。左脚上げ（walk-a 相） |
| o1-ready-b | 2 | 合格 | 銃身の先が右肩、ストックが左腰に覗く。右脚上げ（walk-d 相） |
| o2-ready-a | 3 | **不合格 → フォールバック** | 試行1=横向きプロファイル、試行2=両手が背中の中央、試行3=横向きプロファイル。A 相は通常の o2-walk-a を使用 |
| o2-ready-b | 2 | 合格 | 両手が右肩口の外側でピストル後部を保持。右脚上げ・灰色の靴底 |
| o3-ready-a | 1 | 合格 | 銃身が左肩の先、ストックが右腰。左脚上げ |
| o3-ready-b | 1 | 合格 | 同上・右脚上げ。元の歩行 B 相に無かったライフルを前方保持で追加 |
| o4-ready-a | 2 | 合格 | 試行1は弓が左側面に垂下＋右手が矢筒で不合格。試行2は弓の両端だけが左シルエット際に覗く |
| o4-ready-b | 2 | 合格 | 試行1は矢が背中を横断して不合格。試行2は弓の上端が左肩上・下端が右下に覗く（前方で斜め保持と読める） |

足開き監査（`audit_walk.py`、hipW は v1.4 と同一）: o1-ready-a 0.36 / o1-ready-b 単独成分（spread 0）/ o2-ready-b 単独成分 / o3-ready-a 0.44 / o3-ready-b 0.32 / o4-ready-a 単独成分 / o4-ready-b 0.36 — 全て合格。脚相は歩行フレームと成分パターンでも照合（o4-ready-b は walk-b とほぼ一致、o4-ready-a は walk-a の主成分と一致。弓先端の有無差は前方保持による想定内）。

### Change 3 — 射撃ポーズ（shoot フレーム）

衣装ごとに背面の射撃ポーズを1枚生成。発射時にスプライトを約0.28秒差し替え、マズルフラッシュは発射側の肩／頭の高さの前方（px ±0.35, 1.6, pz +0.55）。ポーズ表示中は従来のリーン・パルスを切る（`leanUntil = -1`）。ポーズが無い衣装は従来のリーンにフォールバックする設計（今回は全衣装合格したため未使用）。

| フレーム | 試行 | 判定 | 備考 |
|---|---|---|---|
| o1-shoot | 3 | 合格 | 試行1・2は銃身が右へ横向きプロファイルで不合格。試行3はストックが右肩、銃身は頭の脇に短く覗くだけ |
| o2-shoot | 1 | 合格 | ピストル後部（スライド背面）が右肩口の先に両手保持。前方を向いていると一読できる |
| o3-shoot | 1 | 合格（弱） | スコープ背面が頭の右脇に見える＝前方照準の強い手掛かり。銃身の付け根が短く横に出る |
| o4-shoot | 2 | 合格（弱） | 試行1は弓・矢とも横向きで不合格。試行2は弓の両端だけが覗き、羽根が右頬の脇（引き手位置）。羽根が頭のシルエットを少しはみ出す |

### 検証

- verify.mjs: バージョン表記 v1.5 に更新。v1.3 の歩行サイクル・サンプリング6件は Change 2 で移動中スプライトが ready 版に変わるため期待値を ready キーに変更（理由は上の通り。歩行フレーム自体の形状・足開き監査は従来ブロックで継続）。v1.3 input 節は、停止中左スワイプの結果が「近くに物陰がない」表示に変わったため期待文言を更新。
- 追加チェック: ready/shoot フレームの形状一致（flutter）と ready フレームの足開き監査、停止→右スワイプで右側の物陰（車 z=9）に入る、物陰なし（z=0）で留まりメッセージ表示、停止→ダブル右で物陰（車 z=30）＋射撃＋射撃ポーズ表示、停止中は idle 表示。
- 結果: **86 PASS / 0 FAIL / 0 errors**（73 件の既存チェック＋新規 13 件）。ログ: `shots/verify-v15.log`。
- ハーネス側のタイミング堅牢化（ゲームロジックは不変）: SwiftShader ではゲーム時計が実時間の約 1/5 に膨張するため、(1) ロケット警告の待機窓 30s→100s（警告は 60m 超過後 +5 ゲーム秒で発射される v1.2 仕様のまま。30s 窓は実時間換算で際どくなっていた）、(2) 歩行サイクルのサンプリング窓 2.9–5.2s→9–12s（サイクルは距離／ゲーム時間ベース）、(3) 停止ダブルスワイプ検証は2回のスワイプを単一 evaluate 内（間隔 60ms）で発行（evaluate 往復がメインスレッド停滞で実時間 350ms 窓を超える事象を観測したため）。いずれも初回・2回目の実行で実際にフレークとして観測したものへの対処で、検証内容自体は弱めていない。
- 目視ストリップ: `shots/v1.5-strip-ready-o1.png` 〜 `o4`（[idle, ready-A, idle, ready-B]、o2 の A 枠はフォールバックの walk-a）、`shots/v1.5-strip-shoot.png`（生成: `make_v15_strips.py`）。暗背景でフリンジ残りなしを確認。

---

## v1.4 (2026-11-23) — 開始カットインの目を小さく／歩行は脚を前後に（横開き廃止）

Built locally, verified, **not published**. Version label: "REDLINE AGENT prototype v1.4".
実機（v1.3）フィードバック2件への対応。サイクル構造・距離ベース2.2m/周・後退逆順・ボブなし・形状正規化は v1.3 のまま不変。

### Fix A — 衣装3・4の開始カットイン、目が大きい → o1 準拠に縮小（ピクセル編集）

実測（紫アイリスの縦幅）: o1-start ≒ 11–12px（図形高499）に対し o3-start 16px / o4-start 16px（図形高485）。

- **発見**: v1.3 生成の o3/o4-start は抽出時のポケット除去で目の内側に透明穴が残っていた（o3=238px、o4=250px。o1 は 0）。暗いカットイン背景では穴が黒い目の塊として読め、アイリス実測以上に目が大きく見えていた。
- **最終手法**（`fix_v14_eyes_final.py`、バックアップから冪等に再実行可）: (1) 目の内側の透明穴を、その目自身の白目色（青白画素の中央値）で充填。(2) 各目のボックスを 85%×70% に NEAREST 縮小し、NN 充填の上に貼付（o3 は上揃え dy=1、o4 は中央 dy=3）。
  - 白目充填: o3 近167px RGB(205,203,210)／遠71px (207,203,211)、o4 近206px (201,188,213)／遠43px (216,173,217)。
  - 結果: アイリス縦幅 16px → **11px**（両衣装とも o1 と同等）。
- **不合格だった試行**: 全ボックス再サンプル＋肌色充填（縞・瞑り目線が出て不合格）、目クラスタ分割（同）、メディア画像編集（o3 はアイリス81%止まり、o4 は約100%＝実質無縮小、全面再描画ドリフト。不合格品は `assets/rei-outfits/generated-v14/media-generation-{o3,o4}-start-smalleyes-*.webp` に保管）。
- 目ボックス (x0,y0,x1,y1): o3 (146,107,180,132)・(194,109,217,131)、o4 (166,108,198,132)・(215,112,232,132)。
- バックアップ: `assets/rei-outfits/frames-before-v14-eyes/{o3,o4}-start.png`。監査合成（o1参照｜編集前｜編集後）: `shots/v14-eyes-o3-ref-before-after.png` / `shots/v14-eyes-o4-ref-before-after.png`、目視合格（同一人物・同一表情、目だけ一回り小さい、滲み・崩れなし）。
- 検証にピクセル差分チェックを追加: 出荷 o3/o4-start とバックアップの差分は、目ボックス和集合＋4px の顔領域の**内側のみ**（外側 0 件であること）を自動確認。

### Fix B — 歩行の脚が横に開く（スーツは「タコみたい」）→ 全衣装で前後動に作り直し

全8相を idle を参照に1コマずつ生成し直し（`process_v14_frames.py`: フラッドフィル背景除去＋ポケット除去＋スペック除去＋idle 図形高へ統一＋デフリンジ）、v1.3 の形状正規化（`normalize_cycle_frames.py`）を再適用。ポーズ要件: 片脚は接地して真っ直ぐ、反対脚は膝を曲げて**後方へ持ち上げ靴底がこちらを向く**、両足は腰幅の内側・中心線近く。反転流用はなし（武器の側が入れ替わるため）。

**監査メトリクス**（`audit_walk.py`、verify.mjs にも移植）: hipW = idle の図形高59–65%帯で中心列を通る不透明ランの平均幅。足 = フレーム図形の下位12%帯の不透明成分（n≥150、幅15–120px。3個以上なら中心に近い2個）。spread = 両足中心の水平距離。**合格: spread ≤ 0.45 × hipW**。hipW: o1=171.9 / o2=159.1 / o3=190.4 / o4=205.6。

| フレーム | v1.3 spread（比） | v1.4 spread（比） | 判定 |
|---|---|---|---|
| o1-walk-a | 0.0（単一成分*） | 61.1（0.36） | 合格 |
| o1-walk-d | 205.8（1.20） | 0.0（単一成分） | 合格 |
| o2-walk-a | 135.7（0.85） | 0.0（単一成分） | 合格 |
| o2-walk-b | 129.7（0.82） | 0.0（単一成分） | 合格 |
| o3-walk-a | 188.9（0.99） | 83.8（0.44） | 合格 |
| o3-walk-b | 196.9（1.03） | 59.4（0.31） | 合格 |
| o4-walk-a | 0.0（単一成分*） | 66.5（0.32） | 合格 |
| o4-walk-b | 91.1（0.44） | 73.6（0.36） | 合格 |

- *単一成分 = 両足が帯内で1成分に融合、または持ち上げた足が帯より上にあるため帯メトリクスは 0 を返す。v1.3 の o1-walk-a / o4-walk-a はこの過小計測で「0」だったが、見た目は横開きだった（＝ユーザーの指摘どおり）。衣装ごとの最悪位相で比べると o1: 205.8→61.1、o2: 135.7→0、o3: 196.9→83.8、o4: 91.1→73.6 と全衣装で大幅に狭くなっている。
- o4-walk-a は持ち上げた靴底が帯の僅か上にあり、帯内の第2成分は弓の先端（cx 217.2）。ストリップ目視では靴底は接地足のほぼ真上＝本セットで最も狭いスタンス。
- **試行記録**: o1 A/B・o2 A/B・o3 B・o4 A/B は1回目で合格。**o3-walk-a は2回目**: 1回目は帯内に接地足＋ライフル銃口が入り、持ち上げた靴底が帯の上 → 110.6（0.58）で不合格。持ち上げを低くした2回目で靴底が帯に入り 84.2（0.44）合格（正規化後は 83.8）。o4 は生成器が A/B を逆に描いたため、内容（左脚持ち上げ／右脚持ち上げ）でファイルへ割当。
- **没にした監査変法**: 「腰スパン外の成分を足から除外する」変法は、v1.3 ベースラインの o1-walk-d で本物の足を除外し FAIL 205.8 を偽 PASS に変えたため不採用。比較は上下両版とも素の定義で統一。
- v1.3 歩行フレームのバックアップ: `assets/rei-outfits/frames-before-v14-walk/`（全8枚）。
- 歩行ストリップ（出荷テクスチャ、暗背景 [idle, A, idle, B]）: `shots/v1.4-walk-strip-o1/o2/o3/o4.png`、全て目視確認済み — 左右交互の後方リフト、足は中心線近く、星型・タコ型の開きなし。武器の側・髪型（o2 撫で付け、ポニーテールなし）も全フレーム維持。

### 検証結果（v1.4、puppeteer）

- **自動チェック 73件すべて PASS、console/page エラー 0件**（v1.3 の67件＋v1.4 新規6件: 歩行スプレッド監査4件（衣装ごと両位相）、目の差分配分チェック2件。バージョンラベルチェックは v1.4 表記に更新）。
- 幾何チェック（v1.3 から継続）: 全衣装で図形高・下余白・キャンバス高が完全一致（o1 528/5/540、o2 529/7/544、o3 540/7/554、o4 531/4/542）。
- 目の差分: o3-start は顔領域内 1,230px のみ変更・外 0px、o4-start は内 1,061px・外 0px。
- スクリーンショット: 歩行ストリップ4枚（上記）＋目の監査合成2枚（上記）。

---

## v1.3 (2026-11-23) — 全衣装2相歩行／ドタバタ解消／トレーサー残留修正／開始カットイン髪型修正／メッセージ中入力

Built locally, verified, **not published**. Version label: "REDLINE AGENT prototype v1.3".
実機（v1.2）フィードバック5件への対応。

### 1. 直立不動の解消 — 不足していた背面歩行フレームを生成（全衣装2相化）

元シートに背面歩行コマが無い衣装について、既存フレームを参照画像にして1コマずつ画像生成し、厳密監査（背面厳密／脚相／武器の側・向き／髪型／色・ディテール／暗背景での白フリンジなし）を通ったものだけを組み込んだ。生成→抽出は `process_v13_frames.py`（四隅サンプルのフラッドフィル＋囲まれた白ポケット除去＋デフリンジ＋idleの高さへ統一）。

| フレーム | 試行 | 判定 | 備考 |
|---|---|---|---|
| o2-walk-a（左脚前） | 1 | **合格** | 背面厳密、右手にP38、撫で付けウルフカット |
| o2-walk-b（右脚前） | 1 | **合格** | 同上。a と明確に逆位相 |
| o1-walk-d（右脚前の接地相） | 3 | **合格（3回目）** | 1回目＝中間相＋ブーツ横向きで不合格、2回目＝ほぼ直立で不合格、3回目＝両足接地のワイドな接地相で合格。生成物が描いた地面の影バーは処理で除去 |
| o4-walk-b（walk-a の逆位相） | 1 | **合格** | 弓・矢筒の側は idle と一致。弓脇の白ポケットは抽出時に除去 |

- サイクルは全衣装 `[idle, A, idle, B]`（距離ベース1周2.2m、後退は逆順）に統一：
  - 衣装1: `[o1-idle, o1-walk-a, o1-idle, o1-walk-d]`（o1-walk-c＝walk-a と同脚の中間相はサイクルから引退）
  - 衣装2: `[o2-idle, o2-walk-a, o2-idle, o2-walk-b]`（v1.2 は idle のみ＝直立不動だった）
  - 衣装3: `[o3-idle, o3-walk-a, o3-idle, o3-walk-b]`（不変。walk-b にライフルが無いのは元絵どおり＝捏造しない）
  - 衣装4: `[o4-idle, o4-walk-a, o4-idle, o4-walk-b]`
- 歩行ストリップ（実テクスチャ、暗背景）: `shots/v1.3-walk-strip-o1/o2/o3/o4.png`、全て目視確認済み。衣装2の実機内歩行: `shots/v1.3-o2-walking.png`。

### 2. 衣装1の歩行ドタバタ — フレーム形状の正規化＋ボブ廃止

ゲームはビルボード高さをキャンバス高で固定し幅をアスペクト比で決めるため、見かけの背丈＝2.05×(図形高/キャンバス高)、足元位置＝2.05×(下余白/キャンバス高)。全衣装のサイクル内フレームで「図形高・下余白・キャンバス高」を idle に完全一致させた（`normalize_cycle_frames.py`、一様スケールのみ・引き伸ばしなし。修正前は `frames-before-v13-normalize/` に保管）。

| 衣装 | フレーム | 図形高 before→after | 下余白 before→after | キャンバス高 before→after |
|---|---|---|---|---|
| o1 | idle / walk-a / walk-d | 528 / 528 / 520 → **528** | 5 / 5 / 17 → **5** | 540 / 540 / 544 → **540** |
| o2 | idle / walk-a / walk-b | 529 / 531 / 531 → **529** | 7 / 7 / 7 → **7** | 544 / 545 / 545 → **544** |
| o3 | idle / walk-a / walk-b | 540 / 535 / 536 → **540** | 7 / 8 / 7 → **7** | 554 / 550 / 550 → **554** |
| o4 | idle / walk-a / walk-b | 531 / 535 / 533 → **531** | 4 / 0 / 7 → **4** | 542 / 542 / 547 → **542** |

- あわせて v1.2 の歩行ボブ（|sin| の上下動 0.03m）を廃止。全衣装に実位相が揃った今はフレーム側が動きを持つため、ボブは脈動の原因にしかならない。
- 検証: 幾何チェック4件（衣装ごとに図形高・下余白・キャンバス高が ±1px で一致）を自動化し PASS。

### 3. 弾道の線が残る — トレーサーの寿命管理を実時間化

- **根本原因**: FX の除去判定がゲーム内時計 `S.time` 基準だった。`S.time` はプレイ中しか進まずリセットで 0 に戻るため、ラン終盤に出たトレーサー（until≈45s）は死亡・リスタート後に次ランの時計が追いつくまで画面に残り続けた（ユーザー撮影の「木の近くの黄色い点まで伸びる橙色の線」はこれ）。
- **修正**: 全 FX（トレーサー／マズル／ロックオンマーカー）を `performance.now()` 基準の実時間デッドラインに変更（可視寿命は最大 0.15s にクランプ、残り時間でフェード）。除去ループはフェーズ不問で毎フレーム実行。さらに `resetState()` で全 FX を強制パージ＋レーザー消去。射手・対象が途中で死んでも、物陰状態が変わっても、寿命で必ず消える。
- 検証: 実発射経路（`RLA.debugTracer`）でトレーサーを出し、直後は存在・0.6s 後は Line オブジェクト 0 件／FX 0 件を確認。

### 4. 衣装3・4の開始カットインがポニーテール — 差し替え

- 監査結果: ポニーテールなのは **o3-start / o4-start のみ**。o1/o2 の start・clear、o3/o4 の clear は全てほどいたウルフカット（目視確認）。
- 差し替えを生成（各衣装の clear＝顔・髪の同一性、idle＝衣装、o2-start＝構図を参照）。**両方とも1回目で合格**: 膝立ち振り返り、ほどいたウルフカット（結び目なし）、衣装3は迷彩フェイスペイント維持。
- 処理は `process_v13_starts.py`（図形高 485px ≒ 旧 483px に統一、デフリンジ）。旧ポニーテール版は `frames-before-defringe/o3-start-ponytail.png` / `o4-start-ponytail.png` に保管。

### 5. メッセージ表示中に操作不能 — 入力経路の堅牢化

- 調査: `#hud` は元々 `pointer-events:none` で、メッセージ自体はタッチを奪っていなかった。実機で効かなくなる要因として次を特定し対処:
  1. **`touch-action` 未指定**: ブラウザがジェスチャをスクロール等と解釈すると `touchend` でなく `touchcancel` が来てスワイプが丸ごと消える。→ `#app` に `touch-action:none` を追加し、`touchmove` で座標を追跡、`touchcancel` でも最後の移動量でスワイプを成立させるようにした。
  2. **物陰移動（coverAnim）の到達判定が素の目標点基準**: ロケットで車が裏返る等で目標点が占有フットプリント内に入ると、衝突解決後の位置が目標に永遠に届かず移動が終わらない（前進入力が死んで見える）。→ 到達判定を「衝突解決後の目標点」基準に変更し、セグメント 1.2s／全体 3.5s のウォッチドッグで必ず遷移を終えるようにした。
  3. 念のため `#msg` にも明示的に `pointer-events:none`。
- メッセージは従来どおり自動消滅のみ（閉じる操作なし・ゲームは止まらない）。
- 検証: チュートリアル文「移動中に左右スワイプで物陰へ」表示中に↑スワイプ→前進開始を確認。60m 超えで「ロケット接近！」表示中に↓スワイプ→警告が表示されたまま（opacity 1）入力に応答することを確認。

### 検証結果（v1.3、puppeteer）

- **自動チェック 67件すべて PASS、console/page エラー 0件**（v1.2 の56件＋v1.3 新規11件: 衣装2後退サイクル、幾何4件、トレーサー2件、入力4件）。歩行サンプリングは全衣装で前進・（衣装1・2は）後退とも両位相の出現を確認。
- スクリーンショット: 上記ストリップ4枚＋ `shots/v1.3-o2-walking.png`（衣装2が実際に脚を動かして歩行中）。

---

## v1.2 (2026-11-23) — collision / walk cycle / car variety / rockets / lane rules

Built locally, verified, **not published** (publishing is a separate step). Version label: "REDLINE AGENT prototype v1.2".

### Fix 1 — never sink into objects (collision)
- Root cause of the v1.1 bug (Rei inside a car, only head/feet visible): the cover spot was computed as `obj.z − 1.8`, but a regular car's half-depth is 2.15 m — the "cover position" was *inside* the car body.
- Every solid object now has an oriented footprint `{cx, cz, hx, hz, rot}` (cars and street trees; trees are circles approximated by a 0.35 m square). Collision uses an OBB test in the object's local frame. The player (radius 0.25 m) is pushed out along the least-penetration axis after **every** movement integration: walk, dash, backstep, lane retreat, cover transit and centre drift all pass through the same resolver. Street clamp |x| ≤ 5.55 also applies.
- Cover spots are derived from the footprint: spot = centre − (local z extent + player radius + 0.18 m) on the player's side, so the sprite is always fully outside the mesh and occluded from the enemy side.
- Cover entry / cover-to-cover transit uses a path: out to a clear corridor (|x| = 1.6, inside every footprint's inner edge), along the street, then into the spot — the player never cuts through a car to reach cover.

### Fix 2 — no shrink in cover
- The cover squash (sprite scale-y ×0.82) is removed. In cover the sprite keeps full height (verified: scale stays at the full 2.05 world height in cover); hiding is expressed by position/occlusion only. The brief shooting lean (scale 1.04 pulse) is unchanged.

### Fix 3 — walk-cycle leg phases (audit of the v2 sheets, top row)
Every top-row figure of all four v2 sheets was extracted and classified before changing anything:

| Outfit | Back-view figures found | Leg phases available | Cycle used in v1.2 |
|---|---|---|---|
| 1 紳士 | standing; contact phase A (cell 2, fig 1); a second figure in cell 2 that is the *same* phase as A (pixel-diff ≈ duplicate); **passing phase** (cell 1, fig 2 — one leg grounded, other raised behind) | standing / contact / passing — **no opposite-leg contact frame exists in the art** | standing → A → standing → passing (both legs appear in turn; honest best with the art that exists) |
| 2 スーツ | standing only — the two walk cells are **side views** (profile face visible) | none | standing only, no bob (no back-view walk art exists) |
| 3 迷彩 | standing; stride A (right leg forward); stride B (left leg forward) | full two-phase | standing → A → standing → B (full cycle). Note: in the source art, stride B has **no rifle drawn** — the rifle disappears for that one frame; kept as-is rather than fabricating |
| 4 狩人 | standing; one stride phase | single phase | standing ↔ step, no bob |

- New frames extracted for v1.2: `o1-walk-c.png` (passing phase), `o3-walk-a.png`, `o3-walk-b.png`, `o4-walk-a.png` (corner-sampled background removal + defringe, same method as the corrected o1-idle). The old `o1-walk-b` (duplicate phase) is retired from the cycle.
- Cycle timing is distance-based: one full cycle per 2.2 m (step length 0.55 m), so cadence scales with speed (walk ≈ 4 steps/s, dash faster). Backward movement plays the same cycle in reverse order. Vertical bob (0.03) only for outfits with a full two-phase cycle; single/no-phase outfits get no bob so they don't fake a cycle.
- Outfit-select cards use the same frames as the game; start/clear cut-ins unchanged.

### Car variety
Types (same multi-part construction: body, cabin/cargo, wheels, head/tail lights; front = +z local, lights rotate with the car):
- **regular** 4.3 m (as v1.1), **compact** 3.4 m (narrower/lower), **van** 5.6 m (tall box body), **truck** 6.0 m (cab + cargo box).
- Right column (all cover-capable, 18): z=9 regular, 24.5 compact, 30 regular, 45.5 van, 51 regular, 66.5 **sideways regular (90°)**, 72 compact, 87.5 **diagonal compact (+35°)**, 93 regular, 108.5 **diagonal regular (−40°)**, 114 van, 129.5 truck, 135 regular, 150.5 compact, 156 regular, 171.5 **sideways compact (90°)**, 177 regular, 192.5 truck.
- Left column: street trees only (as v1.1, all cover-capable).
- Decorative cars (obstacles, **not** cover): (5.05, 35) regular, (4.6, 141) compact +20°, (−4.95, 98) van, (5.05, 160) regular.
- Collision and cover spots use each car's real oriented footprint (OBB), so sideways/diagonal cars have correct no-entry zones and cover spots that follow their angle. The centre lane (|x| ≲ 3) is always passable for the full 200 m. Note: the sideways regular (66.5) and diagonal regular (108.5) overhang the road edge (|x| = 6) by ~0.2 m onto the sidewalk — cosmetic only.

### Enemy rockets + car flips
Chosen values: rockets only from **z ≥ 60 m**, global cooldown **18 s**, rocket speed **14 m/s**, telegraphed by a 0.5 s aim/flash at the firing enemy; HUD warning 「ロケット接近！」 on launch. The rocket flies straight at the player's position at fire time, with a smoke-puff trail and glowing head, and is collision-tested step-by-step against every footprint.
- **Hits a car** (incl. the player's own cover car, or any car in the path): explosion; the car is launched — impulse up + away from the blast (vy ≈ 10–11.5 m/s), spin on pitch/roll/yaw, gravity 20 m/s², one bounce at 35% energy, then settles **upside-down or upright ≈ 50/50** with a new yaw. Landing spot is searched to stay on the street, out of other footprints, and never on the player. Footprint + cover spot update to the new pose; an upside-down car is still an obstacle and still cover-capable. If it was the player's cover, cover/lane state clears with **no damage**.
- **Direct hit on an uncovered player**: 1 damage (same as a bullet). **Tree/building/miss**: explosion effect only. Only cars flip.
- Stage reset restores every car to its original pose.

### Lane rules (edge lanes = cover columns)
- Three lanes: centre, left edge, right edge. Being in cover (or transiting between covers) puts the player in that side's edge lane; the lane persists after stepping back within it.
- **Dash is centre-lane only**: ↑↑ in an edge lane is just a normal forward move.
- In cover, ↑ advances to the **next cover ahead on the same side** and ↓ returns to the **previous** one, staying in cover the whole transit (HUD shows 物陰（左/右）while transiting too). ↓ at the first cover (or ↓ while in a lane but not in cover) steps back within the lane and drops cover state.
- **Protection**: while in an edge lane (at a cover or in transit), enemies on the **opposite** side cannot hit the player — normal shots are blocked (tracer stops at the cover row), sniper aim is blocked across the street, and opposite-side rockets target the player's cover car instead of the player. Same-side enemies hit normally.
- Leaving the lane: swipe toward the centre exits to the centre lane (existing behaviour). **Backstep (↓↓) in a lane**: performs the normal ~2 m hop backward, drops cover state, and the player **stays at the lane's x** (lane membership kept; only cover state drops). Documented choice, per spec's request.

### Verification (verify.mjs, headless Chromium)
**56 checks, all PASS, 0 console/page errors.** Includes everything from v1.1 (31 checks) plus:
- Systematic collision: the player is driven into **every** cover object from 4 directions (108 drives) — never inside any footprint expanded by the player radius; dash clamps at a car's face (pz = 6.60 exactly); backstep trajectory sampled — never inside; cover snap at a diagonal car lands outside its OBB.
- Cover sprite height = full size in tree cover and car cover (Fix 2).
- Lane rules: ↑↑ in a lane does not dash (centre dash still works); ↑/↓ cover-to-cover advance/retreat keep cover state and lane; opposite-side sniper shot blocked (no damage), same-side sniper shot hits.
- Rockets: forced rocket at the player's cover car → car leaves the ground, flips, settles displaced/rotated with `flipped` recorded, footprint changed, player unharmed; direct rocket hit on an uncovered player deals exactly 1 damage.
- Walk cycles: sprite sampling while walking forward/backward confirms outfit 1 shows standing + both step figures in both directions, outfit 3 shows both stride phases, outfit 4 alternates standing↔step, outfit 2 stays on idle (honest fallback).
- Test note: segments that assert no damage run with a verification-only god-mode hook (`RLA.setGod`) because ambient same-side fire during the long teleports/transits otherwise randomly kills the player and cascades; all damage assertions run with it off in fresh page loads.
- Screenshots: `shots/v1.2-01-cover-car.png` (diagonal-car cover, full height, fully outside), `v1.2-02-car-flip-mid.png` (car airborne mid-flip with explosion), `v1.2-03-car-settled.png`, `v1.2-04-gameplay.png`, `v1.2-05-variety-a.png` (sideways car), `v1.2-06-variety-b.png` (diagonal car + van + truck), `v1.2-walk-strip-o1.png` / `v1.2-walk-strip-o3.png` (extracted cycle frames side by side).

### Still missing / honest gaps
- Outfit 1 has no opposite-leg *contact* frame in the source art — the passing-phase frame substitutes; outfit 2 has no back-view walk art at all; outfit 4 has a single step phase; outfit 3's stride-B frame lacks the rifle in the original art. All four would need new art from the image AI to fix properly (no mirroring — it would flip the weapon shoulder — and no regeneration, per standing rules).

---

## v1.1（2026-11-23）— ロゴスタート画面／ホーム画面アイコン／新操作／ステージ質感／デフリンジ

### スタート画面（新規・最初の画面）
- フローを `start（ロゴ）→ outfit（衣装選択）→ cutin（作戦開始）→ play` に変更。旧 title フェーズは廃止し、スタート画面に統合。
- スタート画面は黒背景に採用ロゴ（logo.jpg）を大きく中央表示（幅は可視幅の最大92%）。ロゴ自体が黒背景なので継ぎ目なく溶け込む。
- ロゴの下に「タップではじめる」をクリーム色（#ffe9c9）で表示し、下に十分な余白を確保（safe-area 対応）。
- 右下にバージョン表記「REDLINE AGENT prototype v1.1」を、タップ文字と同じ #ffe9c9 で表示。
- クリア／ゲームオーバー後のタップはスタート画面に戻る。
- 検証中に発見して修正：タップ後に発火する互換マウスイベントで画面遷移が二重に進む問題（クリア→スタートを素通り）。touchend 後700ms以内の mouseup を無視する重複排除を追加（実機でも起こり得た問題）。

### ホーム画面アイコン（PWA）
- logo.jpg（1024×1024）から生成：`apple-touch-icon.png`（180×180）、`icon-192.png`（192×192）、`icon-512.png`（512×512）。ロゴは全面ベタのまま（黒背景がロゴ絵柄と一致）。
- `manifest.webmanifest`：name／short_name「REDLINE AGENT」、display standalone、background_color／theme_color #000000、start_url "."、icons は 192／512（purpose "any"）。
- `<head>` に manifest、apple-touch-icon（180）、icon（192）、apple-mobile-web-app-capable "yes"、status-bar-style "black"、apple-mobile-web-app-title、theme-color #000000 を追加。`<title>` は「REDLINE AGENT」。

### 新操作（ダブルスワイプ、既存の350msウィンドウを共用）
- **↑↑ ダッシュ**：2回目の上スワイプで発動。前進速度 4.8m/s（歩行2.2m/sの約2.2倍）で 0.7秒間、その後通常歩行に戻る。停止中からでも発動し、そのまま前進する。HUD状態表示は「ダッシュ」。
- **↓↓ バックステップ**：2回目の下スワイプで発動（単発↓の「停止／後退」より優先）。約2mを 0.25秒で後方へホップし、停止で終わる。HUD状態表示は「バックステップ」。
- 無敵時間はなし。物陰から出れば通常どおり露出する（カバー判定は不変）。
- パラメータは `DASH_SPEED=4.8`、`DASH_S=0.7`、`BACKSTEP_DIST=2.0`、`BACKSTEP_S=0.25`（仮置き）。
- HUDヘルプに「↑↑ ダッシュ　↓↓ バックステップ」を追加。RLA フックに `dashing`／`backstepping` フラグを追加。

### デフリンジ（フレームの白い縁取り除去）
- ライブ v1 のスクリーンショットで、レイのフレーム PNG の周囲に明るいハロー（白い縁取り）が暗背景で目立つ問題を確認。
- `defringe_frames.py` で全14フレームを処理：アルファの二値化＋マスク1px erosion＋明るい縁ピクセルの除去。**`extract_frames.py` は再実行禁止**（縁取りが戻る）。
- 修正前フレームは `assets/rei-outfits/frames-before-defringe/` にローカル保管のみ。**公開対象に含めない**。
- v1.1 の衣装選択／ゲーム中スクリーンショットで、暗背景でも白い縁取りが消えていることを目視確認済み。

### ステージの質感向上（単一ファイル Three.js のまま、夜の雰囲気を維持）
- **ビル**：手続きキャンバステクスチャを刷新。窓グリッドに暖色／寒色の点灯バリエーション、消灯・薄明かりの反射も混在。ベース色の異なる6種を用意し、建物ごとに高さ・幅・奥行きと色にばらつき。一部のビルに屋上のペントハウスブロックを追加して屋根ラインに変化。基部は暗くして夜に窓が読めるようにした。
- **車（物陰）**：下部ボディ＋肩ライン＋暗いガラスのキャビンの組み合わせに変更。車輪4つ（暗色シリンダー）、ヘッドライト（淡色・発光）とテールライト（赤・発光）を追加。塗色は MeshStandardMaterial（metalness 0.55／roughness 0.38）で艶を付け、色バリエーションは従来どおり。 footprint と物陰判定は従来と同一。
- **街路樹（物陰）**：幹＋大小5つの暗緑色の葉塊（色にばらつき）で立体感を出した。位置・物陰判定は不変。
- **道路・歩道**：アスファルトテクスチャにノイズと補修跡、両側の縁ライン（白）を追加。歩道は平面から高さ0.12mのボックスに変更し、スラブの目地入りテクスチャで一段高く見せる（z-fighting 回避）。
- **街灯**：約21m間隔で左右交互に配置。ポール＋アーム＋発光ヘッド＋グロー用スプライト（加算的な光晕はスプライトで代用し、PointLight は増やさない＝モバイル性能配慮）。
- **接地感**：シャドウマップは使わず、blob shadow（暗い楕円）をレイ・敵・車・街路樹の下に追加。特にレイは影なし問題が解消方向。
- **敵**：従来の単純フィギュアに暗い髪キャップを追加し、エージェントらしいシルエットに微調整（優先度は低く、最小限）。
- ライティングは既存の hemisphere＋directional とフォグを維持し、窓と街灯の発光を主なアクセントにした。

### 検証結果（2026-11-23、puppeteer、480×860）
- **自動チェック 31件すべて PASS、console/page エラー 0件**（v1 の20件＋v1.1新規11件：スタート画面表示とロゴ読込／バージョン表記が完全一致／マニフェスト取得・JSON・名前／アイコン192・512の記載／apple-touch-icon リンクと180px読込／icon-192 読込／スタートのタップで衣装選択へ／ダッシュ状態と高速移動（≥1.6mバースト）／バックステップ状態と約2m後退→停止／クリア後のタップでスタート画面へ復帰）。
- スクリーンショット（全て目視確認済み）：
  - `shots/v1.1-01-start.png` スタート画面（ロゴが黒に溶け込み、タップ文字の下に余白、右下にバージョン表記）
  - `shots/v1.1-02-outfit-select.png` 衣装選択（4衣装とも暗背景で白い縁取りなし＝デフリンジ確認）
  - `shots/v1.1-03-gameplay.png` ゲーム中（窓の点灯バリエーション、街灯のグロー、縁ラインと歩道、レイと街路樹の blob shadow、ヘルプの新操作表示）
  - 従来の 01〜06、v1-01〜05 も再撮影済み。

### 残る点（v1.1 時点）
- 車は部品構成にはなったが、夜間＋このカメラ距離ではキャビンが大きな黒い箱に見えがち。丸みは限定的。
- 街灯のグローはスプライト代用なので、路面への光の広がりはない。
- カットイン拡大時の輪郭は、デフリンジで v1 時点のフリンジより改善しているはずだが、拡大表示専用の再確認は実機で行いたい。

---

## v1（2026-11-23）— レイ組み込み

### 変更内容
- **衣装選択画面を新設**（起動直後、タイトルの前）。4衣装を背姿（back-idle）の切り抜き＋名前で表示し、タップで選択 → タイトルへ。キーボード 1〜4 でも選択可。
  1. 英国紳士＋ライフル／2. ダークスーツ＋ワルサーP38／3. 海兵隊迷彩＋M40／4. 中世の狩人＋弓
- **ゲーム内の主人公をボックス人型からレイのビルボードスプライトに差し替え**。使用フレームは選択衣装の **back-idle（シート左上パネル）** のみ。高さ約2.05m、テクスチャのアスペクト比で横幅を決定。
  - 歩行：脚フレームアニメの代わりに上下ボブ（±0.055m）＋スケールパルス（±1.5%）。**衣装1だけは実物の後ろ姿歩行フレーム2枚（o1-walk-a/b）を交互表示**（下記抽出参照）。
  - 物陰：スプライトを物陰側へ移動（従来どおり）＋高さスケールを 0.82 に滑らかに縮小（しゃがみ表現）。
  - 射撃：0.18秒の前傾（前方へ0.12m＋スケール1.04倍）とマズルフラッシュ（従来）で表現。
- **ステージ開始カットイン**：ラン開始時に選択衣装のパネル7（膝立ち振り返り目配せ）を大きく表示＋「作戦開始」＋衣装名、約1.6秒でフェードしてゲーム開始。リスタート時も同じ流れ。
- **ステージクリア画面**：選択衣装のパネル8（立ち姿・少し振り返り）を既存のクリアテキストと併せて表示。
- フロー変更：`outfit → title → cutin → play`（RLA フックに `outfit`（1始まり）と `sprite`（現在のテクスチャキー、例 `o2-idle`）を追加、`RLA.selectOutfit(i)` を追加。承認パラメータ・ゲームロジックは無変更）。

### 使ったフレーム／あえて使わなかったフレーム
- 使用：全衣装の idle（左上）・start（下段3番目）・clear（下段4番目）、衣装1の後ろ姿歩行×2。
- **使っていない（意図的）**：各シートの歩行・物陰・射撃パネルの大半は**横向き**であり、後ろ姿スプライトとして使うと嘘になるため、ゲーム内には組み込んでいない。横向きパネルの後ろ姿化は今回の作業範囲外（未修正のまま）。

### 抽出方法（`extract_frames.py`、再実行可能）
- 対象：`assets/rei-outfits/rei-outfit{1..4}-*-sheet-v2.webp`（各2240×1120）。
- シートのグリッド線は公称560px間隔と一致しない（実測：o2 横線 y=566、o3 y=581、o4 y=564、縦線も ±20px 程度ずれる）ため、**線の位置を画像から検出**（背景色からの距離が大きい画素の割合が最大の行・列）してセルを切り分けた。
- 背景除去：セル全体の中央値＝背景色とし、セル端からのフラッドフィル（領域拡張）で背景を透明化。さらに（a）腕と脚の間など端から届かない**背景色ポケット**を「背景色にほぼ一致しフラッド領域に接しない成分」として除去、（b）セル境界に残る**グリッド線の残骸**を「端に接する小成分」として除去、（c）輪郭のフリンジ1pxを半透明化。
- 衣装1の上段2・3セルは後ろ姿が2体ずつ入っているため、透明列のギャップで個体に分割し、歩幅の異なる2ポーズを walk-a／walk-b とした。
- 成果物：`assets/rei-outfits/frames/` に14枚（o{1..4}-idle／start／clear、o1-walk-a／b）。

### 検証結果（2026-11-23、puppeteer、480×860）
- **自動チェック 20件すべて PASS、console/page エラー 0件**（従来16件＋v1新規4件：衣装選択画面の出現／衣装2選択でスプライトが `o2-idle` に切替／開始カットイン表示／クリアカットインが `o2-clear` で表示）。検証は最終フレームで2回実行し両方 PASS（2回目は射撃が命中し撃破1も確認）。
- 追加確認：衣装1で歩行中に `o1-walk-a`／`o1-walk-b` の両フレームが交互に使われることを状態フックで確認（エラー0）。
- スクリーンショット（全て目視確認済み）：
  - `shots/v1-01-outfit-select.png` 衣装選択（4衣装とも背姿・透過で表示）
  - `shots/v1-02-start-cutin.png` 開始カットイン（衣装2、膝立ち振り返り＋「作戦開始」）
  - `shots/v1-03-gameplay-rei.png` ゲーム中（衣装2の背姿スプライト、背景透過・縁の滲みなし）
  - `shots/v1-04-clear-cutin.png` クリア画面（衣装2の振り返り＋MISSION CLEAR）
  - `shots/v1-05-gameplay-o1-walk.png` 衣装1の歩行フレーム（ライフルを背負った後ろ姿・歩幅ポーズ）
  - 従来の 01〜06 も v1 で再撮影済み。

### 残る絵の問題（v1 時点）
- カットイン拡大表示では、輪郭に**1pxの明るいフリンジ**がうっすら見える（半透明化した縁。ゲーム内サイズでは視認できない）。
- 主人公スプライトには影がないため、地面への接地感はボックス時代より弱い（歩行ボブで補っているが、実機で要確認）。
- 横向きパネル（歩行・物陰・射撃）の後ろ姿化は未着手 ＝ 造形側の残件として据え置き。
- 敵・車・街路樹・ビルは v0 の仮素材のまま。音も未実装のまま。

---
以下は v0 の記録（そのまま保持）。

## 成果物
- `index.html`（単一ファイル、約632KB）— ゲーム本体
- `logo.jpg` — 採用ロゴ（16bit版）を企画フォルダからコピー
- `shots/` — 検証スクリーンショット6枚
- `verify.mjs` — 検証スクリプト（puppeteer、要 `~/workspace/tools/shots` の node_modules）

## レンダリング方式
- **Three.js WebGL（r128）**。ただし CDN（cdnjs）版はヘッドレス Chromium の検証環境で SSL エラーにより取得できなかったため、**同一ライブラリ・同一バージョンを index.html 内にインライン同梱**して自己完結化した（FIELD GOLF が使うのと同じ r128）。実機ブラウザでは CDN でも動くはずだが、同梱版はオフラインでも動くのでそのまま採用。
- プレイ領域は FIELD GOLF と同様に visualViewport に合わせてリサイズ（controls が画面外に出ない方式）。

## 承認パラメータ（v0.2）の実装状況
| # | 承認値 | 実装 |
|---|---|---|
| 1 | 誤認防止 A案：1回目で即・物陰、350ms以内の2回目で物陰からロックオン射撃 | `DOUBLE_MS=350`。左右スワイプ共通処理 `handleSide()`：1回目で即カバー移動開始＋時刻記録、2回目が350ms以内なら射撃に転換。物陰に居る間の同方向ダブルスワイプでも射撃可 |
| 2 | ロックオン優先：赤い線の敵 ＞ 近い敵 ＞ 中央寄り | `pickTarget()` のソート順そのまま。第1キー＝スナイパーが aim 状態か、第2キー＝距離、第3キー＝\|x\| |
| 3 | 敵回避率30%、回避後0.8秒は必中、スナイパーは回避せず発砲後に物陰 | `DODGE_RATE=0.30`、`DODGE_GUARANTEE_S=0.8`（`dodgeUntil` で管理）。スナイパーは命中即確定、発砲後 `incover` 状態へ |
| 4 | 赤い線：警告1.2秒、最終0.2秒で太く／明るく、物陰で遮断 | `SNIPER_WARN_S=1.2`、`SNIPER_FINAL_S=0.2`。レーザーは円柱メッシュで太さ変更可能に実装（0.018→0.05、opacity 0.8→1.0）。物陰中は線が物陰オブジェクト位置で止まり、被弾しない |
| 5 | 試作ステージ：約90秒の一本道、通常敵1→通常敵2→スナイパー1→複合、車と街路樹を交互配置 | 全長200m ÷ 前進2.2m/s ＝ 約91秒。敵配置：z=38 通常（左）／z=82 通常（左）＋z=86 通常（右）／z=128 スナイパー（右）／z=168 通常（左）＋z=172 スナイパー（右）、計6体。物陰は車・街路樹を約10.5m間隔で左右交互＋反対側ペア配置 |
| 6 | ライフ3、クリアはゴール到達、全滅は任意目標として表示のみ | ライフ♥×3、0でミス→最初から。ゴール（z=200のゲート）到達でクリア。HUDに「撃破 x / 6」を常時表示、クリア画面にも明記 |

## 操作系
- スワイプ（タッチ／マウスドラッグ）：上＝前進、下＝停止／停止中は後退、移動中の左右＝物陰、同方向2回＝射撃
- キーボード（デスクトップ検証用）：矢印キー／WASD、スワイプと同一の状態機械を通る
- 状態機械：`stop / forward / back` ＋ `cover{side}`。物陰中の下スワイプは「物陰を出て停止」（試作上の解釈、仕様未規定）

## 検証結果（2026-11-23、puppeteer、480×860 portrait、deviceScaleFactor 2）
- **自動チェック 16件すべて PASS、console/page エラー 0件**
  - タイトル表示→タップで開始、前進／停止／後退の状態遷移（全パターン）、移動中左スワイプ→物陰（左）確定
  - 350ms以内のダブルスワイプ→射撃発生、命中で撃破カウント増加（別試行では敵の回避＝dodging 状態も確認）
  - スナイパーが aim 状態に入り赤い線が表示されること、ゴール到達→クリア画面
  - 別試行では敵弾による被弾（ライフ3→2）も自然発生し、ダメージ処理の動作を確認
- スクリーンショット（全て目視確認済み）：
  - `shots/01-title.png` ロゴ＋「タップではじめる」
  - `shots/02-walking.png` 前進中（ビル街・車・街路樹・HUD）
  - `shots/03-in-cover.png` 物陰（左）＝街路樹の陰、状態表示も「物陰（左）」
  - `shots/04-lockon-shot.png` ロックオンマーカー＋トレーサー＋「HIT!」、敵が倒れている
  - `shots/05-sniper-redline.png` スナイパーの赤い線が主人公まで届いている
  - `shots/06-clear.png` MISSION CLEAR（撃破数・残りライフ表示）

## 検証中に見つけて直した不具合
1. **画面左右の反転**：カメラが +Z 方向を向くため、ワールド −X が画面右に映り、「左スワイプで右の物陰」が起きた。画面左＝ワールド +X として敵・物陰の左右割り当てと判定を統一し、スクショで再確認済み。
2. **撃破カウンタの分母**：敵は計6体なのに総数定数が5になっていた → 6に修正し再確認済み。

## 仮素材・未規定の仮置き（PLACEHOLDER）
- **絵は全て仮素材**：主人公／敵はボックス組みの人型シルエット（スーツ色＋ネクタイ色で区別、スナイパーは暗赤系）。車・街路樹・ビルも単純な立体＋窓テクスチャの自動生成。キャラ造形（ドット絵か板絵か）は未決定事項なので、決まり次第差し替える前提。
- 仕様未規定のため仮置きした数値（承認パラメータではない）：
  - 通常敵の命中率：露出時に35%（被弾＝ライフ−1）。通常敵の射撃間隔：露出1.15秒で1発→1.5秒物陰、のサイクル
  - スナイパーの物陰待機：2.2秒サイクル。後退速度：1.4m/s。被弾後の無敵時間：1.0秒
  - 同方向に敵がいない場合の射撃は、前方の有効な敵全体から優先順位で選ぶ（フォールバック）
  - 物陰が近くに無い場所での左右スワイプは、その場で横ステップ回避（短時間物陰扱い）
- 気づき：ヘッドレス検証環境では描画が重く、ゲーム内時間の進みが実時間より遅くなった（dtクランプ0.05秒の影響）。実機では通常速度のはずだが、実機での体感確認が必須。
- HUD はタイトル画面の裏にも薄く見える（オーバーレイ半透明のため）。試作としては許容、気になる場合は不透明度調整。
- 音は未実装（試作 v0 の範囲外）。
