# RUNEFALL

## 1. 概要

スマートフォンブラウザ向けローグライクパズルゲーム。単一HTMLファイル（index.html）で完結。外部ライブラリなし。

- **バージョン**: APP_VERSION = '1.7.2'
- **ゲームループ**: タイトル → 難易度選択 → デッキ選択 → マップ（8フロア） → 戦闘/イベント/ショップ/休憩/宝箱 → 報酬 → 繰り返し → ボス戦 → クリア/ゲームオーバー
- **ゴール**: F8のボス（魔竜）を倒してクリア

---

## 2. 動作環境

- **対象**: スマートフォンブラウザ（iOS Safari / Android Chrome）
- **公開先**: GitHub Pages（URLは開発者が保持）
- **依存**: なし（Webフォント：Google Fonts Cinzel のみ外部参照）
- **ストレージ**: localStorage のみ使用

---

## 3. ファイル構成

```
index.html   2601行（CSS + HTML + JS 全て内包）
  ├─ <head>    meta / CSS / キャッシュ対策スクリプト
  ├─ <body>    画面HTML × 14画面
  └─ <script>  ブロック1: APP_VERSION管理（16行）
               ブロック2: ゲームロジック全体（1870行）
```

補助ドキュメント:
- `todo.md` — タスク管理（廃止済み・changelog.md / spec.md §10 参照）
- `roadmap.md` / `changelog.md` / `deploy.md` / `modularization.md` / `README.md`
- `arch.md` / `balance.md` / `item.md` / `spec_old.md` — 廃止済み（spec.md に統合）

---

## 4. システム構造

### RFモジュール（12個・IIFE + デストラクチャリング展開方式）

| モジュール | 内容 | 将来分離先 |
|-----------|------|-----------|
| RF.Tiles | TILE_DEFS, PARTICLE_COLORS | data/tiles.js |
| RF.Spells | SPELL_COMBOS(27種), COMBO_HINTS | data/spells.js |
| RF.Enemies | ENEMIES(5種), BOSS_DEF, getBossAction | data/enemies.js |
| RF.Config | DIFFICULTIES, UPGRADE_DEFS, RANDOM_EVENTS, TUTORIAL_STEPS, ROOM_TYPES, DUNGEON_FLOORS, SHOP_PRICE, MAX_SELECTED, DECK_PRESETS | data/game-config.js |
| RF.Effects | SPELL_EFFECTS(9種) | engine/effect.js |
| RF.Screen | showScreen, onScreenEnter, initTitleCanvas | engine/screen.js |
| RF.Map | generateDungeon, renderMap, enterRoom, advanceFloor | engine/map.js |
| RF.Audio | getAudio, playSE | engine/audio.js |
| RF.Meta | ACHIEVEMENTS(30種), calcMetaXP, checkAchievements, getAchieved | engine/meta.js |
| RF.Rank | RANKS(7段階), getCurrentRank, getRankTitle, checkRankUp, getRankEffects | engine/rank.js |
| RF.Shop | openShop, switchShopTab, renderUpgradeList | ui/shop.js |
| RF.Tutorial | startTutorial〜endTutorial（状態変数内包） | ui/tutorial.js |

**拡張ルール**: 将来のファイル分離時は展開行（`const { ... } = RF.XXX`）を `import` 文に1行置換するだけでよい。

### グローバル変数

```javascript
let G = {}                 // ゲーム状態（ランごとにinitGame()で初期化）
let currentDifficulty      // 選択中の難易度キー
let currentDeckPreset      // 選択中のデッキID（localStorage永続）
let currentEvent = null    // 現在のランダムイベント
let titleRaf / battleRaf / goRaf / clearRaf / shakeRaf  // RAFハンドル
```

### G オブジェクトの主要フィールド

```javascript
G = {
  screen, floor, difficulty, diffConf, deckPreset,
  player: { hp, maxHp, gold, atkBonus, healBonus, goldRate, handSize },
  upgradeLevels: { hp_up, atk_up, heal_up, gold_up, hand_up },
  deck, hand, selected,
  enemy: { name, icon, hp, maxHp, atk, turn, enraged },
  enemyEffects,           // string[] 状態異常
  dungeon,                // { choices:[{type,cleared}] }[] フロア配列（長さ8）
  currentRoom,            // 現在いる部屋 { type, cleared, floorIdx, ... }
  mapRooms, mapCurrentIdx,
  animating, particles,
  totalTurns, spellComboBonus, totalDmg, totalHealed, comboCount,
  floorNoDamage,          // 無傷の勇者実績用（advanceFloorでリセット）
  achEffects,             // 実績ゲーム内効果（initGameで一括計算）
  metaRestBonus,          // ランク永続強化: 休憩+5（Rank3）
  metaShopDiscount,       // ランク永続強化: 価格-3G
  metaBossPreview,        // ランク永続強化: ボス予告フラグ
  metaShopChoices,        // ランク永続強化: ショップ選択肢数（4 or 5）
}
```

### 画面遷移（showScreen）

**禁止事項**: `position:absolute + opacity + transition` 方式は禁止。`display:none/flex` の切り替えのみ。

```javascript
showScreen(id) {
  // 全RAFを一括キャンセル（titleRaf/battleRaf/goRaf/clearRaf/shakeRaf）
  // 全.screenをdisplay:none → 対象をdisplay:flex
  // G.screen = id
  // onScreenEnter(id)
}
```

onScreenEnterの対応:
- `title` → initTitleCanvas + updateTitleBestScores
- `deck-select` → openDeckSelectで描画済み（処理なし）
- `achievements` → renderAchievements
- `map` → renderMap
- `battle` → enterBattle
- `gameover` → initGameoverCanvas
- `clear` → initClearCanvas

---

## 5. 主要機能

### 5.1 難易度

| キー | label | playerHp | handSize | atkMult | hpMult | deckLimit | goldMult | scoreMult※ | 解放条件 |
|------|-------|----------|----------|---------|--------|-----------|----------|-----------|---------|
| easy | EASY | 100 | 6 | 0.7 | 0.8 | 999 | 1.2 | 0.7（+0.05実績） | 初期 |
| normal | NORMAL | 80 | 6 | 1.0 | 1.0 | 999 | 1.0 | 1.0 | 初期 |
| hard | HARD | 80 | 6 | 1.0 | 1.1 | 20 | 0.8 | 1.5 | 初期 |
| void | VOID | 70 | 6 | 1.3 | 1.3 | 15 | 0.7 | 2.0 | Rank6 or clear_hard実績 |

※ `scoreMult` は `DIFFICULTIES` オブジェクト外。`calcScore()` 内でインライン定義（§6.1参照）。

VOIDボタン（btn-void）: 通常は `display:none`。`updateTitleBestScores()` でRank解放時に表示。

### 5.2 デッキプリセット（6種）

| id | 名前 | 初期枚数 | 解放条件（unlock field） |
|----|------|---------|----------------------|
| balanced | バランス | 12 | null（初期） |
| aggro | 攻撃特化 | 12 | null（初期） |
| sustain | 回復特化 | 12 | null（初期） |
| control | 妨害特化 | 12 | null（初期） |
| chaos | 混沌 | 12（ランダム生成） | 'deck_chaos'（Rank4） |
| ancient | 古代 | 12 | 'deck_ancient'（Rank5） |

各デッキクリア実績（deck_*_clear）達成時: `extraDeckTile = 1` → 初期枚数+1。

### 5.3 タイル属性（6種）

| id | icon | power | color |
|----|------|-------|-------|
| fire | 🔥 | 3 | #ff6633 |
| water | 💧 | 2 | #3399ff |
| wind | 💨 | 2 | #66dd88 |
| earth | 🪨 | 4 | #cc9944 |
| light | ✨ | 2 | #ffdd44 |
| dark | 🌑 | 3 | #9944cc |

### 5.4 ダンジョン構造

- DUNGEON_FLOORS = 8
- F1: enemy固定（1択）
- F2〜F7: 3択（全て異なるタイプ保証）
- F8: boss固定（1択）
- ROOM_TYPES = ['enemy','enemy','enemy','shop','treasure','rest','event']

マップデータ構造:
```javascript
G.dungeon = [{ choices:[{ type, cleared }] }]  // 長さ8
G.mapRooms = [{ ...choice, floorIdx, choiceIdx, x, y }]  // 描画用
```

### 5.5 コンボ呪文（27種）

コンボキー: `[...ids].sort().join(',')` で一致判定。

**2枚（同属性）**: earth×2(Boulder), light×2(HolyLight), fire×2(Ember), wind×2(Gust), water×2(Flood), dark×2(Shadow)

**2枚（異属性）**: fire+dark(Hellfire), water+wind(Storm), fire+water(Steam), earth+light(GaiaBeam), dark+light(Eclipse), earth+water(MudTrap), dark+earth(VoidCrush), light+water(Cleanse), fire+wind(Wildfire), earth+fire(Magma), fire+light(SacredFire), dark+water(Miasma), earth+wind(Sandstorm), light+wind(Zephyr), dark+wind(HexGale)

**3枚（同属性）**: fire×3(Inferno), wind×3(Tornado), dark×3(Abyss), water×3(Tsunami), earth×3(Earthquake), light×3(Radiance)

### 5.6 状態異常（8種）

| effect | 効果 |
|--------|------|
| burn | 次の攻撃時ダメージ+3 |
| stun / sweep | 敵の次ターンスキップ |
| slow | 敵ATK×0.7 |
| curse | 被ダメ+20%（1ターン） |
| blind | プレイヤー選択上限2枚（1ターン） |
| pierce | 次の攻撃ダメ+5（1回） |
| wet | 炎コンボ時ダメ+3（1回） |

### 5.7 通常敵（5種）

| 名前 | maxHp | atk | gold | 特殊行動 |
|------|-------|-----|------|---------|
| スライム | 18 | 3 | 8 | なし |
| ゴブリン | 28 | 5 | 12 | 3T毎Gold盗み |
| スケルトン | 38 | 7 | 16 | HP半減後2連撃 |
| オーガ | 52 | 10 | 22 | 3T毎強打ATK×1.8 |
| 魔竜（抽選用） | 90 | 14 | 60 | — |

### 5.8 ボス（魔竜）

- BOSS_DEF: maxHp:90, atk:10, gold:60
- 難易度倍率適用後のHP/ATKでゲーム内使用
- ボス戦前: HP+50+achEffects.bossHealBonus を自動回復
- 特殊行動（getBossAction）:
  - enrage: HP40%以下（初回）→ ATK×1.8永続
  - breath: 3T毎 → ATK×1.6 + burn付与
  - roar: 5T毎 → 手牌シャッフル

### 5.9 ランクシステム

| Rank | 必要XP | 名前 | title | effect | 解放内容 |
|------|--------|------|-------|--------|---------|
| 1 | 0 | 見習い冒険者 | 見習い | null | 初期（称号はRank1から表示） |
| 2 | 50 | 冒険者 | 冒険者 | shop_choice_5 | ショップ選択肢5枚 |
| 3 | 150 | 熟練冒険者 | 熟練者 | rest_bonus_5 | 休憩回復+5 |
| 4 | 350 | ベテラン冒険者 | ベテラン | deck_chaos | 混沌デッキ解放 |
| 5 | 700 | 精鋭冒険者 | 精鋭 | deck_ancient | 古代デッキ解放 |
| 6 | 1200 | 伝説の冒険者 | 伝説 | diff_void | VOID難易度解放 |
| 7 | 2000 | 魔導の覇者 | 覇者 | title_special | タイトル演出強化（金色星220個） |

将来候補（コメントアウト済み）: Rank8(3500XP), Rank9(5500XP), Rank10(8000XP)。

**ランク拡張手順**: RANKSに1行追加 → 新effectはapplyRankEffects()にif文追加のみ。

### 5.10 実績システム（30種）

カテゴリ: 進行(8) / コンボ(7) / 戦略(7) / デッキ(6) / スコア(2)

各実績達成時: XPボーナスを一度だけ付与 + G.achEffectsにゲーム内効果を反映。

**G.achEffects フィールドと対応実績**:

| フィールド | 効果 | 対応実績 |
|-----------|------|---------|
| xpBonus | 毎ランXP基礎+1〜+3 | reach_f4/combo_first/score_500 |
| initGold | 初期Gold+5 | reach_f7 |
| easyScoreBonus | EASY倍率+0.05（0.7→0.75） | clear_easy |
| restBonus | 休憩回復+5 | clear_normal |
| shopDiscount | ショップ-1G | gold_200 |
| deckLimitBonus | deckLimit+2（HARD:20→22） | deck_30 |
| darkPowerBonus | 闇タイルpower+1 | combo_abyss |
| earthPowerBonus | 土タイルpower+1 | combo_quake |
| holyHealBonus | heal系コンボ回復+3 | combo_radiance |
| libraryRateUp | 書庫イベント出現2倍 | combo_all |
| bossHealBonus | ボス前回復+10〜+20 | no_damage/heal_total_500 |
| bossGold | ボス撃破Gold+30 | boss_kill |
| atkUpMaxLv | 攻撃強化上限Lv4（通常Lv3） | dmg_total_1000 |
| extraDeckTile | 初期デッキ+1枚 | deck_*_clear（選択デッキ一致時） |
| turnBonusRate | ターンボーナス減少率7（通常8） | score_1000 |
| goldBonusRate | Goldボーナス率0.6（通常0.5） | score_500 |

### 5.11 ランダムイベント（6種）

| id | 名前 | canDecline | 出現条件 |
|----|------|-----------|---------|
| blessing | 祝福の泉 | true | 常時 |
| curse | 呪いの祭壇 | true | 常時 |
| gamble | 運命の賭け | true | 常時 |
| merchant | 旅の魔術師 | true | 常時 |
| library | 古代の書庫 | false | 常時（combo_all達成でpool内2倍） |
| cross | 運命の交差点 | false | Rank解放(event_cross効果) or runs_20実績達成後 |

### 5.12 ショップ・強化

- SHOP_PRICE: 15G
  - rankEffects shop_discount: -3G
  - achEffects shopDiscount: -1G
  - 最小価格: 11G（両方適用時）
- 選択肢枚数: 4枚（Rank2 shop_choice_5で5枚）
- UPGRADE_DEFS（5種）:

| id | 名前 | baseCost | maxLv | 効果 |
|----|------|----------|-------|------|
| hp_up | 体力強化 | 20 | 3 | maxHP+20/Lv |
| atk_up | 攻撃強化 | 25 | 3（実績で4） | ダメージ+2/Lv |
| heal_up | 回復強化 | 20 | 2 | 回復量+6/Lv |
| gold_up | 財宝の加護 | 15 | 2 | Gold獲得+30%/Lv |
| hand_up | 手札拡張 | 30 | 2 | 手札+1枚/Lv |

---

## 6. 重要ロジック

### 6.1 スコア計算

```
floorScore = floor × 100
turnBonus  = max(0, 500 - totalTurns × turnBonusRate)   // 8 or 7（score_1000実績）
goldBonus  = floor(gold × goldBonusRate)                 // 0.5 or 0.6（score_500実績）
diffMult   = { easy:0.7+easyScoreBonus, normal:1.0, hard:1.5, void:2.0 }
clearBonus = 300（クリア時のみ）
score = floor((floorScore + turnBonus + goldBonus + clearBonus) × diffMult)
```

### 6.2 メタXP計算（calcMetaXP）

```
基礎: 3 + achEffects.xpBonus（最大+3）
F4到達: +5
F7到達: +10
クリア: +20
コンボ5種以上: +3
Abyss発動: +5
NORMALクリアボーナス: +8
HARDクリアボーナス: +20
```

実績達成ボーナス（一度のみ）: 合計最大1120XP。

### 6.3 キャッシュバスティング

```javascript
// <head>の最初のscriptブロック（16行）で実行
var APP_VERSION = '1.7.1';
var key = 'runefall_version';
var stored = localStorage.getItem(key);
if (stored !== APP_VERSION) {
  localStorage.setItem(key, APP_VERSION);
  if (stored !== null) window.location.reload(true);
}
// 更新時はAPP_VERSIONを変更するだけで全端末に新版が配信される
```

### 6.4 Canvas初期化パターン

```javascript
requestAnimationFrame(() => {
  const rect = canvas.getBoundingClientRect();
  canvas.width = rect.width * dpr;
  canvas.height = rect.height * dpr;
  // ... 初期化処理
});
// display:flex直後はサイズ0になるため必ずRAF遅延を使う
```

### 6.5 Canvas touch-action 設計

| Canvas | CSS設定 | 理由 |
|--------|---------|------|
| #title-canvas | pointer-events:none | 演出専用・ボタン操作を通過（Android対応） |
| #gameover-canvas | pointer-events:none | 同上 |
| #clear-canvas | pointer-events:none | 同上 |
| #map-canvas | touch-action:none | マップタップに必要 |
| #battle-canvas | touch-action:none | スクロール防止に必要 |

**背景**: `touch-action:none` を持つCanvasが `position:absolute` で全面展開するとAndroid Chromeがタッチを消費し、後ろのボタンが反応しない。演出専用Canvasは `pointer-events:none` を使うこと。

### 6.6 resolveSpell

```javascript
key = [...ids].sort().join(',')
→ SPELL_COMBOSでkey一致を検索
→ 未ヒット: 基本攻撃（ids.reduce でpower合計）
// spellComboBonus[key]（古代の書庫強化）を加算
// heal > 0 の場合 achEffects.holyHealBonus（+3）を加算
```

---

## 7. UI / 操作

### 7.1 画面一覧（14画面）

| screen id | 用途 |
|-----------|------|
| screen-title | タイトル・難易度選択 |
| screen-achievements | 実績・ランク（6タブ） |
| screen-deck-select | デッキ選択 |
| screen-howto | 遊び方 |
| screen-map | マップ（Canvas） |
| screen-battle | 戦闘（Canvas） |
| screen-reward | 報酬選択 |
| screen-shop | ショップ（タイル購入・強化） |
| screen-deck | デッキ確認 |
| screen-rest | 休憩 |
| screen-treasure | 宝箱 |
| screen-event | ランダムイベント |
| screen-gameover | ゲームオーバー |
| screen-clear | クリア |

### 7.2 ゲームフロー

```
[title] 難易度ボタン → openDeckSelect(diffKey)
[deck-select] デッキ選択 → initGame(diffKey, deckPresetId) → showScreen('map')
[map] タップ → enterRoom(room) → 部屋種別に応じた画面へ
[battle/shop/rest/treasure/event] → 処理後 → showScreen('map') or showScreen('reward')
[gameover/clear] → 再挑戦 → openDeckSelect(currentDifficulty)
```

### 7.3 戦闘操作

1. 手牌タイルをタップ（MAX_SELECTED=3枚、blind時は2枚上限）
2. 「詠唱」ボタン → castSpell()
3. 「スキップ」ボタン → skipTurn()

### 7.4 実績画面タブ構成（6タブ）

ランク / 進行 / コンボ / 戦略 / デッキ / スコア

- `grid-template-columns: repeat(3, 1fr)` で2行×3列
- max-width:480px（page-bodyと同じ幅）
- 未達成を上・達成済みを下にソート
- ランクタブ: 各Rankの達成状況・残りXP・解放内容を表示

---

## 8. エラー・既知問題

### 解決済み

| 問題 | 原因 | 解決策 |
|------|------|--------|
| Android操作不能 | 演出Canvasのtouch-action:none | pointer-events:noneに変更 |
| titleRaf未宣言 | RF.Screen移動時に変数が消えた | RF.Screen定義直前に let titleRaf = null を追加 |
| RF.Shop展開順序 | 定義より前に展開行があった | 展開行を定義後に移動 |
| タブ切り替え不可 | .achievements-listのdisplay:flexが.ach-tab-contentのdisplay:noneを上書き | ach-tab-contentから achievements-listクラスを除去 |

### 現在の制約

- metaBossPreview（竜の知識）はフラグとして存在するが、ボス予告は固定文字列（動的な次アクション予告ではない）
- VOID難易度のベストスコアは保存（runefall_best_void）されるが、実績画面での明示表示なし
- 「運命の交差点」の2択はランダムのため同じ内容が重複して出ることがある

---

## 9. 開発ガイド

### 9.1 更新時の必須手順

```
1. 変更を実装
2. APP_VERSIONを変更（例: '1.7.1' → '1.7.2'）
3. コミット・プッシュ
// キャッシュバスティングはこれだけで全端末に自動適用される
```

### 9.2 コミットメッセージ形式（確定）

```
要約（英語・50文字以内・命令形）

本文（日本語・変更内容の箇条書き）
```

### 9.3 ランク追加手順

```javascript
// 1. RF.Rank内のRANKSに1行追加
{ rank:8, xp:3500, name:'魔術の探求者', title:'探求者', effect:'combo_xp_bonus', desc:'説明' },

// 2. applyRankEffects()に対応処理を追加
if (rank.effect === 'combo_xp_bonus') { ... }

// getRankEffects()は自動的に新effectを返す（他の変更不要）
```

### 9.4 実績追加手順

```javascript
// 1. RF.Meta内のACHIEVEMENTSに1行追加
{ id:'new_ach', cat:'戦略', icon:'🎯', name:'新実績', cond:'条件説明', xp:XX },

// 2. checkAchievements()内にcheck呼び出しを追加
check('new_ach', 条件式);

// 3. ゲーム内効果が必要な場合はinitGame内のachEffectsにフィールド追加
```

### 9.5 新コンボ追加手順

```javascript
// 1. RF.Spells内のSPELL_COMBOSに追加
{ key:'a,b', name:'SpellName', dmg:X, heal:Y, effect:'状態異常', desc:'説明' },

// 2. COMBO_HINTSにも追加（デッキ確認画面の表示用）
{ name:'SpellName', id:'a', need:1, id2:'b', need2:1 },
```

### 9.6 Canvas新規追加時の注意

`showScreen()` 冒頭の全RAFキャンセル処理に新しいRAFハンドルを追加すること。

```javascript
if (newRaf) { cancelAnimationFrame(newRaf); newRaf=null; }
```

背景Canvasには必ず `pointer-events:none` を設定すること（Android対応）。

---

## 10. 改善課題（保留・将来候補）

| 優先度 | 内容 |
|--------|------|
| 高 | ~~`deck_chaos_clear` / `deck_ancient_clear` の達成判定：確認済み・実装正常~~ ✅ |
| 中 | VOIDベストスコアをタイトル画面・実績画面に表示（`runefall_best_void` はすでに保存済み。`updateTitleBestScores` と `renderAchievements` に追記するだけ） |
| 中 | `metaBossPreview` の動的ボス次アクション予告（現在は固定文字列。`getBossAction` の返値を利用して次ターン行動を表示） |
| ~~中~~ | ~~ショップ価格の最小値ガード：`Math.max(1, price)` で0G以下になるケースを防止~~ ✅ |
| ~~低~~ | ~~`score_2000` の `cat` が `'戦略'` になっているが `'スコア'` が正しい~~ ✅ |
| 中 | 「竜の知識」の動的ボス予告（現在は固定文字列） |
| 低 | PWA対応（manifest.json / Service Worker） |
| 低 | フロア数拡張（8→10〜12） |
| 低 | 新敵の追加（後半フロアの多様性向上） |
| 低 | 案C：呪い・祝福システム |
| 低 | 案D：ミニボス追加（F4・F6） |
| 低 | ランク拡張（Rank8〜10候補はコメントで保持済み） |
| 低 | VOID難易度のベストスコアを実績画面・タイトルに明示表示 |

---

## 11. 運用ルール

### localStorage キー一覧

| キー | 型 | 用途 |
|------|-----|------|
| runefall_best_easy | number | EASYベストスコア |
| runefall_best_normal | number | NORMALベストスコア |
| runefall_best_hard | number | HARDベストスコア |
| runefall_best_void | number | VOIDベストスコア |
| runefall_total_runs | number | 累計プレイ回数 |
| runefall_tutorial_done | string("1") | チュートリアル完了フラグ |
| runefall_last_deck | string | 最後に選択したデッキID |
| runefall_meta_xp | number | 累計メタXP |
| runefall_achievements | JSON string[] | 達成済み実績IDリスト |
| runefall_all_combos | JSON string[] | 通算発動コンボ名セット |
| runefall_total_dmg | number | 通算総ダメージ |
| runefall_total_healed | number | 通算総回復量 |
| runefall_title_special | string("1") | Rank7演出フラグ |
| runefall_version | string | APP_VERSIONキャッシュバスティング |

### ボタン登録一覧（DOMContentLoaded）

| ボタンID | 処理 |
|---------|------|
| btn-easy/normal/hard/void | openDeckSelect(難易度) |
| btn-deck-select-go | initGame → showScreen('map') |
| btn-deck-select-back | showScreen('title') |
| btn-howto / btn-howto-back | showScreen('howto') / showScreen('title') |
| btn-achievements / btn-achievements-back | showScreen('achievements') / showScreen('title') |
| btn-cast | castSpell |
| btn-skip | skipTurn |
| btn-reward-skip | showScreen('map') |
| btn-shop-leave / btn-rest-leave / btn-treasure-leave | showScreen('map') |
| btn-open-deck | openDeck |
| btn-deck-back | showScreen('map') |
| btn-deck-to-shop | openShop() |
| btn-tutorial-next | nextTutorialStep |
| btn-event-accept | currentEvent.onAccept(G) |
| btn-event-decline | showScreen('map') |
| btn-retry / btn-clear-retry | openDeckSelect(currentDifficulty) |
| btn-title / btn-clear-title | showScreen('title') |
