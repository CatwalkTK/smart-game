# サウンド・グラフィック強化（第1バッチ）実装計画

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** SMASH ARENA に音源サンプル（7 SE + バトルBGM）とヒット/足場/着地の視覚エフェクトを追加する。

**Architecture:** `index.html` 単体構成を維持。`SAMPLES` オブジェクト（AudioBuffer ローダー+再生）を `SFX` の隣に新設し、既存 `SFX.play()` の各 case にサンプル再生をレイヤーとして重ねる。グラフィックは既存 `particles` 配列に `type` 付きパーティクル（ring / crack）を追加して実現。全変更はフォールバック安全（音源未ロードでも現行合成音で動作）。

**Tech Stack:** Vanilla JS / Canvas 2D / WebAudio API（ライブラリ追加なし）

**Spec:** `docs/superpowers/specs/2026-07-02-sound-graphics-upgrade-design.md`

## Global Constraints

- `index.html` 単体構成を維持。新規JSファイルを作らない。
- 既存のゲームロジック（判定・物理・CPU）を変更しない。
- 音源未ロード時（`file://` 直開き・ファイル欠落）は現行の合成音がそのまま鳴ること。
- パーティクルの無制限生成をしない。
- このプロジェクトにテストフレームワークはない。検証は各タスクの手動確認手順（ローカルサーバー + ブラウザ）で行う。
- 記載の行番号は計画作成時点（commit `d98f6ed`）のもの。先行タスクの編集でずれるため、行番号ではなく引用コードで位置を特定すること。

## 検証環境の起動（各タスク共通）

```bash
cd /Users/Jobs1/AI/smart-game
python3 -m http.server 8127 --bind 127.0.0.1
```

ブラウザで `http://127.0.0.1:8127/index.html` を開く。コンソールにエラーが出ないことを毎回確認する。

---

### Task 1: SAMPLES ローダーとミュート一元化

**Files:**
- Modify: `index.html`（`SFX` オブジェクト末尾 = `bgmTick` の閉じ括弧 `};` の直後、および keydown/クリックのミュート切替2箇所）

**Interfaces:**
- Produces: `SAMPLES.play(name, opts) => boolean`（name: `"jump"|"hit"|"smash"|"break"|"ko"|"select"|"go"`、opts: `{vol?, bus?, pan?}`、再生できたら true）
- Produces: `SAMPLES.startBgm()` / `SAMPLES.stopBgm()` / `SAMPLES.bgmActive: boolean` / `SAMPLES.buffers: {[name]: AudioBuffer}`
- Produces: `SFX.setMuted(m: boolean)`（muted フラグ + master ゲインを同時制御）

- [ ] **Step 1: SAMPLES オブジェクトを追加**

`const SFX = { ... };` の閉じ（`bgmTick(scene) {...},` の後の `};`、現在の1387行目付近）の直後に追加:

```js
// ===== サンプル音源(assets/audio)。未ロード時は合成音にフォールバック =====
const SAMPLES = {
  buffers: {}, loading: false, bgmSource: null, bgmActive: false,
  manifest: {
    jump: "assets/audio/se/jump.mp3",
    hit: "assets/audio/se/hit.mp3",
    smash: "assets/audio/se/smash.mp3",
    break: "assets/audio/se/shieldbreak.mp3",
    ko: "assets/audio/se/ko.mp3",
    select: "assets/audio/se/select.mp3",
    go: "assets/audio/se/go.mp3",
    bgm_battle: "assets/audio/bgm/battle.mp3",
  },
  load() {
    if (this.loading || !SFX.ctx || typeof fetch === "undefined") return;
    this.loading = true;
    for (const [name, url] of Object.entries(this.manifest)) {
      fetch(url)
        .then((r) => { if (!r.ok) throw new Error(String(r.status)); return r.arrayBuffer(); })
        .then((ab) => SFX.ctx.decodeAudioData(ab))
        .then((buf) => { this.buffers[name] = buf; })
        .catch(() => {});
    }
  },
  play(name, opts) {
    const buf = this.buffers[name];
    if (!buf || !SFX.ctx || SFX.muted) return false;
    opts = opts || {};
    const src = SFX.ctx.createBufferSource();
    src.buffer = buf;
    const g = SFX.ctx.createGain();
    g.gain.value = opts.vol != null ? opts.vol : 1;
    src.connect(g);
    SFX.connectOut(g, opts.bus || "sfx", opts.pan || 0);
    src.start();
    return true;
  },
  startBgm() {
    this.stopBgm();
    const buf = this.buffers.bgm_battle;
    if (!buf || !SFX.ctx) return;
    const src = SFX.ctx.createBufferSource();
    src.buffer = buf;
    src.loop = true;
    src.connect(SFX.musicBus);
    src.start();
    this.bgmSource = src;
    this.bgmActive = true;
  },
  stopBgm() {
    if (this.bgmSource) { try { this.bgmSource.stop(); } catch (_) {} }
    this.bgmSource = null;
    this.bgmActive = false;
  },
};
```

- [ ] **Step 2: SFX.setMuted を追加し、ミュート2箇所を差し替え**

`SFX` オブジェクト内、`resume() {...},` の直後にメソッド追加:

```js
  setMuted(m) {
    this.muted = m;
    if (this.master) this.master.gain.value = m ? 0 : 0.72;
  },
```

keydown ハンドラ（現在398行目）:

```js
// 変更前
if (e.code === "KeyM") SFX.muted = !SFX.muted;
// 変更後
if (e.code === "KeyM") SFX.setMuted(!SFX.muted);
```

選択画面のミュートボタン（現在1534行目）:

```js
// 変更前
else if (inRect(mouse, UI.mute)) { SFX.muted = !SFX.muted; if (!SFX.muted) SFX.play("select"); }
// 変更後
else if (inRect(mouse, UI.mute)) { SFX.setMuted(!SFX.muted); if (!SFX.muted) SFX.play("select"); }
```

理由: ループ再生中の BGM ソースは `play()` 系の muted チェックを通らないため、master ゲインを 0 にしないとミュートが効かない。

- [ ] **Step 3: 初回ジェスチャでロード開始**

`onFirstGesture()`（現在393行目）を変更:

```js
// 変更前
function onFirstGesture() { SFX.init(); SFX.resume(); }
// 変更後
function onFirstGesture() { SFX.init(); SFX.resume(); SAMPLES.load(); }
```

- [ ] **Step 4: 動作確認**

サーバー起動 → ページを開いてキーを1回押した後、DevTools コンソールで:

```js
Object.keys(SAMPLES.buffers)
```

Expected: 数秒以内に 8 要素（`jump, hit, smash, break, ko, select, go, bgm_battle`）。コンソールにエラーなし。`M` キーでミュート切替してもエラーなし。

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat(audio): add SAMPLES loader and unified mute control"
```

---

### Task 2: SE 7種のサンプル統合（レイヤー方式）

**Files:**
- Modify: `index.html`（`SFX.play(name)` 内の case `jump` / `hit` / `smash` / `break` / `ko` / `select` / `go`）

**Interfaces:**
- Consumes: `SAMPLES.play(name, opts) => boolean`（Task 1）

方針（スペック準拠）: サンプル再生成功時、低域レイヤー（`thump`）とノイズは**そのまま**、中高域の `tone`（square/sawtooth/triangle の主旋律成分）と `chord` は音量を**約50%に減衰**（コードパスは同一、音量値のみ三項演算子で切替）。

- [ ] **Step 1: 7つの case を書き換え**

`SFX.play(name)` の switch 内、対象 case を以下の通り差し替える（`djump` / `shield` / `shieldhit` / `move` / `count` / `win` は変更しない）:

```js
      case "jump": {
        const s = SAMPLES.play("jump", { vol: 0.5, pan: -0.08 });
        this.tone(360, 0.13, "triangle", s ? 0.09 : 0.18, 760, { attack: 0.004, release: 0.09, filter: 2600, pan: -0.08 });
        this.noise(0.06, 0.055, { freq: 2600, type: "bandpass" });
        break;
      }
      case "hit": {
        if (!this.canPlay("hit", 0.035)) break;
        const s = SAMPLES.play("hit", { vol: 0.9 });
        this.thump(118, 0.16, 0.34, 58);
        this.noise(0.105, 0.24, { freq: 1400, type: "bandpass", q: 1.8, decay: 2.2 });
        this.tone(620, 0.08, "square", s ? 0.055 : 0.11, 360, { attack: 0.001, release: 0.05, filter: 2200 });
        break;
      }
      case "smash": {
        const s = SAMPLES.play("smash", { vol: 1.0 });
        this.thump(72, 0.34, 0.55, 34);
        this.noise(0.22, 0.38, { freq: 620, type: "bandpass", q: 1.1, decay: 2.8 });
        this.noise(0.10, 0.20, { freq: 4200, type: "highpass", delay: 0.025 });
        this.tone(96, 0.30, "sawtooth", s ? 0.12 : 0.24, 42, { attack: 0.002, release: 0.2, filter: 520 });
        break;
      }
      case "break": {
        const s = SAMPLES.play("break", { vol: 0.9 });
        this.thump(58, 0.45, 0.62, 24);
        this.noise(0.36, 0.45, { freq: 520, type: "bandpass", decay: 1.2 });
        this.chord([190, 255, 340, 455], 0.38, s ? 0.06 : 0.12, "sfx", 0.04);
        break;
      }
      case "ko": {
        const s = SAMPLES.play("ko", { vol: 1.0 });
        this.thump(46, 0.58, 0.70, 22);
        this.noise(0.46, 0.50, { freq: 880, type: "bandpass", decay: 1.4 });
        this.tone(740, 0.42, "sawtooth", s ? 0.12 : 0.24, 96, { attack: 0.003, release: 0.28, filter: 1800 });
        break;
      }
      case "select": {
        const s = SAMPLES.play("select", { vol: 0.7, bus: "ui" });
        this.chord([523, 659, 988], 0.16, s ? 0.05 : 0.105, "ui");
        this.tone(1320, 0.08, "sine", s ? 0.04 : 0.08, 1580, { bus: "ui", filter: 5200, delay: 0.03 });
        break;
      }
      case "go": {
        const s = SAMPLES.play("go", { vol: 0.8, bus: "ui" });
        this.thump(64, 0.36, 0.54, 34);
        this.chord([523, 784, 1046], 0.34, s ? 0.09 : 0.18, "ui", 0.02);
        this.noise(0.18, 0.18, { freq: 2800, type: "highpass", delay: 0.035 });
        break;
      }
```

注意: 各 case は `{}` ブロックで囲む（`const s` のスコープ衝突防止）。

- [ ] **Step 2: 動作確認**

サーバー起動 → キャラ選択（select 音）→ バトル開始（go 音）→ ジャンプ/攻撃ヒット/スマッシュ/ガードブレイク/KO を一通り発生させ、音が現行より厚く鳴ること・コンソールエラーなしを確認。

- [ ] **Step 3: フォールバック確認**

```bash
mv assets/audio assets/audio_disabled
```

リロードして同じ操作 → 現行同等の合成音のみで鳴り、エラーが出ないこと（fetch の 404 は `catch(() => {})` で握りつぶされる。ネットワークタブの404は許容、コンソールの未捕捉エラーは不可）。確認後:

```bash
mv assets/audio_disabled assets/audio
```

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(audio): layer sampled SEs over synth for 7 key sounds"
```

---

### Task 3: バトルBGMのループ再生

**Files:**
- Modify: `index.html`（メインループ `loop()` 内の `bgmTick` 呼び出し行、現在2316行目）

**Interfaces:**
- Consumes: `SAMPLES.startBgm()` / `SAMPLES.stopBgm()` / `SAMPLES.bgmActive` / `SAMPLES.buffers.bgm_battle`（Task 1）

- [ ] **Step 1: loop() 内で BGM のライフサイクルを一元管理**

```js
// 変更前
if (frame % 24 === 0 && !(game.state === "battle" && game.paused)) SFX.bgmTick(game.state === "battle" ? "battle" : "menu");
// 変更後
if (game.state === "battle" && !SAMPLES.bgmActive && SAMPLES.buffers.bgm_battle && !SFX.muted) SAMPLES.startBgm();
if (game.state !== "battle" && SAMPLES.bgmActive) SAMPLES.stopBgm();
const sampleBgm = game.state === "battle" && SAMPLES.bgmActive;
if (frame % 24 === 0 && !(game.state === "battle" && game.paused) && !sampleBgm) SFX.bgmTick(game.state === "battle" ? "battle" : "menu");
```

挙動: バトル中にバッファがあれば mp3 ループ開始（ロード遅延時は途中から自動開始）。バトル以外へ遷移（result / R キーで select）で自動停止。mp3 再生中は合成バトルBGMを止め、選択画面では従来の合成メニューBGMを維持。ミュート中は開始せず、再生中のミュートは master ゲイン 0（Task 1）で即時消音。

注: ポーズ中も mp3 BGM は流し続ける（ポーズ中に止まるのは合成BGMのみ、という従来挙動は維持）。ミュート解除時の再開は次フレームの startBgm 判定で自動処理される（ミュート中に停止しないため、実際は流れ続けており master ゲイン解除で即復帰する）。

- [ ] **Step 2: 動作確認**

- バトル開始 → mp3 BGM がループ再生される（123秒後も途切れず継続）
- `M` でミュート → 即無音、再度 `M` → 即復帰
- KO で決着 → リザルト画面で BGM 停止（win ジングルは鳴る）
- ポーズ → BGM は継続、`R` で選択画面 → BGM 停止し合成メニューBGMに戻る
- コンソールエラーなし

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat(audio): loop sampled battle BGM with synth fallback"
```

---

### Task 4: ヒットフラッシュ + 衝撃リング

**Files:**
- Modify: `index.html`（`spawnSparks` 直後にヘルパー追加、`applyHit`、`Fighter.draw`、`Fighter.update` 内タイマー減算部、`updateParticles`、`drawBattle` のパーティクル描画ループ）

**Interfaces:**
- Consumes: `particles` 配列、`spawnSparks(x, y, color, count, speed)`（既存）
- Produces: `spawnRing(x, y, color, maxR, life)`、パーティクル `type: "ring"`、`Fighter.flashTimer: number`

- [ ] **Step 1: spawnRing ヘルパーを追加**

`spawnSparks` 関数（現在438-444行目)の直後に:

```js
function spawnRing(x, y, color, maxR, life) {
  particles.push({ type: "ring", x, y, vx: 0, vy: 0, life, max: life, color, r: 6, maxR });
}
```

- [ ] **Step 2: updateParticles で type 付きを重力対象外に**

```js
// 変更前
function updateParticles() {
  for (let i = particles.length - 1; i >= 0; i--) {
    const p = particles[i];
    p.x += p.vx; p.y += p.vy; p.vy += 0.15; p.vx *= 0.96; p.life--;
    if (p.life <= 0) particles.splice(i, 1);
  }
}
// 変更後
function updateParticles() {
  for (let i = particles.length - 1; i >= 0; i--) {
    const p = particles[i];
    if (!p.type) { p.x += p.vx; p.y += p.vy; p.vy += 0.15; p.vx *= 0.96; }
    p.life--;
    if (p.life <= 0) particles.splice(i, 1);
  }
}
```

- [ ] **Step 3: drawBattle のパーティクル描画に ring 分岐を追加**

```js
// 変更前
  for (const p of particles) {
    ctx.globalAlpha = Math.max(0, p.life / p.max); ctx.fillStyle = p.color;
    ctx.beginPath(); ctx.arc(p.x, p.y, p.r, 0, Math.PI * 2); ctx.fill();
  }
// 変更後
  for (const p of particles) {
    if (p.type === "ring") {
      const t = 1 - p.life / p.max;
      ctx.globalAlpha = Math.max(0, (1 - t) * 0.8);
      ctx.strokeStyle = p.color;
      ctx.lineWidth = 1 + 3 * (1 - t);
      ctx.beginPath(); ctx.arc(p.x, p.y, p.r + (p.maxR - p.r) * t, 0, Math.PI * 2); ctx.stroke();
      continue;
    }
    ctx.globalAlpha = Math.max(0, p.life / p.max); ctx.fillStyle = p.color;
    ctx.beginPath(); ctx.arc(p.x, p.y, p.r, 0, Math.PI * 2); ctx.fill();
  }
```

- [ ] **Step 4: applyHit にフラッシュタイマーとリングを追加**

`applyHit` 末尾（`spawnSparks(target.cx, target.cy, src.char.color, 8, 4);` の直後）に:

```js
  const heavy = def.baseKB >= 5 || def.dmg >= 9;
  target.flashTimer = heavy ? 4 : 2;
  spawnRing(target.cx, target.cy, heavy ? src.char.color : "#ffffff", heavy ? 64 : 34, heavy ? 16 : 10);
```

（`heavy` の条件は既存のSE分岐 `SFX.play(def.baseKB >= 5 || def.dmg >= 9 ? "smash" : "hit")` と同一）

- [ ] **Step 5: Fighter にフラッシュ描画と減算を追加**

`resetForLife` 内の `this.squash = 0;`（現在476行目）の直後に:

```js
    this.flashTimer = 0;
```

`update` 内の `if (this.squash > 0) this.squash--;`（現在521行目）の直後に:

```js
    if (this.flashTimer > 0) this.flashTimer--;
```

`draw()` 内、キャラ本体の `ctx.restore();`（`paintCharacter(...)` 呼び出し後の restore、現在762行目）の直後に:

```js
    if (this.flashTimer > 0) {
      ctx.save();
      ctx.globalCompositeOperation = "lighter";
      ctx.globalAlpha = 0.55 * (this.flashTimer / 4);
      ctx.fillStyle = "#fff";
      ctx.beginPath(); ctx.ellipse(this.cx, this.cy, this.w * 0.9, this.h * 0.75, 0, 0, Math.PI * 2); ctx.fill();
      ctx.restore();
    }
```

- [ ] **Step 6: 動作確認**

バトルで弱攻撃ヒット → 被弾者が白く光り小さな白リングが広がる。スマッシュヒット → 長めのフラッシュ + 攻撃側キャラ色の大きいリング。連打してもパーティクル総数が発散しない（リングは10〜16フレームで自動消滅）。コンソールエラーなし。

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat(fx): add hit flash overlay and impact rings"
```

---

### Task 5: 足場の接地ライン発光

**Files:**
- Modify: `index.html`（`drawMainArenaPlatform` の緑ライン描画部、`drawSoftArenaPlatform` の上辺ライト部）

**Interfaces:**
- Consumes: グローバル `frame`（既存）、`roundRect(ctx, ...)`（既存）

- [ ] **Step 1: メイン足場の緑ラインを脈動発光に**

`drawMainArenaPlatform` 内:

```js
// 変更前
  ctx.fillStyle = "#72d66c";
  roundRect(ctx, p.x, p.y, p.w, 7, 4);
  ctx.fill();
  ctx.fillStyle = "rgba(200,255,160,0.55)";
  roundRect(ctx, p.x + 16, p.y + 1, p.w - 32, 2, 2);
  ctx.fill();
// 変更後
  const pulse = 0.5 + 0.5 * Math.sin(frame * 0.05);
  ctx.save();
  ctx.shadowColor = "rgba(140,255,120,0.9)";
  ctx.shadowBlur = 8 + pulse * 10;
  ctx.fillStyle = "#72d66c";
  roundRect(ctx, p.x, p.y, p.w, 7, 4);
  ctx.fill();
  ctx.restore();
  ctx.fillStyle = "rgba(220,255,180," + (0.45 + pulse * 0.35).toFixed(3) + ")";
  roundRect(ctx, p.x + 16, p.y + 1, p.w - 32, 2, 2);
  ctx.fill();
```

- [ ] **Step 2: ソフト足場の上辺を脈動発光に**

`drawSoftArenaPlatform` 内:

```js
// 変更前
  ctx.fillStyle = "#95dfff";
  roundRect(ctx, p.x, p.y, p.w, 6, 3);
  ctx.fill();
// 変更後
  const pulse = 0.5 + 0.5 * Math.sin(frame * 0.05 + i * 1.7);
  ctx.save();
  ctx.shadowColor = "rgba(120,220,255,0.9)";
  ctx.shadowBlur = 6 + pulse * 8;
  ctx.fillStyle = "#95dfff";
  roundRect(ctx, p.x, p.y, p.w, 6, 3);
  ctx.fill();
  ctx.restore();
```

（`i` は既存の関数引数 `drawSoftArenaPlatform(p, i)`。位相をずらして3枚が同期しないようにする）

- [ ] **Step 3: 動作確認**

バトル画面でメイン足場の緑ラインと3枚のソフト足場上辺がゆっくり明滅する。キャラより目立ちすぎない（発光がキャラ描画を覆わない: 足場はキャラより先に描画されるので z順は不変）。60 FPS 維持（DevTools Performance で確認、または体感でカクつきなし）。

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(fx): add pulsing edge glow to platforms"
```

---

### Task 6: 重量級着地インパクト

**Files:**
- Modify: `index.html`（`spawnRing` の直後にヘルパー追加、`Fighter.physics()` の着地判定、`drawBattle` パーティクル描画に crack 分岐追加）

**Interfaces:**
- Consumes: `particles` 配列、パーティクル type 機構（Task 4）
- Produces: `spawnLandingImpact(x, y, weight)`、パーティクル `type: "crack"`（追加プロパティ `w: number` = 亀裂の幅）

- [ ] **Step 1: spawnLandingImpact ヘルパーを追加**

`spawnRing`（Task 4）の直後に:

```js
function spawnLandingImpact(x, y, weight) {
  const n = Math.round(6 + weight * 6);
  for (let i = 0; i < n; i++) {
    const dir = i % 2 === 0 ? 1 : -1;
    particles.push({
      x: x + dir * (4 + Math.random() * 10), y: y - 2,
      vx: dir * (1.2 + Math.random() * 1.8) * weight, vy: -(0.5 + Math.random() * 1.2),
      life: 14 + Math.random() * 10, max: 24, color: "#c8beaa", r: 2 + Math.random() * 3,
    });
  }
  if (weight >= 1.2) {
    particles.push({ type: "crack", x, y, vx: 0, vy: 0, life: 12, max: 12, color: "#282218", w: 30 + weight * 18 });
  }
}
```

- [ ] **Step 2: physics() の着地判定にフック**

```js
// 変更前
        if (!this.onGround && this.prevVy > 6) this.squash = 8;
// 変更後
        if (!this.onGround && this.prevVy > 6) {
          this.squash = 8;
          if (this.char.weight >= 1.0 && this.prevVy > 9) spawnLandingImpact(this.cx, p.y, this.char.weight);
        }
```

（`p` はこのループの着地プラットフォーム。weight 1.0 未満の軽量キャラは発生しない。ガル weight=1.45 のみ crack も出る）

- [ ] **Step 3: drawBattle のパーティクル描画に crack 分岐を追加**

Task 4 で追加した `if (p.type === "ring") {...}` の直後に:

```js
    if (p.type === "crack") {
      ctx.globalAlpha = Math.max(0, p.life / p.max) * 0.7;
      ctx.strokeStyle = p.color;
      ctx.lineWidth = 2;
      ctx.beginPath();
      ctx.moveTo(p.x - p.w / 2, p.y);
      ctx.lineTo(p.x - p.w * 0.2, p.y + 3);
      ctx.lineTo(p.x + p.w * 0.15, p.y + 1);
      ctx.lineTo(p.x + p.w / 2, p.y + 4);
      ctx.stroke();
      continue;
    }
```

- [ ] **Step 4: 動作確認**

ガル（重量級）を選び、大ジャンプから着地 → 土煙 + 短い亀裂ラインが出る。ノヴァ（weight 1.00）は控えめな土煙のみ、ジン/ピコ（weight < 1.0）は何も出ない。小ジャンプの着地（落下速度が低い）では出ない。コンソールエラーなし。

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat(fx): add heavyweight landing dust and crack effect"
```

---

### Task 7: 受け入れ検証

**Files:** なし（検証のみ）

- [ ] **Step 1: フルプレイスルー**

サーバー起動 → 1P vs CPU を1試合完走、2P対戦を開始確認。以下をチェック:

- 選択→バトル→KO→リザルト→選択 の全遷移でコンソールエラーゼロ
- 7 SE すべてがイベントで鳴る（select / go / jump / hit / smash / break / ko）
- バトルBGMがループし、リザルトで止まり、選択画面で合成メニューBGMに戻る
- `M` ミュートが BGM 含め即時に効く
- ヒットフラッシュ・リング・足場発光・重量級着地が視認できる
- 体感でフレーム落ちなし

- [ ] **Step 2: フォールバック最終確認**

```bash
mv assets/audio assets/audio_disabled
```

リロード → 合成音のみで1試合遊べる・未捕捉エラーなし →

```bash
mv assets/audio_disabled assets/audio
```

- [ ] **Step 3: README 更新と最終コミット**

`README.md` の「現在の内容」に追記:

```markdown
- サンプル音源SE（Kenney CC0）+ Sunoバトル BGM（未ロード時は合成音へ自動フォールバック)
- ヒットフラッシュ / 衝撃リング / 足場発光 / 重量級着地エフェクト
```

```bash
git add README.md
git commit -m "docs: note sampled audio and new battle effects in README"
```
