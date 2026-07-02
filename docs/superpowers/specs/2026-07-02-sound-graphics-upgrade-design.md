# SMASH ARENA サウンド・グラフィック強化（第1バッチ）設計書

作成日: 2026-07-02
対象: `index.html`（単体HTML構成を維持）

## 背景と方針

`SMASH_ARENA_3D_POLYGON_SPEC.md` のフル3D化は行わず、現行の Canvas 2D + WebAudio アーキテクチャのまま、体感インパクトの大きい改善だけを段階的に入れる（ユーザー承認済みの方針）。

第1バッチの範囲:

- 音源: 7 SE + バトルBGM 1曲のサンプル音源統合（レイヤー方式）
- グラフィック: ヒットフラッシュ強化、足場の接地ライン発光、重量級着地インパクト

対象外（効果確認後に判断）:

- キャラクター陰影・リムライトの底上げ
- 残り11種のSE、選択画面BGM
- Three.js / Vite 移行

## アセット（配置済み）

| パス | 内容 | 出典 |
|---|---|---|
| `assets/audio/se/jump.mp3` | ジャンプ | Kenney Digital Audio (CC0) |
| `assets/audio/se/hit.mp3` | 通常ヒット | Kenney Impact Sounds (CC0) |
| `assets/audio/se/smash.mp3` | スマッシュヒット | Kenney Impact Sounds (CC0) |
| `assets/audio/se/shieldbreak.mp3` | シールドブレイク | Kenney Impact Sounds (CC0) |
| `assets/audio/se/ko.mp3` | KO | Kenney Impact Sounds (CC0) |
| `assets/audio/se/select.mp3` | メニュー決定 | Kenney Interface Sounds (CC0) |
| `assets/audio/se/go.mp3` | 試合開始 GO | Kenney Digital Audio (CC0) |
| `assets/audio/bgm/battle.mp3` | バトルBGM（約123秒） | Suno生成（所有者作成） |

出典詳細は `assets/audio/CREDITS.md` を正とする。

## オーディオ設計

### ローダー（`SAMPLES` 新設）

- `index.html` 内に `SAMPLES` オブジェクトを追加。
- 初回ユーザージェスチャ（既存 `onFirstGesture()`）で `SFX.init()` 後に、`fetch` + `decodeAudioData` で上記8ファイルを非同期ロードし `AudioBuffer` として保持する。
- ロード失敗（`file://` 直開き、ファイル欠落、デコード失敗）は握りつぶし、該当音は未ロード扱いにする。コンソールへのエラー出力はしない（warn 1行まで可）。

### SE再生（レイヤー方式）

- `SFX.play(name)` の対象7 case（`jump` / `hit` / `smash` / `break` / `ko` / `select` / `go`）の先頭でサンプル再生を試みる。
- サンプル再生成功時も、既存合成のうち低域レイヤー（`thump`）とノイズは鳴らし続けて厚みを出す。
- サンプルと帯域が被る中高域の合成トーン（`tone` の square/sawtooth 系）は、サンプル再生時のみ音量を約50%に下げる（省略はしない。フォールバック時と同じコードパスを通すため）。
- サンプル未ロード時は現行の合成音がそのまま鳴る（挙動不変のフォールバック）。
- 再生は `AudioBufferSourceNode` → 既存バス（sfx / ui）接続。既存の `canPlay()` 連打抑制はそのまま効かせる。

### BGM

- バトル開始時に `battle.mp3` を `AudioBufferSourceNode.loop = true` で再生開始し、music バスへ接続する。
- バトル終了（リザルト表示 or キャラ選択へ戻る）で停止する。
- サンプルBGM再生中は、バトル用の合成 `bgmTick()` を呼ばない。選択画面は現行の合成BGMを継続する。
- `battle.mp3` 未ロード時は現行の合成バトルBGMにフォールバックする。
- ミュート（`M`）・音量は既存の master / musicBus 系統で制御されるため追加実装なし。

## グラフィック設計

### 1. ヒットフラッシュ強化

- `applyHit()` で被弾者に `flashTimer`（2〜4フレーム）を設定し、キャラ描画後に白のオーバーレイ（`globalCompositeOperation: "source-atop"` 相当の白塗り、または白ディスクの重ね）を出す。
- ヒット地点に衝撃リング（拡大しながらフェードする円）を追加。通常ヒットは小さめ白、スマッシュ級（`baseKB >= 5 || dmg >= 9`、既存のSE分岐と同条件）は大きめ+攻撃側キャラ色。
- リングは既存 `particles` 配列に type 付きで追加するか、専用の短い配列で管理する（既存パーティクル描画を壊さない）。

### 2. 足場の接地ライン発光

- `drawMainArenaPlatform()` の緑ライン（`#72d66c` の帯）に、`frame` 基準の正弦波で脈動する発光（`shadowBlur` + 明度変化）を追加する。
- `drawSoftArenaPlatform()` の上辺にも同様の細い発光ラインを追加する。
- 発光は背景から足場の接地可能範囲が常に読み取れる強さにし、キャラより目立たせない。

### 3. 重量級着地インパクト

- `physics()` の着地判定（既存の `prevVy > 6` で `squash` を設定している箇所）で、`char.weight >= 1.0` かつ落下速度が閾値以上の場合に:
  - 土煙パーティクル（左右に流れる灰色系、既存 `spawnSparks` の変種）
  - 接地点に短命の地割れ風ライン（数フレームでフェード）
- ガル（weight 1.45）で最も顕著、ノヴァ/ボルト（1.00〜1.05）は控えめになるよう weight でスケールする。

## 実装上の制約

- `index.html` 単体構成を維持。新規JSファイルは作らない。
- 既存のゲームロジック（判定・物理・CPU）は変更しない。
- 既存関数の書き換えは最小限にし、追加コード中心で実装する。
- パーティクル・エフェクトは既存の配列と描画ループに統合し、上限を超える無制限生成をしない。

## テスト・受け入れ基準

- ローカルHTTPサーバーで起動し、選択→バトル→KO→リザルトまでコンソールエラーゼロ。
- 7 SE がイベントで鳴り、バトルBGMがループ再生される。
- `M` ミュートが即時に全音へ効く。
- ヒット/スマッシュ/重量級着地/KO のスクリーンショットでエフェクト差が視認できる。
- `assets/audio/` を一時リネームしても、現行同等の合成音で最後まで遊べる（フォールバック確認）。
- 60 FPS を維持する（エフェクト追加でフレーム落ちしない）。
