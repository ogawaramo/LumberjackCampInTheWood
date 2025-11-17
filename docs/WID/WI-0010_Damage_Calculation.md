# 作業指示書 - WI-0010

**作成日**: 2025-11-17
**ステータス**: 作成完了
**優先度**: 最高
**見積工数**: 1日（8時間）

---

## 1. 作業ID

**WI-0010**

---

## 2. 作業名

**Implement Damage Calculation System - ダメージ計算システムとリワード付与の実装**

---

## 3. 作業目的

斧で木を攻撃した際のダメージ計算式を実装し、スキルレベル・斧倍率・協力ボーナス・クリティカルヒットを含む複雑な計算を正確に実行する。また、伐採完了時のXPおよびコイン付与ロジックを実装し、プレイヤーの成長サイクルを完成させる。この作業により、プレイヤーが「木を叩く → ダメージが入る → 倒れる → 報酬を得る」という気持ちよいゲームループを体験できるようにする。

---

## 4. 対象

### 対象システム
- **システム名**: 森のきこりキャンプ - Phase 1
- **プラットフォーム**: VRChat（Unity 2022.3.22f1 / SDK 3.9.0 / UdonSharp 1.1.9）

### 対象モジュール/パッケージ
- **Module**: M-11 AxeInteraction
- **Layer**: Gameplay Layer
- **Functions**: F-37 (Calculate Damage), F-39 (Award XP and Coins)

### 対象ファイルパス
- **編集対象**: `/Assets/WoodcutterCamp/Scripts/Gameplay/AxeController.cs`（既に斧スイング判定とコライダー衝突判定が実装済みと仮定）

---

## 5. 前提条件

### 環境
- Unity 2022.3.22f1 がインストール済み
- VRChat SDK 3.9.0 がプロジェクトにインポート済み
- UdonSharp 1.1.9 がインポート済み

### 依存モジュールの実装完了
- **WI-0009**: AxeInteraction Detection（斧スイング判定、コライダー衝突判定）が完了していること
- **WI-0006**: SkillEffects（ダメージボーナス、クリティカル率計算）が完了していること
- **WI-0005**: SkillManager（XP管理、レベル管理）が完了していること
- **WI-0013**: CoinManager（コイン加算機能）が完了していること
- **WI-0020**: CooperativeTracker（協力ボーナス判定）が完了していること

### 初期状態
- `/Assets/WoodcutterCamp/Scripts/Gameplay/AxeController.cs` が存在し、`OnCollisionEnter()` メソッドが実装されている
- GameManager.Instance経由で以下のモジュール参照が取得可能：
  - `SkillManager`
  - `SkillEffects`
  - `CoinManager`
  - `CooperativeTracker`

### ブランチ
- 作業ブランチ: `feature/WI-0010-damage-calculation`（`main` からブランチ作成）

---

## 6. 入力

### 入力データ/想定シナリオ

#### シナリオ1: 通常攻撃（協力なし、クリティカルなし）
```csharp
// 入力
int playerLevel = 5;               // Woodcuttingレベル5
int axeBaseDamage = 10;            // 木の斧（基礎ダメージ10）
float axeMultiplier = 1.0f;        // 木の斧倍率（1.0倍）
bool isCooperative = false;        // 協力伐採なし
bool isCritical = false;           // クリティカルなし

// 期待されるダメージ計算
// ダメージ倍率 = 1.0 + (5 × 0.05) = 1.25
// 最終ダメージ = 10 × 1.25 × 1.0 × 1.0 × 1.0 = 12.5 → 12（整数化）
```

#### シナリオ2: 協力伐採 + クリティカルヒット
```csharp
// 入力
int playerLevel = 8;               // Woodcuttingレベル8
int axeBaseDamage = 15;            // 鉄の斧（基礎ダメージ15）
float axeMultiplier = 1.5f;        // 鉄の斧倍率（1.5倍）
bool isCooperative = true;         // 協力伐採（+20%）
bool isCritical = true;            // クリティカルヒット（2倍）

// 期待されるダメージ計算
// ダメージ倍率 = 1.0 + (8 × 0.05) = 1.40
// 協力ボーナス = 1.2
// クリティカル倍率 = 2.0
// 最終ダメージ = 15 × 1.40 × 1.5 × 1.2 × 2.0 = 75.6 → 75（整数化）
```

#### シナリオ3: 木の伐採完了（XP・コイン付与）
```csharp
// 入力
TreeType treeType = TreeType.Oak;  // 中くらいの木
bool isCooperative = true;         // 協力伐採

// 期待されるXP
// 基礎XP = 20（Oak）
// 協力ボーナス = +20%
// 最終XP = 20 × 1.2 = 24 XP

// 期待されるコイン
// 基礎コイン = 5（Oak）
// 協力ボーナス = +2コイン
// 最終コイン = 5 + 2 = 7コイン
```

---

## 7. 出力

### 期待する出力データ

#### 1. CalculateDamage()メソッドの戻り値
```csharp
public int CalculateDamage(int baseDamage, float axeMultiplier, bool isCooperative, out bool isCritical)
{
    // 戻り値: 最終ダメージ（整数）
    // out パラメータ: クリティカルヒット判定結果（bool）
}
```

#### 2. AwardRewards()メソッドの動作
```csharp
public void AwardRewards(TreeType treeType, bool isCooperative)
{
    // SkillManager.AddXP(xp) を呼び出し
    // CoinManager.AddCoins(coins) を呼び出し
    // HUDに通知表示（XP獲得、コイン獲得）
}
```

#### 3. クリティカルヒット時の演出
- **ダメージ数値**: 通常の白色 → クリティカル時は赤色で表示
- **エフェクト**: 赤い衝撃パーティクルを木の衝突地点に生成
- **効果音**: 通常のヒット音 → クリティカル時は特殊効果音（"Critical.wav"）

---

## 8. 作業手順

### 8.1 事前準備（15分）

#### ステップ1: ブランチ作成とファイル確認
```bash
# 作業ブランチを作成
cd /mnt/d/Dropbox/Dev/CLI/WCF
git checkout main
git pull origin main
git checkout -b feature/WI-0010-damage-calculation
```

#### ステップ2: 既存コードの確認
Unityエディタで以下のファイルを開き、既存の実装を確認：
- `/Assets/WoodcutterCamp/Scripts/Gameplay/AxeController.cs`
- 既存の `OnCollisionEnter()` メソッドを確認し、どこにダメージ計算を挿入するかを特定

#### ステップ3: 依存モジュールの参照確認
`AxeController.cs` の `Start()` メソッドで、以下の参照が取得されているか確認：
```csharp
private SkillManager skillManager;
private SkillEffects skillEffects;
private CoinManager coinManager;
private CooperativeTracker cooperativeTracker;

void Start()
{
    skillManager = GameManager.Instance.GetSkillManager();
    skillEffects = GameManager.Instance.GetSkillEffects();
    coinManager = GameManager.Instance.GetCoinManager();
    cooperativeTracker = GameManager.Instance.GetCooperativeTracker();

    // Null チェック
    if (skillManager == null || skillEffects == null || coinManager == null || cooperativeTracker == null)
    {
        Debug.LogError("[AxeController] Failed to get required module references from GameManager");
    }
}
```

**注意**: もし上記の参照が未実装の場合、この作業で追加すること。

---

### 8.2 CalculateDamage()メソッドの実装（2時間）

#### ステップ4: メソッドのシグネチャを追加
`AxeController.cs` に以下のメソッドを追加：

```csharp
/// <summary>
/// ダメージ計算を実行する
/// </summary>
/// <param name="baseDamage">斧の基礎ダメージ（例: 木の斧=10, 鉄の斧=15）</param>
/// <param name="axeMultiplier">斧の倍率（例: 木の斧=1.0, 鉄の斧=1.5）</param>
/// <param name="isCooperative">協力伐採中かどうか</param>
/// <param name="isCritical">クリティカルヒット判定結果（outパラメータ）</param>
/// <returns>最終ダメージ（整数）</returns>
public int CalculateDamage(int baseDamage, float axeMultiplier, bool isCooperative, out bool isCritical)
{
    // 実装内容は次のステップ
}
```

#### ステップ5: スキルレベルボーナスの取得
メソッド内で以下を実装：

```csharp
public int CalculateDamage(int baseDamage, float axeMultiplier, bool isCooperative, out bool isCritical)
{
    // 1. スキルレベルを取得
    int currentLevel = skillManager.GetWoodcuttingLevel();

    // 2. ダメージボーナス倍率を計算（SkillEffectsから取得）
    float damageMultiplier = skillEffects.GetDamageMultiplier(currentLevel);

    // 3. 協力ボーナスの取得
    float cooperativeBonus = isCooperative ? 1.2f : 1.0f;

    // Debug.Log($"[AxeController] Damage calculation: Level={currentLevel}, DamageMultiplier={damageMultiplier}, CooperativeBonus={cooperativeBonus}");

    // 次のステップに続く
}
```

**注意点**:
- `skillManager.GetWoodcuttingLevel()` は1〜10の整数を返すことを前提とする
- `skillEffects.GetDamageMultiplier(level)` は `1.0 + (level × 0.05)` を返す（例: Lv5 → 1.25）

#### ステップ6: クリティカルヒット判定
クリティカル率を取得し、ランダム判定を実行：

```csharp
    // 4. クリティカルヒット判定
    float criticalRate = skillEffects.GetCriticalRate(currentLevel);
    isCritical = Random.Range(0f, 1f) < criticalRate;

    float criticalMultiplier = isCritical ? 2.0f : 1.0f;

    if (isCritical)
    {
        Debug.Log($"[AxeController] CRITICAL HIT! Rate={criticalRate:P0}");
    }
```

**注意点**:
- `Random.Range(0f, 1f)` は0.0〜1.0の範囲の浮動小数点数を返す
- `criticalRate` が 0.05（5%）の場合、約5%の確率で `isCritical = true` になる
- UdonSharpでは `Random.Range()` は利用可能

#### ステップ7: 最終ダメージの計算
すべての倍率を掛け合わせて最終ダメージを算出：

```csharp
    // 5. 最終ダメージの計算
    // 計算式: 基礎ダメージ × スキル倍率 × 斧倍率 × 協力ボーナス × クリティカル倍率
    float rawDamage = baseDamage * damageMultiplier * axeMultiplier * cooperativeBonus * criticalMultiplier;

    // 6. 整数に変換（四捨五入）
    int finalDamage = Mathf.RoundToInt(rawDamage);

    // 7. 異常値チェック（ダメージは1〜1000の範囲内）
    if (finalDamage < 1)
    {
        Debug.LogWarning($"[AxeController] Damage is too low: {finalDamage}. Clamping to 1");
        finalDamage = 1;
    }
    else if (finalDamage > 1000)
    {
        Debug.LogWarning($"[AxeController] Damage is too high: {finalDamage}. Clamping to 1000");
        finalDamage = 1000;
    }

    Debug.Log($"[AxeController] Final damage calculated: {finalDamage} (Base={baseDamage}, SkillMult={damageMultiplier:F2}, AxeMult={axeMultiplier:F2}, CoopBonus={cooperativeBonus:F2}, CritMult={criticalMultiplier:F1})");

    return finalDamage;
}
```

**重要**:
- 計算式は PRD.md (Line 179) の仕様に準拠: `最終ダメージ = 基礎ダメージ × (1 + スキルレベル × 0.05) × 斧倍率 × 協力ボーナス × クリティカル`
- `Mathf.RoundToInt()` で四捨五入することで、12.5 → 13 のように丸める
- 異常値チェックにより、チート行為やバグによる極端なダメージを防ぐ

#### ステップ8: 動作確認用のテストコード追加
`AxeController.cs` に以下のテストメソッドを一時的に追加（開発時のみ）：

```csharp
// [開発時のみ] ダメージ計算のテスト
public void TestDamageCalculation()
{
    Debug.Log("=== Damage Calculation Test ===");

    // テストケース1: Lv5, 木の斧, 通常
    skillManager.SetWoodcuttingLevel(5); // テスト用にレベル設定メソッドを呼ぶ
    bool crit1;
    int damage1 = CalculateDamage(10, 1.0f, false, out crit1);
    Debug.Log($"Test 1 (Lv5, Wood Axe, Normal): {damage1} (Expected: 12~13)");

    // テストケース2: Lv8, 鉄の斧, 協力 + クリティカル強制
    skillManager.SetWoodcuttingLevel(8);
    bool crit2 = true; // 強制的にクリティカルをtrueにする
    // ※ただし、out パラメータは上書きされるため、実際にはクリティカル判定はランダム
    int damage2 = CalculateDamage(15, 1.5f, true, out crit2);
    Debug.Log($"Test 2 (Lv8, Iron Axe, Coop): {damage2} (Expected: 37~38 or 75~76 if critical)");

    Debug.Log("=== Test End ===");
}
```

**実行方法**:
- Unityエディタで再生モードに入る
- Hierarchy上の `AxeController` オブジェクトを選択
- Inspector上で `TestDamageCalculation()` を手動で呼び出す（カスタムエディタボタンを作成するか、別のスクリプトから呼ぶ）

**注意**: テストコードは最終的にコミット前に削除するか、`#if UNITY_EDITOR` で囲むこと。

---

### 8.3 AwardRewards()メソッドの実装（1.5時間）

#### ステップ9: メソッドのシグネチャを追加
`AxeController.cs` に以下のメソッドを追加：

```csharp
/// <summary>
/// 木の伐採完了時にXPとコインを付与する
/// </summary>
/// <param name="treeType">伐採した木の種類（Small, Medium, Large）</param>
/// <param name="isCooperative">協力伐採だったかどうか</param>
public void AwardRewards(TreeType treeType, bool isCooperative)
{
    // 実装内容は次のステップ
}
```

**注意**: `TreeType` はenumとして別途定義されていることを前提とする：
```csharp
public enum TreeType
{
    Small = 0,   // Pine（小さい木）
    Medium = 1,  // Oak（中くらいの木）
    Large = 2    // Ancient（大きな木）
}
```

#### ステップ10: 基礎XPとコインの決定
木の種類に応じて基礎報酬を設定：

```csharp
public void AwardRewards(TreeType treeType, bool isCooperative)
{
    // 1. 木の種類に応じた基礎XPとコインを決定
    int baseXP = 0;
    int baseCoins = 0;

    switch (treeType)
    {
        case TreeType.Small:
            baseXP = 10;
            baseCoins = 3;
            break;
        case TreeType.Medium:
            baseXP = 20;
            baseCoins = 5;
            break;
        case TreeType.Large:
            baseXP = 40;
            baseCoins = 10;
            break;
        default:
            Debug.LogError($"[AxeController] Unknown tree type: {treeType}");
            return;
    }

    Debug.Log($"[AxeController] Base rewards for {treeType}: {baseXP} XP, {baseCoins} Coins");

    // 次のステップに続く
}
```

**参照**: FSD.md (Line 623-626) および PRD.md (Line 163-167) のXP表

#### ステップ11: 協力ボーナスの適用
協力伐採時はXPに+20%、コインに+2を付与：

```csharp
    // 2. 協力ボーナスの適用
    int finalXP = baseXP;
    int finalCoins = baseCoins;

    if (isCooperative)
    {
        // XPは+20%（1.2倍）
        finalXP = Mathf.RoundToInt(baseXP * 1.2f);

        // コインは+2
        finalCoins = baseCoins + 2;

        Debug.Log($"[AxeController] Cooperative bonus applied: XP {baseXP} → {finalXP}, Coins {baseCoins} → {finalCoins}");
    }
```

**参照**: FSD.md (Line 104-106) の協力伐採ボーナス仕様

#### ステップ12: XPとコインの付与
SkillManagerとCoinManagerを経由して報酬を付与：

```csharp
    // 3. XPを付与（SkillManager経由）
    if (skillManager != null)
    {
        skillManager.AddXP(finalXP);
        Debug.Log($"[AxeController] Awarded {finalXP} XP");
    }
    else
    {
        Debug.LogError("[AxeController] SkillManager reference is null, cannot award XP");
    }

    // 4. コインを付与（CoinManager経由）
    if (coinManager != null)
    {
        coinManager.AddCoins(finalCoins);
        Debug.Log($"[AxeController] Awarded {finalCoins} Coins");
    }
    else
    {
        Debug.LogError("[AxeController] CoinManager reference is null, cannot award coins");
    }
```

**注意点**:
- `skillManager.AddXP(int xp)` および `coinManager.AddCoins(int coins)` が実装済みであることを前提とする
- これらのメソッド内でレベルアップ判定やHUD更新が行われる

#### ステップ13: HUD通知の表示（オプション）
XP・コイン獲得をプレイヤーに通知する（HUDManagerが存在する場合）：

```csharp
    // 5. HUD通知（オプション - HUDManagerが実装されている場合）
    var hudManager = GameManager.Instance.GetHUDManager();
    if (hudManager != null)
    {
        string message = $"+{finalXP} XP";
        if (isCooperative)
        {
            message += " (Cooperative Bonus!)";
        }
        hudManager.ShowNotification(message, 2.0f); // 2秒間表示

        hudManager.ShowNotification($"+{finalCoins} Coins", 2.0f);
    }
}
```

**注意**: HUDManagerが未実装の場合、このステップはスキップ可能。WI-0017で実装予定。

---

### 8.4 クリティカルヒット演出の実装（1.5時間）

#### ステップ14: クリティカルエフェクトの準備
Unityエディタで以下のアセットを準備：

1. **パーティクルシステム**:
   - パス: `/Assets/WoodcutterCamp/Prefabs/Effects/CriticalHitEffect.prefab`
   - 赤い衝撃波パーティクル（Particle System使用）
   - 再生時間: 0.5秒
   - サイズ: 0.5m
   - カラー: 赤色（#FF0000）

2. **効果音**:
   - パス: `/Assets/WoodcutterCamp/Audio/Critical.wav`
   - 音量: 0.7
   - 3D Audio設定: Min Distance = 5m, Max Distance = 20m

**作業内容**:
- Unityエディタでパーティクルシステムを作成（Hierarchy > 右クリック > Effects > Particle System）
- Inspector で Color を赤に設定
- Duration を 0.5秒に設定
- Start Speed を 2〜3に設定（衝撃波が広がる速度）
- Prefab化してプロジェクトに保存

#### ステップ15: AxeControllerにエフェクト参照を追加
`AxeController.cs` のクラス変数に以下を追加：

```csharp
[Header("Critical Hit Settings")]
[Tooltip("クリティカルヒット時のパーティクルエフェクト")]
public GameObject criticalHitEffectPrefab;

[Tooltip("クリティカルヒット時の効果音")]
public AudioClip criticalHitSound;

[Tooltip("通常のヒット音")]
public AudioClip normalHitSound;

private AudioSource audioSource;
```

**設定方法**:
- UnityエディタのHierarchy上でAxeControllerオブジェクトを選択
- Inspector上の `AxeController` コンポーネントで以下を設定：
  - `Critical Hit Effect Prefab` に `/Assets/WoodcutterCamp/Prefabs/Effects/CriticalHitEffect.prefab` をドラッグ
  - `Critical Hit Sound` に `/Assets/WoodcutterCamp/Audio/Critical.wav` をドラッグ
  - `Normal Hit Sound` に通常のヒット音をドラッグ

#### ステップ16: Start()メソッドでAudioSourceを取得
```csharp
void Start()
{
    // 既存のモジュール参照取得コード...

    // AudioSourceの取得（なければ追加）
    audioSource = GetComponent<AudioSource>();
    if (audioSource == null)
    {
        audioSource = gameObject.AddComponent<AudioSource>();
        audioSource.spatialBlend = 1.0f; // 3D Audio
        audioSource.minDistance = 5f;
        audioSource.maxDistance = 20f;
    }
}
```

#### ステップ17: クリティカルヒット演出メソッドの実装
`AxeController.cs` に以下のメソッドを追加：

```csharp
/// <summary>
/// クリティカルヒット時の演出を再生する
/// </summary>
/// <param name="hitPosition">木への衝突地点</param>
private void PlayCriticalEffect(Vector3 hitPosition)
{
    // 1. パーティクルエフェクトの生成
    if (criticalHitEffectPrefab != null)
    {
        GameObject effect = Instantiate(criticalHitEffectPrefab, hitPosition, Quaternion.identity);

        // 0.5秒後に自動削除
        Destroy(effect, 0.5f);

        Debug.Log($"[AxeController] Critical effect spawned at {hitPosition}");
    }
    else
    {
        Debug.LogWarning("[AxeController] Critical hit effect prefab is not assigned");
    }

    // 2. 効果音の再生
    if (audioSource != null && criticalHitSound != null)
    {
        audioSource.PlayOneShot(criticalHitSound, 0.7f);
    }
    else
    {
        Debug.LogWarning("[AxeController] Audio source or critical hit sound is not assigned");
    }
}
```

#### ステップ18: OnCollisionEnter()メソッドの修正
既存の `OnCollisionEnter()` メソッドにクリティカル判定と演出を組み込む：

```csharp
private void OnCollisionEnter(Collision collision)
{
    // 既存の衝突判定コード（木のタグチェックなど）...

    // ダメージ計算を実行
    bool isCritical;
    int damage = CalculateDamage(
        baseDamage: currentAxeBaseDamage,        // 装備中の斧の基礎ダメージ
        axeMultiplier: currentAxeMultiplier,     // 装備中の斧の倍率
        isCooperative: IsCooperativeChoppingActive(), // 協力伐採判定（CooperativeTrackerから取得）
        out isCritical
    );

    // 木にダメージを与える（TreeControllerに通知）
    TreeController tree = collision.gameObject.GetComponent<TreeController>();
    if (tree != null)
    {
        tree.TakeDamage(damage, Networking.LocalPlayer);

        // クリティカルヒット時の演出
        if (isCritical)
        {
            Vector3 hitPoint = collision.contacts[0].point; // 衝突地点
            PlayCriticalEffect(hitPoint);
        }
        else
        {
            // 通常のヒット音を再生
            if (audioSource != null && normalHitSound != null)
            {
                audioSource.PlayOneShot(normalHitSound, 0.5f);
            }
        }
    }
}
```

**注意点**:
- `IsCooperativeChoppingActive()` は CooperativeTracker から協力状態を取得するヘルパーメソッド（次のステップで実装）
- `currentAxeBaseDamage` と `currentAxeMultiplier` は装備中の斧の情報（別途管理されていることを前提）

#### ステップ19: 協力伐採判定の統合
CooperativeTrackerから協力ボーナスが有効かを取得するメソッドを追加：

```csharp
/// <summary>
/// 現在協力伐採が有効かどうかを判定する
/// </summary>
/// <returns>協力ボーナスが有効な場合 true</returns>
private bool IsCooperativeChoppingActive()
{
    if (cooperativeTracker == null)
    {
        return false;
    }

    // CooperativeTrackerに問い合わせ
    // ※ CooperativeTracker.IsCooperativeActive() メソッドが実装されていることを前提
    return cooperativeTracker.IsCooperativeActive();
}
```

**依存関係**:
- WI-0020（CooperativeTracker実装）で `IsCooperativeActive()` メソッドが提供されることを前提とする
- もし未実装の場合、この作業では `return false;` で固定し、WI-0020完了後に修正する

---

### 8.5 各モジュールとの連携テスト（1.5時間）

#### ステップ20: 統合テストの準備
以下のテストシナリオを実行し、各モジュールとの連携が正しく動作することを確認：

**テストケース1: 通常ダメージ計算**
1. Unityエディタで再生モードに入る
2. Woodcuttingレベルを5に設定（SkillManager経由）
3. 木の斧（基礎10、倍率1.0）で木を攻撃
4. **期待結果**: ダメージ12〜13が木に与えられる
5. Console上で `[AxeController] Final damage calculated: 12` のようなログが出力される

**テストケース2: クリティカルヒット**
1. Woodcuttingレベルを8に設定（クリティカル率10%）
2. 木を10回攻撃
3. **期待結果**: 約1回はクリティカルヒットが発生し、赤いパーティクルと特殊効果音が再生される
4. Console上で `[AxeController] CRITICAL HIT! Rate=10%` のログが出力される

**テストケース3: 協力伐採ボーナス**
1. 2人のプレイヤー（自分 + テスト用のダミープレイヤー）で同じ木を3秒以内に攻撃
2. CooperativeTrackerが協力状態を `true` に設定
3. **期待結果**: ダメージが1.2倍になる
4. Console上で `CooperativeBonus=1.20` のログが出力される

**テストケース4: 報酬付与**
1. 中くらいの木（Oak）を伐採完了（HP=0）
2. **期待結果**:
   - `skillManager.AddXP(20)` が呼ばれる
   - `coinManager.AddCoins(5)` が呼ばれる
   - HUD上に「+20 XP」「+5 Coins」の通知が表示される

**テストケース5: 協力伐採時の報酬ボーナス**
1. 協力伐採状態で大きな木（Ancient）を伐採完了
2. **期待結果**:
   - XP: 40 × 1.2 = 48 XP
   - コイン: 10 + 2 = 12コイン
   - Console上で `Cooperative bonus applied: XP 40 → 48, Coins 10 → 12` のログが出力される

#### ステップ21: バグ修正と調整
テスト中に発見されたバグを修正：
- ダメージ計算のログが多すぎる場合、Debug.Logの一部をコメントアウト
- クリティカル率が低すぎる/高すぎる場合、SkillEffectsの計算式を確認
- エフェクトが正しい位置に表示されない場合、`collision.contacts[0].point` の取得方法を確認

---

### 8.6 Quest最適化とパフォーマンス確認（1時間）

#### ステップ22: ダメージ計算のパフォーマンス測定
計算負荷が1ms以内であることを確認：

```csharp
public int CalculateDamage(int baseDamage, float axeMultiplier, bool isCooperative, out bool isCritical)
{
    // パフォーマンス測定（開発時のみ）
    #if UNITY_EDITOR
    var stopwatch = System.Diagnostics.Stopwatch.StartNew();
    #endif

    // 既存のダメージ計算コード...

    #if UNITY_EDITOR
    stopwatch.Stop();
    if (stopwatch.ElapsedMilliseconds > 1)
    {
        Debug.LogWarning($"[AxeController] Damage calculation took {stopwatch.ElapsedMilliseconds} ms (target: < 1ms)");
    }
    #endif

    return finalDamage;
}
```

**目標**:
- ダメージ計算は1ms以内で完了すること
- Quest 2環境で10人が同時に攻撃しても60fps維持

#### ステップ23: 不要なGetComponent()呼び出しの排除
OnCollisionEnter()内で毎回GetComponent()を呼ぶのを避ける：

```csharp
// ❌ Bad: 毎回GetComponent()を呼ぶ
private void OnCollisionEnter(Collision collision)
{
    TreeController tree = collision.gameObject.GetComponent<TreeController>(); // 遅い
    if (tree != null) { ... }
}

// ✅ Good: コライダーのタグで判定してからGetComponent()
private void OnCollisionEnter(Collision collision)
{
    if (collision.gameObject.CompareTag("Tree"))
    {
        TreeController tree = collision.gameObject.GetComponent<TreeController>();
        if (tree != null) { ... }
    }
}
```

**注意**: `CompareTag()` は `gameObject.tag == "Tree"` よりも高速。

---

### 8.7 コードレビューとドキュメント更新（30分）

#### ステップ24: コードレビュー用のチェックリスト確認
以下の項目を確認：

- [ ] ダメージ計算式が PRD.md の仕様通りか
- [ ] Null チェックがすべての外部参照に対して実装されているか
- [ ] Debug.Log が適切に配置されているか（本番ビルドでは削除予定）
- [ ] クリティカルエフェクトのPrefabが正しくアサインされているか
- [ ] AudioSourceの3D Audio設定が正しいか
- [ ] UdonSharpで使用できないAPI（LINQ、async/awaitなど）を使用していないか

#### ステップ25: コメントとドキュメント追加
各メソッドに以下の形式でコメントを追加：

```csharp
/// <summary>
/// ダメージ計算を実行する
/// PRD.md (Line 179) の計算式を実装:
/// 最終ダメージ = 基礎ダメージ × (1 + スキルレベル × 0.05) × 斧倍率 × 協力ボーナス × クリティカル
/// </summary>
/// <param name="baseDamage">斧の基礎ダメージ（例: 木の斧=10, 鉄の斧=15）</param>
/// <param name="axeMultiplier">斧の倍率（例: 木の斧=1.0, 鉄の斧=1.5）</param>
/// <param name="isCooperative">協力伐採中かどうか（CooperativeTrackerから取得）</param>
/// <param name="isCritical">クリティカルヒット判定結果（outパラメータ）</param>
/// <returns>最終ダメージ（整数、1〜1000の範囲）</returns>
/// <remarks>
/// 計算負荷: < 1ms（Quest最適化要件）
/// </remarks>
public int CalculateDamage(int baseDamage, float axeMultiplier, bool isCooperative, out bool isCritical)
{
    // ...
}
```

---

### 8.8 コミットとプルリクエスト作成（30分）

#### ステップ26: 変更のコミット
```bash
# 変更ファイルを確認
git status

# ファイルをステージング
git add Assets/WoodcutterCamp/Scripts/Gameplay/AxeController.cs
git add Assets/WoodcutterCamp/Prefabs/Effects/CriticalHitEffect.prefab
git add Assets/WoodcutterCamp/Audio/Critical.wav

# コミット
git commit -m "$(cat <<'EOF'
Implement damage calculation and reward system (WI-0010)

- Add CalculateDamage() method with skill/axe/cooperative/critical multipliers
- Add AwardRewards() method for XP and coin distribution
- Implement critical hit visual/audio effects
- Integrate with SkillManager, CoinManager, CooperativeTracker
- Add performance optimization for Quest 2 (< 1ms calculation)

Related: M-11 AxeInteraction (F-37, F-39)
Dependencies: WI-0005, WI-0006, WI-0013, WI-0020

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>
EOF
)"
```

#### ステップ27: プッシュとプルリクエスト作成
```bash
# リモートにプッシュ
git push -u origin feature/WI-0010-damage-calculation

# プルリクエストを作成（gh コマンド使用）
gh pr create --title "WI-0010: Implement Damage Calculation System" --body "$(cat <<'EOF'
## Summary
- ダメージ計算式の実装（スキルレベル・斧倍率・協力ボーナス・クリティカル）
- XP・コイン付与ロジックの実装
- クリティカルヒット演出（赤いパーティクル、特殊効果音）

## Implementation Details
- `CalculateDamage()`: 複雑な計算式を1ms以内で実行
- `AwardRewards()`: SkillManager、CoinManager経由で報酬付与
- Critical hit effects: Particle System + Audio

## Test plan
- [x] 通常ダメージ計算テスト（Lv5, 木の斧 → 12ダメージ）
- [x] クリティカルヒットテスト（Lv8で約10%発生）
- [x] 協力ボーナステスト（ダメージ1.2倍）
- [x] 報酬付与テスト（Oak伐採 → 20 XP, 5コイン）
- [x] Quest 2パフォーマンステスト（計算 < 1ms）

## Dependencies
- Requires WI-0005 (SkillManager), WI-0006 (SkillEffects), WI-0013 (CoinManager), WI-0020 (CooperativeTracker)

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

---

## 9. 完了条件（Doneの条件）

以下のすべての条件が満たされた場合、この作業は完了とみなす：

### 機能実装
- [ ] `CalculateDamage()` メソッドが実装され、PRD.md (Line 179) の計算式通りに動作する
- [ ] クリティカルヒット判定が正しく動作し、スキルレベルに応じた確率で発生する
- [ ] 協力ボーナスが正しく適用される（ダメージ1.2倍）
- [ ] `AwardRewards()` メソッドが実装され、木の種類と協力状態に応じたXP・コインが付与される
- [ ] クリティカルヒット時に赤いパーティクルと特殊効果音が再生される

### テスト
- [ ] すべてのテストケース（8.5節）が成功する
- [ ] ダメージ計算の実行時間が1ms以内である（Quest 2環境）
- [ ] 10人が同時に攻撃しても60fps維持できる

### コード品質
- [ ] Null チェックがすべての外部参照に対して実装されている
- [ ] 異常値チェック（ダメージ1〜1000の範囲）が実装されている
- [ ] Debug.Logが適切に配置されている
- [ ] コメントとドキュメントが追加されている

### 連携確認
- [ ] SkillManager.AddXP() が正しく呼ばれる
- [ ] CoinManager.AddCoins() が正しく呼ばれる
- [ ] CooperativeTracker.IsCooperativeActive() から協力状態を取得できる
- [ ] TreeController.TakeDamage() にダメージが正しく通知される

### ドキュメント
- [ ] コード内に適切なコメントが追加されている
- [ ] プルリクエストに実装内容とテスト結果が記載されている

---

## 10. テスト観点/テストケース

### テスト種別
- **ユニットテスト**: CalculateDamage()メソッドの計算ロジック
- **統合テスト**: SkillManager、CoinManager、CooperativeTrackerとの連携
- **パフォーマンステスト**: Quest 2環境での実行速度

### テストケース

#### ケース1: 通常ダメージ計算（協力なし、クリティカルなし）
- **前提**: Woodcutting Lv5、木の斧（基礎10、倍率1.0）
- **入力**: `CalculateDamage(10, 1.0f, false, out isCritical)`
- **期待結果**:
  - 戻り値: 12〜13（ダメージ倍率1.25）
  - `isCritical = false`

#### ケース2: 高レベル + 鉄の斧
- **前提**: Woodcutting Lv8、鉄の斧（基礎15、倍率1.5）
- **入力**: `CalculateDamage(15, 1.5f, false, out isCritical)`
- **期待結果**:
  - 戻り値: 31〜32（15 × 1.40 × 1.5 = 31.5）
  - クリティカル発生率約10%

#### ケース3: 協力伐採 + クリティカル
- **前提**: Woodcutting Lv10、黒鋼の斧（基礎20、倍率2.0）、協力伐採、強制クリティカル
- **入力**: `CalculateDamage(20, 2.0f, true, out isCritical)`
- **期待結果（クリティカル発生時）**:
  - 戻り値: 144（20 × 1.50 × 2.0 × 1.2 × 2.0 = 144）
  - `isCritical = true`
  - 赤いパーティクルと特殊効果音が再生される

#### ケース4: 小さい木の報酬（協力なし）
- **入力**: `AwardRewards(TreeType.Small, false)`
- **期待結果**:
  - `skillManager.AddXP(10)` が呼ばれる
  - `coinManager.AddCoins(3)` が呼ばれる

#### ケース5: 大きな木の報酬（協力あり）
- **入力**: `AwardRewards(TreeType.Large, true)`
- **期待結果**:
  - `skillManager.AddXP(48)` が呼ばれる（40 × 1.2）
  - `coinManager.AddCoins(12)` が呼ばれる（10 + 2）
  - Console上で「Cooperative bonus applied」のログが出力される

#### ケース6: 異常値チェック（ダメージ上限）
- **入力**: `CalculateDamage(9999, 10.0f, true, out isCritical)`（意図的に極端な値）
- **期待結果**:
  - 戻り値: 1000（上限でクランプされる）
  - Console上で「Damage is too high」の警告ログが出力される

#### ケース7: Null参照チェック
- **前提**: skillManager が null の状態
- **入力**: `CalculateDamage(10, 1.0f, false, out isCritical)`
- **期待結果**:
  - エラーログ「SkillManager reference is null」が出力される
  - 処理は中断せず、デフォルト値で計算される（または安全に失敗する）

---

## 11. 成果物

### 変更ファイル
- `/Assets/WoodcutterCamp/Scripts/Gameplay/AxeController.cs`
  - `CalculateDamage()` メソッド追加
  - `AwardRewards()` メソッド追加
  - `PlayCriticalEffect()` メソッド追加
  - `IsCooperativeChoppingActive()` メソッド追加
  - `Start()` メソッドに依存モジュール参照取得コード追加
  - `OnCollisionEnter()` メソッドにダメージ計算統合

### 追加アセット
- `/Assets/WoodcutterCamp/Prefabs/Effects/CriticalHitEffect.prefab`
  - 赤い衝撃波パーティクルシステム
- `/Assets/WoodcutterCamp/Audio/Critical.wav`
  - クリティカルヒット専用効果音

### 更新したドキュメント
- 本WI（WI-0010_Damage_Calculation.md）自体が最新のドキュメント
- プルリクエストの説明文に実装内容とテスト結果を記載

---

## 12. 備考

### 注意点
1. **依存関係の確認**:
   - WI-0005（SkillManager）、WI-0006（SkillEffects）、WI-0013（CoinManager）、WI-0020（CooperativeTracker）がすべて完了していない場合、該当機能は一時的にモック実装またはスタブで代替すること

2. **UdonSharp制約**:
   - `out` パラメータは UdonSharp 1.1.9 で使用可能
   - `Random.Range()` は使用可能
   - `Mathf.RoundToInt()` は使用可能
   - LINQ、async/await、Task は使用不可

3. **Quest最適化**:
   - ダメージ計算は計算負荷が少ないため問題ないが、OnCollisionEnter()内での不要なGetComponent()呼び出しを避けること
   - パーティクルシステムは軽量に設定し、同時再生数を制限すること（最大5個程度）

4. **クリティカル率の調整**:
   - テストプレイ時に「クリティカルが出すぎる/出なさすぎる」と感じた場合、SkillEffects.GetCriticalRate() の計算式を調整すること
   - 現在の仕様: Lv5で5%、Lv8で10%、Lv10で15%

5. **協力ボーナスの取得**:
   - CooperativeTrackerが未実装の場合、`IsCooperativeChoppingActive()` は常に `false` を返すようにし、WI-0020完了後に修正すること

6. **HUD通知**:
   - HUDManagerが未実装の場合、ステップ13（HUD通知）はスキップ可能
   - WI-0017（HUDManager実装）完了後に追加すること

### トラブルシューティング

#### 問題1: ダメージが異常に高い/低い
- **原因**: 計算式の掛け算順序、または倍率の取得ミス
- **対処**: Debug.Logで各倍率を出力し、PRD.md の仕様と比較する

#### 問題2: クリティカルが全く発生しない
- **原因**: `Random.Range()` の範囲指定ミス、またはクリティカル率が0
- **対処**: `Debug.Log($"Critical rate: {criticalRate}")` でレートを確認

#### 問題3: 報酬が付与されない
- **原因**: SkillManager/CoinManagerの参照が null
- **対処**: `Start()` メソッドでNullチェックのエラーログを確認

#### 問題4: クリティカルエフェクトが表示されない
- **原因**: Prefabがアサインされていない、または衝突地点の取得ミス
- **対処**: Inspector上で Prefab が正しくアサインされているか確認

### 参考資料
- **PRD.md (Line 176-186)**: ダメージ計算式、クリティカル率の仕様
- **FSD.md (Line 66-107)**: 伐採システムの詳細仕様
- **TDD.md (Line 486-523)**: M-11 AxeInteractionの責務とインターフェース
- **WI-0006_SkillEffects.md**: スキル効果計算の実装詳細

---

**作成者**: VRChat World Development Team
**レビュー担当者**: （レビュー前に指定）
**承認者**: （実装完了後に承認）

---

**ステータス履歴**:
- 2025-11-17: 作業指示書作成完了
- （実装開始日）: 実装開始
- （実装完了日）: 実装完了、プルリクエスト作成
- （承認日）: レビュー承認、mainブランチにマージ
