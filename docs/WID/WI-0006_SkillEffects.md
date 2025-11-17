# 作業指示書 - WI-0006

**作成日**: 2025-11-17
**ステータス**: 作成完了（更新版v2.0）
**優先度**: 最高
**見積工数**: 0.5日（4時間）

---

## 1. 作業ID

**WI-0006**

---

## 2. 作業名

**Implement SkillEffects Calculation - スキル効果計算ロジックの実装（Pure関数方式）**

---

## 3. 作業目的

Woodcuttingスキルレベルに応じたダメージボーナス、クリティカル率、丸太生成数ボーナス率を計算する**純粋関数（Pure Function）**を実装する。状態を持たない関数として設計することで、AxeInteraction（M-11）やLogSpawner（M-12）から呼び出される際の計算負荷を最小限に抑え、Quest最適化を実現する。

---

## 4. 対象

### 対象システム
- **システム名**: 森のきこりキャンプ - Phase 1
- **プラットフォーム**: VRChat（Unity 2022.3.22f1 / SDK 3.9.0 / UdonSharp 1.1.9）

### 対象モジュール/パッケージ
- **Module**: M-05 SkillEffects
- **Layer**: Progression Layer
- **Functions**: F-13 (Damage Bonus), F-14 (Critical Rate), F-15 (Log Bonus Rate)

### 対象ファイルパス
- **新規作成**: `/Assets/WoodcutterCamp/Scripts/Progression/SkillEffects.cs`

---

## 5. 前提条件

### 環境
- Unity 2022.3.22f1 がインストール済み
- VRChat SDK 3.9.0 がプロジェクトにインポート済み
- UdonSharp 1.1.9 がインポート済み
- **SkillManager（M-04）の実装が完了していること（WI-0005完了）**

### 初期状態
- `/Assets/WoodcutterCamp/Scripts/Progression/` フォルダが作成済み
- SkillManager.cs が存在し、`GetWoodcuttingLevel()` メソッドが使用可能

### ブランチ
- 作業ブランチ: `feature/WI-0006-skill-effects`（`main` からブランチ作成）

---

## 6. 入力

### 入力データ
SkillEffectsクラスのメソッドは、呼び出し側（AxeInteractionなど）から**スキルレベル**を受け取る：

```csharp
// 呼び出し側の例（AxeInteraction）
int currentLevel = skillManager.GetWoodcuttingLevel(); // 1〜10
float damageMultiplier = skillEffects.GetDamageMultiplier(currentLevel);
```

### 想定する入力例
- **Lv1の場合**: `level = 1`
- **Lv5の場合**: `level = 5`
- **Lv10の場合**: `level = 10`

---

## 7. 出力

### 期待する出力データ

#### 1. ダメージボーナス倍率（GetDamageMultiplier）
```csharp
// Lv1 → 1.05 （基礎ダメージの5%増加）
// Lv2 → 1.10 （基礎ダメージの10%増加）
// Lv5 → 1.25 （基礎ダメージの25%増加）
// Lv10 → 1.50 （基礎ダメージの50%増加）
```

**計算式**: `1.0 + (レベル × 0.05)`

#### 2. クリティカル率（GetCriticalRate）
```csharp
// Lv1〜4 → 0.0 （0%）
// Lv5〜7 → 0.05 （5%）
// Lv8〜9 → 0.10 （10%）
// Lv10 → 0.15 （15%）
```

#### 3. 丸太生成数ボーナス率（GetLogBonusRate）
```csharp
// Lv1〜2 → 0.0 （0%）
// Lv3〜5 → 0.10 （10%）
// Lv6〜8 → 0.20 （20%）
// Lv9〜10 → 0.30 （30%）
```

---

## 8. 作業手順

### ステップ1: Gitブランチの作成

1. ターミナルを開き、プロジェクトルートに移動:
   ```bash
   cd /mnt/d/Dropbox/Dev/CLI/WCF
   ```

2. mainブランチから新しいブランチを作成:
   ```bash
   git checkout main
   git pull origin main
   git checkout -b feature/WI-0006-skill-effects
   ```

3. ブランチが作成されたことを確認:
   ```bash
   git branch
   # * feature/WI-0006-skill-effects と表示される
   ```

### ステップ2: Unityプロジェクトのセットアップ

1. Unityエディタを起動し、WoodcutterCampプロジェクトを開く

2. Projectウィンドウで `/Assets/WoodcutterCamp/Scripts/Progression/` フォルダを確認
   - 存在しない場合: 右クリック → `Create` → `Folder` で作成

3. Consoleウィンドウを開く（`Window` → `General` → `Console`）
   - エラーがないことを確認

### ステップ3: SkillEffects.cs の新規作成

1. `/Assets/WoodcutterCamp/Scripts/Progression/` フォルダ内で右クリック

2. `Create` → `C# Script` を選択

3. ファイル名を `SkillEffects` に設定（拡張子.csは自動付与）

4. 作成されたファイルをダブルクリックして Visual Studio Code（または任意のエディタ）で開く

### ステップ4: SkillEffects.cs の実装（Pure関数方式）

以下のコードを実装する（**完全なコード例**）:

```csharp
using UdonSharp;
using UnityEngine;

namespace WoodcutterCamp.Progression
{
    /// <summary>
    /// M-05 SkillEffects - スキルレベルに応じた効果計算モジュール
    /// すべてのメソッドは純粋関数（状態を持たない）として実装
    /// </summary>
    [UdonBehaviourSyncMode(BehaviourSyncMode.None)]
    public class SkillEffects : UdonSharpBehaviour
    {
        // ========================================
        // 定数定義（PRDv3.md Section 2.2.2より）
        // ========================================

        /// <summary>ダメージボーナスの基本値</summary>
        private const float BASE_DAMAGE_MULTIPLIER = 1.0f;

        /// <summary>レベルあたりのダメージボーナス増加量（5%/Lv）</summary>
        private const float DAMAGE_BONUS_PER_LEVEL = 0.05f;

        /// <summary>スキルレベルの最小値</summary>
        private const int MIN_SKILL_LEVEL = 1;

        /// <summary>スキルレベルの最大値</summary>
        private const int MAX_SKILL_LEVEL = 10;

        // ========================================
        // 初期化
        // ========================================

        void Start()
        {
            Debug.Log("[SkillEffects] Initialized as Pure Function Module (No State)");
        }

        // ========================================
        // F-13: ダメージボーナス計算
        // ========================================

        /// <summary>
        /// スキルレベルに応じたダメージボーナス倍率を計算する（Pure関数）
        /// </summary>
        /// <param name="level">スキルレベル（1〜10）</param>
        /// <returns>ダメージ倍率（1.05〜1.50）</returns>
        /// <example>
        /// Lv1: 1.05倍 (+5%)
        /// Lv5: 1.25倍 (+25%)
        /// Lv10: 1.50倍 (+50%)
        /// </example>
        /// <remarks>
        /// 計算式: 1.0 + (レベル × 0.05)
        /// PRDv3.md Section 2.2.2より
        /// </remarks>
        public float GetDamageMultiplier(int level)
        {
            // バリデーション: レベルは1〜10の範囲内
            if (level < MIN_SKILL_LEVEL)
            {
                Debug.LogWarning($"[SkillEffects] Level {level} is less than {MIN_SKILL_LEVEL}. Clamping to {MIN_SKILL_LEVEL}.");
                level = MIN_SKILL_LEVEL;
            }
            else if (level > MAX_SKILL_LEVEL)
            {
                Debug.LogWarning($"[SkillEffects] Level {level} is greater than {MAX_SKILL_LEVEL}. Clamping to {MAX_SKILL_LEVEL}.");
                level = MAX_SKILL_LEVEL;
            }

            // 計算式: 1.0 + (レベル × 0.05)
            float multiplier = BASE_DAMAGE_MULTIPLIER + (level * DAMAGE_BONUS_PER_LEVEL);

            return multiplier;
        }

        // ========================================
        // F-14: クリティカル率計算
        // ========================================

        /// <summary>
        /// スキルレベルに応じたクリティカル発生率を計算する（Pure関数）
        /// </summary>
        /// <param name="level">スキルレベル（1〜10）</param>
        /// <returns>クリティカル率（0.0〜0.15）</returns>
        /// <example>
        /// Lv1〜4: 0.0 (0%)
        /// Lv5〜7: 0.05 (5%)
        /// Lv8〜9: 0.10 (10%)
        /// Lv10: 0.15 (15%)
        /// </example>
        /// <remarks>
        /// ステップ関数による段階的な上昇
        /// PRDv3.md Section 2.2.1より
        /// </remarks>
        public float GetCriticalRate(int level)
        {
            // バリデーション: レベルは1〜10の範囲内
            if (level < MIN_SKILL_LEVEL)
            {
                Debug.LogWarning($"[SkillEffects] Level {level} is less than {MIN_SKILL_LEVEL}. Clamping to {MIN_SKILL_LEVEL}.");
                level = MIN_SKILL_LEVEL;
            }
            else if (level > MAX_SKILL_LEVEL)
            {
                Debug.LogWarning($"[SkillEffects] Level {level} is greater than {MAX_SKILL_LEVEL}. Clamping to {MAX_SKILL_LEVEL}.");
                level = MAX_SKILL_LEVEL;
            }

            // ステップ関数による段階的な上昇
            if (level >= 10)
            {
                return 0.15f; // Lv10: 15%
            }
            else if (level >= 8)
            {
                return 0.10f; // Lv8〜9: 10%
            }
            else if (level >= 5)
            {
                return 0.05f; // Lv5〜7: 5%
            }
            else
            {
                return 0.0f; // Lv1〜4: 0%
            }
        }

        // ========================================
        // F-15: 丸太生成数ボーナス率計算
        // ========================================

        /// <summary>
        /// スキルレベルに応じた丸太生成数ボーナス発生率を計算する（Pure関数）
        /// </summary>
        /// <param name="level">スキルレベル（1〜10）</param>
        /// <returns>ボーナス発生率（0.0〜0.30）</returns>
        /// <example>
        /// Lv1〜2: 0.0 (0%)
        /// Lv3〜5: 0.10 (10%)
        /// Lv6〜8: 0.20 (20%)
        /// Lv9〜10: 0.30 (30%)
        /// </example>
        /// <remarks>
        /// ステップ関数による段階的な上昇
        /// PRDv3.md Section 2.2.1より
        /// </remarks>
        public float GetLogBonusRate(int level)
        {
            // バリデーション: レベルは1〜10の範囲内
            if (level < MIN_SKILL_LEVEL)
            {
                Debug.LogWarning($"[SkillEffects] Level {level} is less than {MIN_SKILL_LEVEL}. Clamping to {MIN_SKILL_LEVEL}.");
                level = MIN_SKILL_LEVEL;
            }
            else if (level > MAX_SKILL_LEVEL)
            {
                Debug.LogWarning($"[SkillEffects] Level {level} is greater than {MAX_SKILL_LEVEL}. Clamping to {MAX_SKILL_LEVEL}.");
                level = MAX_SKILL_LEVEL;
            }

            // ステップ関数による段階的な上昇
            if (level >= 9)
            {
                return 0.30f; // Lv9〜10: 30%
            }
            else if (level >= 6)
            {
                return 0.20f; // Lv6〜8: 20%
            }
            else if (level >= 3)
            {
                return 0.10f; // Lv3〜5: 10%
            }
            else
            {
                return 0.0f; // Lv1〜2: 0%
            }
        }

        // ========================================
        // デバッグ用メソッド（開発中のみ使用）
        // ========================================

        /// <summary>
        /// すべてのレベル（1〜10）の効果を出力する（デバッグ用）
        /// </summary>
        /// <remarks>
        /// UdonSharpのCustomEventとして呼び出し可能
        /// Inspectorから手動実行してテスト
        /// </remarks>
        public void _DebugPrintAllEffects()
        {
            Debug.Log("========================================");
            Debug.Log("[SkillEffects] All Level Effects Test");
            Debug.Log("========================================");

            for (int level = 1; level <= 10; level++)
            {
                float damageMultiplier = GetDamageMultiplier(level);
                float criticalRate = GetCriticalRate(level);
                float logBonusRate = GetLogBonusRate(level);

                Debug.Log($"Lv{level}: Damage x{damageMultiplier:F2}, Crit {criticalRate * 100:F0}%, Log Bonus {logBonusRate * 100:F0}%");
            }

            Debug.Log("========================================");
        }

        /// <summary>
        /// 特定のレベルの効果を詳細に出力する（デバッグ用）
        /// </summary>
        /// <param name="level">テストするレベル</param>
        public void _DebugPrintSingleLevel(int level)
        {
            float damageMultiplier = GetDamageMultiplier(level);
            float criticalRate = GetCriticalRate(level);
            float logBonusRate = GetLogBonusRate(level);

            Debug.Log("========================================");
            Debug.Log($"[SkillEffects] Level {level} Effects:");
            Debug.Log($"  Damage Multiplier: {damageMultiplier:F2}x ({(damageMultiplier - 1.0f) * 100:F0}% bonus)");
            Debug.Log($"  Critical Rate: {criticalRate * 100:F0}%");
            Debug.Log($"  Log Bonus Rate: {logBonusRate * 100:F0}%");
            Debug.Log("========================================");
        }
    }
}
```

### ステップ5: コードの保存とUnityへの反映

1. ファイルを保存（Ctrl+S / Cmd+S）

2. Unityエディタに戻り、スクリプトの自動コンパイルを待つ（数秒）

3. Consoleウィンドウでエラーが出ていないことを確認
   - **エラーが出た場合**: コードを見直し、UdonSharpの構文に準拠しているか確認
   - **よくあるエラー**: `var` の使用、LINQ式、一部のC#機能

4. UdonSharpの手動コンパイルを実行（念のため）:
   - メニュー → `UdonSharp` → `Compile All UdonSharp Programs`

### ステップ6: Unityシーンへの配置

1. Hierarchy ウィンドウで右クリック → `Create Empty` を選択

2. オブジェクト名を `SkillEffects` に変更

3. Inspector ウィンドウで `Add Component` をクリック

4. 検索ボックスに `SkillEffects` と入力し、スクリプトをアタッチ

5. オブジェクトをPrefab化（推奨）:
   - Project ウィンドウの `/Assets/WoodcutterCamp/Prefabs/Core/` フォルダを開く
   - Hierarchy の `SkillEffects` オブジェクトをドラッグ&ドロップ
   - Prefabが作成されたことを確認

### ステップ7: デバッグテストの実行

#### 方法A: Unity Playモードでのテスト

1. Unityエディタで Playモード に入る（上部の▶ボタン）

2. Hierarchy で `SkillEffects` オブジェクトを選択

3. Inspector の UdonBehaviour セクションで以下を確認:
   - `_DebugPrintAllEffects` というメソッドが表示される
   - メソッド名の右にある `Send Custom Event` ボタンをクリック

4. Console ウィンドウで以下のようなログが出力されることを確認:
   ```
   ========================================
   [SkillEffects] All Level Effects Test
   ========================================
   Lv1: Damage x1.05, Crit 0%, Log Bonus 0%
   Lv2: Damage x1.10, Crit 0%, Log Bonus 0%
   Lv3: Damage x1.15, Crit 0%, Log Bonus 10%
   Lv4: Damage x1.20, Crit 0%, Log Bonus 10%
   Lv5: Damage x1.25, Crit 5%, Log Bonus 10%
   Lv6: Damage x1.30, Crit 5%, Log Bonus 20%
   Lv7: Damage x1.35, Crit 5%, Log Bonus 20%
   Lv8: Damage x1.40, Crit 10%, Log Bonus 20%
   Lv9: Damage x1.45, Crit 10%, Log Bonus 30%
   Lv10: Damage x1.50, Crit 15%, Log Bonus 30%
   ========================================
   ```

5. すべての値が期待結果と一致することを確認

6. Playモードを終了（▶ボタンを再度クリック）

#### 方法B: テストスクリプトの作成（より詳細なテスト）

1. `/Assets/WoodcutterCamp/Scripts/Tests/` フォルダを作成（存在しない場合）

2. 以下のテストスクリプトを作成:

`SkillEffectsTest.cs`:

```csharp
using UdonSharp;
using UnityEngine;

namespace WoodcutterCamp.Tests
{
    [UdonBehaviourSyncMode(BehaviourSyncMode.None)]
    public class SkillEffectsTest : UdonSharpBehaviour
    {
        public SkillEffects skillEffects;

        void Start()
        {
            RunAllTests();
        }

        void RunAllTests()
        {
            Debug.Log("========================================");
            Debug.Log("[SkillEffectsTest] Starting Tests...");
            Debug.Log("========================================");

            TestDamageMultiplier();
            TestCriticalRate();
            TestLogBonusRate();
            TestEdgeCases();

            Debug.Log("========================================");
            Debug.Log("[SkillEffectsTest] All Tests Completed");
            Debug.Log("========================================");
        }

        void TestDamageMultiplier()
        {
            Debug.Log("[Test] F-13: Damage Multiplier");

            AssertEqual(skillEffects.GetDamageMultiplier(1), 1.05f, "Lv1");
            AssertEqual(skillEffects.GetDamageMultiplier(5), 1.25f, "Lv5");
            AssertEqual(skillEffects.GetDamageMultiplier(10), 1.50f, "Lv10");
        }

        void TestCriticalRate()
        {
            Debug.Log("[Test] F-14: Critical Rate");

            AssertEqual(skillEffects.GetCriticalRate(4), 0.0f, "Lv4");
            AssertEqual(skillEffects.GetCriticalRate(5), 0.05f, "Lv5");
            AssertEqual(skillEffects.GetCriticalRate(8), 0.10f, "Lv8");
            AssertEqual(skillEffects.GetCriticalRate(10), 0.15f, "Lv10");
        }

        void TestLogBonusRate()
        {
            Debug.Log("[Test] F-15: Log Bonus Rate");

            AssertEqual(skillEffects.GetLogBonusRate(2), 0.0f, "Lv2");
            AssertEqual(skillEffects.GetLogBonusRate(3), 0.10f, "Lv3");
            AssertEqual(skillEffects.GetLogBonusRate(6), 0.20f, "Lv6");
            AssertEqual(skillEffects.GetLogBonusRate(9), 0.30f, "Lv9");
        }

        void TestEdgeCases()
        {
            Debug.Log("[Test] Edge Cases");

            // 範囲外の値（Lv0 → Lv1として扱われる）
            AssertEqual(skillEffects.GetDamageMultiplier(0), 1.05f, "Lv0 (clamped to 1)");

            // 範囲外の値（Lv15 → Lv10として扱われる）
            AssertEqual(skillEffects.GetDamageMultiplier(15), 1.50f, "Lv15 (clamped to 10)");
        }

        void AssertEqual(float actual, float expected, string testName)
        {
            float tolerance = 0.001f;
            if (Mathf.Abs(actual - expected) < tolerance)
            {
                Debug.Log($"  ✓ PASS: {testName} = {actual:F2}");
            }
            else
            {
                Debug.LogError($"  ✗ FAIL: {testName} expected {expected:F2}, got {actual:F2}");
            }
        }
    }
}
```

3. テストオブジェクトを作成:
   - Hierarchy → 右クリック → Create Empty
   - 名前を `SkillEffectsTest` に変更
   - `SkillEffectsTest.cs` スクリプトをアタッチ
   - Inspector の `Skill Effects` フィールドに `SkillEffects` オブジェクトをドラッグ

4. Playモードで実行し、テスト結果を確認

---

## 9. 完了条件（Doneの条件）

以下のすべての条件を満たすこと:

- [ ] `SkillEffects.cs` ファイルが `/Assets/WoodcutterCamp/Scripts/Progression/` に作成されている
- [ ] Unityエディタでコンパイルエラーが0件である
- [ ] UdonSharp 1.1.9 でビルドが成功する
- [ ] 3つのpublicメソッド（`GetDamageMultiplier`, `GetCriticalRate`, `GetLogBonusRate`）が実装されている
- [ ] 各メソッドが正しい計算結果を返す（テストケース参照）
- [ ] バリデーション（レベル範囲チェック）がすべてのメソッドに実装されている
- [ ] デバッグメソッド `_DebugPrintAllEffects` でLv1〜10の全効果が正しく出力される
- [ ] SkillEffects Prefabが `/Assets/WoodcutterCamp/Prefabs/Core/` に作成されている
- [ ] XMLドキュメントコメント（`/// <summary>`）がすべてのpublicメソッドに記載されている
- [ ] コード内に `TODO` や未実装箇所がない
- [ ] Playモードでデバッグテストを実行し、すべて成功している

---

## 10. テスト観点/テストケース

### テスト種別
- **ユニットテスト（手動）**: Unityエディタ上での動作確認
- **Pure関数テスト**: 同じ入力に対して常に同じ出力が返ることの確認

### 正常系テストケース

| テストID | テスト内容 | 入力 | 期待結果 | 確認方法 |
|---------|----------|------|---------|---------|
| TC-001 | ダメージ倍率（Lv1） | level = 1 | 1.05 | デバッグログ |
| TC-002 | ダメージ倍率（Lv5） | level = 5 | 1.25 | デバッグログ |
| TC-003 | ダメージ倍率（Lv10） | level = 10 | 1.50 | デバッグログ |
| TC-004 | クリティカル率（Lv1〜4） | level = 3 | 0.0 | デバッグログ |
| TC-005 | クリティカル率（Lv5〜7） | level = 6 | 0.05 | デバッグログ |
| TC-006 | クリティカル率（Lv8〜9） | level = 8 | 0.10 | デバッグログ |
| TC-007 | クリティカル率（Lv10） | level = 10 | 0.15 | デバッグログ |
| TC-008 | 丸太ボーナス率（Lv1〜2） | level = 2 | 0.0 | デバッグログ |
| TC-009 | 丸太ボーナス率（Lv3〜5） | level = 4 | 0.10 | デバッグログ |
| TC-010 | 丸太ボーナス率（Lv6〜8） | level = 7 | 0.20 | デバッグログ |
| TC-011 | 丸太ボーナス率（Lv9〜10） | level = 9 | 0.30 | デバッグログ |

### 異常系テストケース

| テストID | テスト内容 | 入力 | 期待結果 | 確認方法 |
|---------|----------|------|---------|---------|
| TC-012 | レベル下限超過 | level = 0 | 1.05（Lv1として処理） | Warningログ表示 |
| TC-013 | レベル上限超過 | level = 15 | 1.50（Lv10として処理） | Warningログ表示 |
| TC-014 | 負のレベル | level = -5 | 1.05（Lv1として処理） | Warningログ表示 |
| TC-015 | Pure関数性 | 同じlevel値を複数回 | 常に同じ結果 | 繰り返し呼び出し |

### テスト実施手順

1. Unityエディタでテストシーン `Test_SkillEffects` を開く（またはメインシーン）
2. SkillEffects オブジェクトを選択
3. Playモード に入る
4. Inspector で `_DebugPrintAllEffects` を Custom Event として実行
5. Console ウィンドウで以下を確認:
   - Lv1〜10のすべての値が期待結果と一致する
   - 範囲外の値でWarningログが出力される
6. テストスクリプト（SkillEffectsTest）がある場合、自動実行されることを確認
7. Playモードを終了

---

## 11. 成果物

### 変更ファイル
- **新規作成**: `/Assets/WoodcutterCamp/Scripts/Progression/SkillEffects.cs`
- **新規作成**: `/Assets/WoodcutterCamp/Prefabs/Core/SkillEffects.prefab`
- **新規作成（オプション）**: `/Assets/WoodcutterCamp/Scripts/Tests/SkillEffectsTest.cs`

### 追加テストコード
- SkillEffects.cs 内の `_DebugPrintAllEffects()` メソッド
- SkillEffects.cs 内の `_DebugPrintSingleLevel(int level)` メソッド
- （オプション）SkillEffectsTest.cs

### 更新したドキュメント
- なし（本作業指示書のみ）

---

## 12. 備考

### 注意点

#### Pure関数としての設計
- **状態を持たない**: クラス変数にSkillManagerの参照を保持しない
- **副作用なし**: メソッド呼び出しが外部状態を変更しない
- **同じ入力→同じ出力**: level=5なら常にdamage=1.25を返す
- **テストしやすい**: 依存関係が少なく、単体テストが容易

#### UdonSharpの制約
- `Mathf.Clamp()` はUdonSharpで使用可能（ただし、if文で実装）
- `var` キーワードは使用不可（明示的な型指定が必要）
- LINQ式は使用不可
- **Update()ループを使用しない**（すべてのメソッドは呼び出し時に計算）

#### パフォーマンス考慮事項
- すべてのメソッドは純粋関数（状態を持たない）
- 計算負荷が極めて低い（1回の計算時間 < 0.1ms）
- Quest最適化: 複雑な数式を避け、定数とif文のみで実装
- **毎フレーム呼び出されても問題ない軽量さ**

#### 今後の拡張性
- Phase 2でスキル効果を追加する場合、本クラスに新しいメソッドを追加するだけで対応可能
- 例: `GetCarryingSpeedBonus(int level)`, `GetCraftingTimeReduction(int level)` など
- Pure関数なので、新しいメソッド追加が既存機能に影響しない

### 制約事項
- **レベル範囲**: 1〜10固定（Phase 1）
- **効果の種類**: ダメージ、クリティカル、丸太ボーナスの3種類のみ
- **計算式の変更**: Phase 1では固定（バランス調整はPhase 2以降）

### 依存関係
- **依存先**: なし（完全に独立したモジュール）
- **依存元**: AxeInteraction（M-11）、LogSpawner（M-12）
  - これらのモジュールはSkillEffectsのメソッドを呼び出してダメージ/ボーナス計算を行う
  - 呼び出し側がSkillManagerからレベルを取得し、SkillEffectsに渡す

### 参考資料
- **TDD.md**: Module M-05定義（Line 283-311）
- **PRDv3.md**: ダメージ計算式（Line 176-186）、レベル設計（Line 143-158）
- **FSD.md**: スキル効果詳細（Line 586-760）
- **UdonSharp Documentation**: https://udonsharp.docs.vrchat.com/

### トラブルシューティング

#### 問題: UdonSharpでコンパイルエラーが発生する
**原因**: UdonSharp 1.1.9では一部のC#機能が制限されている
**対処法**:
- `var` の使用を避け、明示的な型指定を使う（`float` `int` など）
- LINQ式を使わない（`Where()` `Select()` など）
- 複雑な式を避け、シンプルなif-elseで実装

#### 問題: デバッグメソッドが実行されない
**原因**: UdonSharpのCustomEventが正しく設定されていない
**対処法**:
- メソッド名が `_` で始まっていることを確認（UdonSharp規約）
- メソッドが `public` であることを確認
- Playモード中にInspectorから Custom Event を実行

#### 問題: テスト結果が期待値と異なる
**原因**: 計算式の実装ミス
**対処法**:
- 各メソッドの計算式を再確認（PRDv3.mdと照合）
- Debug.Logで中間値を出力して検証
- デバッグメソッドで全レベルの値を一覧表示

#### 問題: Warning "Level X is greater than 10" が出る
**原因**: バリデーションが正常に動作している（異常ではない）
**対処法**:
- これは期待通りの動作（異常値を検出している）
- 呼び出し側（AxeInteractionなど）で正しいレベル値を渡せば解消

---

## 13. 実装完了後の次ステップ

### 1. Gitコミット

```bash
git add .
git commit -m "feat: Implement SkillEffects calculation as Pure Functions (WI-0006)

- Add GetDamageMultiplier() method (F-13)
- Add GetCriticalRate() method (F-14)
- Add GetLogBonusRate() method (F-15)
- Implement as Pure Functions (no state, no side effects)
- Add input validation for level range (1-10)
- Add debug methods for testing
- Quest-optimized: lightweight calculations only

Refs: TDD.md M-05, PRDv3.md Section 2.2.2, FSD.md F-13/F-14/F-15"
```

### 2. プルリクエスト作成

- ブランチ `feature/WI-0006-skill-effects` から `main` へのPRを作成
- **PRタイトル**: `[WI-0006] Implement SkillEffects Calculation (Pure Functions)`
- **PR説明**:
  ```markdown
  ## 概要
  スキルレベルに応じた効果計算（ダメージボーナス、クリティカル率、丸太ボーナス率）をPure関数として実装

  ## 変更内容
  - SkillEffects.cs 新規作成（M-05）
  - 3つの計算メソッド実装（F-13, F-14, F-15）
  - Pure関数設計（状態なし、副作用なし）
  - Quest最適化済み（軽量計算のみ）

  ## テスト結果
  - デバッグテスト: 全レベル（1-10）で期待値と一致
  - バリデーション: 範囲外の値で正常にWarning表示

  ## スクリーンショット
  （Consoleのデバッグログ出力を添付）

  ## チェックリスト
  - [x] コンパイルエラー0件
  - [x] UdonSharpビルド成功
  - [x] デバッグテスト実施済み
  - [x] XMLコメント記載済み
  ```
- レビュアーにコードレビューを依頼

### 3. 次の作業

- **WI-0007**: TreeController State Machine実装（M-10）
  - 木の状態管理
  - SkillEffectsを直接は使用しない
- **WI-0009**: AxeInteraction Detection実装（M-11）
  - **SkillEffectsを呼び出してダメージ計算を行う**
  - 最終ダメージ = 基礎ダメージ × skillEffects.GetDamageMultiplier(level) × 斧倍率
  - クリティカル判定 = Random.value < skillEffects.GetCriticalRate(level)

---

**作成者**: VRChat World Development Team
**最終更新**: 2025-11-17
**関連Work Instructions**: WI-0005（SkillManager - 前提条件）
**バージョン**: 2.0（Pure関数方式）
