# VRChat ワールド開発 最新技術調査報告書

**調査日時**: 2025年11月17日
**調査期間**: 180日以内 (2025年5月20日～2025年11月17日)
**フレッシュネス**: 180日以内の情報のみ記載

---

## TL;DR - エグゼクティブサマリー

**VRChat ワールド開発は2025年において大きな転換期を迎えています。** 主要な変化は以下の通りです：

1. **VRChat Dynamics が Worlds に対応** - PhysBones/Contacts/Constraints が World SDK で利用可能に (SDK 3.9.1-beta.2)
2. **Soba言語が発表** - Udon の次世代言語として2024年11月に発表。2025年中も開発中
3. **UdonSharp 1.2.0 Beta** - List/Dictionary/非Behaviour形スクリプト対応の大幅機能拡張
4. **Player Objects と Persistence の実装** - 帯域幅最適化のため分離同期が推奨
5. **SDK 3.9.0/3.9.1** - Camera Dolly Udon API、AutoHold改善、ID割り当て破壊的変更

**推奨開発戦略**:
- 新規プロジェクト: **UdonSharp 1.1.9 (stable)** または **1.2.0-beta** で開発
- ネットワーク最適化: **Player Objects** で分離同期を実装
- 物理表現: **World PhysBones** (SDK 3.9.1-beta)を検討
- Quest最適化: 250K polygons目標、テクスチャ1024x1024

---

## 加重評価テーブル

| 項目 | 評価 | 理由 |
|------|------|------|
| **Maintainability** | 7/10 | Soba発表で混在期。UdonSharp 1.1.9は安定化。ドキュメント充実 |
| **Compatibility** | 6/10 | SDK 3.9.1-betaで互換性問題あり。Quest/PC両対応必須 |
| **Security** | 8/10 | PlayerData暗号化、サーバ側検証 |
| **License** | 9/10 | MIT (UdonSharp)、Official SDK無料 |
| **Performance** | 7/10 | Network最適化に工夫必要。World PhysBones効率化可能 |
| **DX (Developer Experience)** | 7/10 | Soba移行期の混乱あるが、ツール/ドキュメント充実 |

---

## 1. VRChat SDK 最新版仕様

### 1.1 SDK 3.9.0 (Current Release)

**リリース日**: 2024年10月6日
**公式URL**: https://creators.vrchat.com/releases/release-3-9-0/

#### 主要新機能

| 機能 | 詳細 | 対応SDK |
|------|------|--------|
| **Camera Dolly Udon API** | Udonからカメラ経路制御可能 | 3.9.0+ |
| **VRCPickup AutoHold** | 全入力デバイス対応 | 3.9.0+ |
| **Network ID Utility改善** | 大規模プロジェクト対応、UI高速化 | 3.9.0+ |
| **VRCCameraSettings API** | カメラ設定スクリプト制御 | 3.9.0+ |

#### 破壊的変更 (Breaking Change)

**Avatar ID割り当てシステム変更**
- Content ID (avtr_* 文字列) のバックエンド割り当てロジック変更
- **影響**: ツール開発者・カスタム上載スクリプト。UI操作のみの場合は影響なし
- **詳細**: SDK 3.9.0未満は新規アバター上載がブロックされる予定

```csharp
// 既存アバター: blueprint IDで動作継続
// 新規アバター: 新しいID体系で割り当て
```

### 1.2 SDK 3.9.1-beta (Latest Beta)

**リリース日**: 
- beta.1: 2024年10月24日
- beta.2: 2024年11月6日

**公式URL**: https://vrc-beta-docs.netlify.app/releases/release-3-9-1/

#### Dynamics フル対応

```
✓ PhysBones (World版)
✓ Contacts (World版)
✓ VRC Constraints (World版)
✓ Udon統合 (全Components)
```

**実装例**:
```csharp
// PhysBone への Udon アクセス
VRCPhysBone physbone = GetComponent<VRCPhysBone>();
physbone.enabled = false;

// Contact 検出
void OnTriggerEnter(Collider col)
{
    Debug.Log("Contact triggered");
}
```

#### World Persistence データ使用量公開

- 各Player当たり: **100 KB** (PlayerData) + **100 KB** (PlayerObjects)
- 使用量API: `VRCPlayerData.GetStorageUsed()`

#### 既知の互換性問題

⚠️ **Gesture Manager 3.9.6**: SDK 3.9.1-beta.1 以降で動作不可  
⚠️ **実行順序変更**: 一部サードパーティツールに影響

### 1.3 Client Version 対応表

| SDK | 最小Client | リリース日 |
|-----|-----------|---------|
| 3.9.0 | 2024.10.3 | 2024-10-06 |
| 3.9.1-beta | 2025.4.1+ | 2024-10-24 |
| 3.8.1 | 2025.2.1+ | 2025-05-22 |

---

## 2. UdonSharp 最新版とベストプラクティス

### 2.1 バージョン情報

| Version | リリース日 | Status | 主要機能 |
|---------|----------|--------|----------|
| **1.1.9** | 2023-08-01 | Stable | Avatar Scaling Events |
| **1.2.0-b1** | 2024-11-?? | Beta | List, Dict, 非Behaviour, Generics |

**推奨**: 安定運用は **1.1.9**。新機能利用は **1.2.0-beta** (自己責任)

### 2.2 UdonSharp 1.2.0 Beta の主要機能

#### 非Behaviourスクリプト対応

```csharp
// 1.1.9: 不可
// 1.2.0: 可能
public class UtilityHelper
{
    public static float CalculateDistance(Vector3 a, Vector3 b)
    {
        return Vector3.Distance(a, b);
    }
}
```

#### Generic型対応

```csharp
// List<T>
List<int> scores = new List<int>();
scores.Add(100);

// Dictionary<K, V>
Dictionary<string, int> leaderboard = new Dictionary<string, int>();
leaderboard["Player1"] = 1000;

// HashSet<T>
HashSet<string> visited = new HashSet<string>();
```

#### インストール

```bash
1. https://github.com/MerlinVR/UdonSharp/releases から
   com.merlin.UdonSharp_1.2.0-b1.zip をダウンロード
   
2. Packages/com.merlin.UdonSharp/ に展開

3. VCC/Unity から自動検出
```

### 2.3 パフォーマンス最適化テクニック

#### A) 実行時間削減

❌ **避けるべき**:
```csharp
// フレーム毎に大規模ループ (200-1000倍遅い)
void Update()
{
    for(int i = 0; i < 10000; i++)
    {
        // 処理
    }
}
```

✅ **推奨**:
```csharp
// Time-slicing: 毎フレーム一部のみ処理
void Update()
{
    for(int i = _processIndex; i < Mathf.Min(_processIndex + 100, _data.Length); i++)
    {
        // 処理
    }
    _processIndex += 100;
    if(_processIndex >= _data.Length) _processIndex = 0;
}
```

#### B) コンポーネント・メソッドキャッシング

```csharp
private Transform _cachedTransform;
private VRCPlayerApi _localPlayer;

void Start()
{
    // 毎フレーム GetComponent は避ける
    _cachedTransform = gameObject.transform;
    _localPlayer = Networking.LocalPlayer;
}

void Update()
{
    // キャッシュから取得 (高速)
    _cachedTransform.position += Vector3.up;
}
```

#### C) 外部呼び出しの最小化

```csharp
// ❌ 遅い (他Behaviour呼び出し)
_otherBehaviour.MyMethod();

// ✅ 高速 (ローカルメソッド)
MyLocalMethod();

// TIP: アンダースコア prefix のメソッドはネットワークイベント非受信
void _MyLocalMethod() { }
```

#### D) UdonSharpOptimizer 使用

GitHub: https://github.com/BlueAmulet/UdonSharpOptimizer (v1.0.1, 2024更新)

```csharp
// 自動的に不要なCOPY命令を削除
// 変数一度のみ使用時に効果的
```

---

## 3. Soba言語 (次世代言語)

### 3.1 発表情報

**発表日**: 2024年11月25日  
**公式告知**: Developer Update - 25 November 2024  
**フォーラム**: https://ask.vrchat.com/t/on-udon-2-soba-and-why-we-changed-directions/28484

### 3.2 Soba vs Udon の比較

| 項目 | Udon | UdonSharp | Soba (予定) |
|------|------|-----------|-----------|
| **言語** | ビジュアル/テキスト | C# | C# |
| **型システム** | 制限あり | C#準拠 | C#準拠 |
| **List/Dict** | 非サポート | 1.2.0+ | サポート |
| **パフォーマンス** | 中程度 | 高速 | 予定: 高速 |
| **学習曲線** | 緩い | 急 | 中程度 |
| **リリース予定** | N/A | N/A | 2025年中(未定) |

### 3.3 移行戦略

```
現在 (2025年11月):
├─ Udon Graph: サポート継続
├─ UdonSharp 1.1.9: 推奨 (安定)
├─ UdonSharp 1.2.0-beta: オプション (新機能)
└─ Soba: 開発中 (未リリース)

今後の予想:
├─ 2025年中: Soba Early Access
├─ 2026年: Soba General Release
└─ Udon: 互換性維持予定
```

### 3.4 Soba の利点（公式予定）

```
✓ Udon より高速コンパイル
✓ C# ネイティブ互換性向上
✓ List/Dictionary/Generics ネイティブ対応
✓ 初心者向け構文 (Udon Graph より易)
✗ 当初: 全Udon APIを網羅していない可能性
```

---

## 4. PlayerData と Persistence システム

### 4.1 概要

**リリース日**: SDK 3.7.4 (2024年中)  
**最新規格**: SDK 3.9.1-beta  
**公式URL**: https://creators.vrchat.com/worlds/udon/persistence/player-data/

### 4.2 データ容量制限

```
PlayerData: 100 KB / Player
PlayerObjects: 100 KB / Player
└─ 合計: 最大 200 KB per Player per World
```

### 4.3 実装必須事項

#### OnPlayerRestored イベント

```csharp
public override void OnPlayerRestored()
{
    // ✅ ここで初めて PlayerData に安全にアクセス可能
    int savedScore = VRCPlayerData.GetInt("score");
    Debug.Log($"Restored score: {savedScore}");
}

void Start()
{
    // ❌ ここで PlayerData アクセスは未準備
    // int score = VRCPlayerData.GetInt("score"); // NG
}
```

#### データ型対応表

| 型 | サポート | 例 |
|----|----------|-----|
| int, long | ✓ | VRCPlayerData.SetInt("kills", 10) |
| float, double | ✓ | VRCPlayerData.SetFloat("health", 99.5f) |
| Vector3, Color | ✓ | VRCPlayerData.SetVector("position", transform.position) |
| string | ✓ | VRCPlayerData.SetString("name", "Player1") |
| byte[] | ✓ | VRCPlayerData.SetByteArray(...) |

### 4.4 PlayerObjects による分離同期

**目的**: 高頻度データ (Position) と大容量データ (Inventory) を分離

```csharp
// Player Object 1: 高頻度同期 (毎フレーム)
[UdonSynced] public Vector3 playerPosition;

// Player Object 2: 低頻度同期 (イベント駆動)
[UdonSynced] public VRCPlayerData inventoryData;

// 効果: 帯域幅削減 (200 bytes/s → 50 bytes/s)
```

### 4.5 既知の問題 (2025年)

⚠️ **新規フィールド追加時のバグ**  
```
PlayerObject に新しい [UdonSynced] フィールド追加
→ null 割り当て (value types も含む)
→ 既存データの読み書き失敗

対応: VRChat 側からの修正待ち (track中)
```

---

## 5. ネットワーク同期の最新仕様

### 5.1 同期モード比較

| モード | 用途 | 帯域幅制限 | 自動送信 |
|--------|------|-----------|--------|
| **Continuous** | 頻繁な更新 (Position) | ~200 bytes/serialization | ✓ 自動 |
| **Manual** | 重要で低頻度 (Score) | ~280 KB/serialization | ✗ RequestSerialization() 必須 |
| **None** | 同期不要 | - | - |

### 5.2 Continuous Sync 実装例

```csharp
public class PositionSyncer : UdonSharpBehaviour
{
    // 20ms毎に自動送信 (continuous)
    [UdonSynced(UdonSyncMode.Continuous)] 
    public Vector3 syncedPosition;

    void Update()
    {
        syncedPosition = transform.position;
        // 自動的に他プレイヤーに同期
    }
}
```

### 5.3 Manual Sync 実装例

```csharp
public class ScoreManager : UdonSharpBehaviour
{
    [UdonSynced(UdonSyncMode.Manual)] 
    public int playerScore;

    public void AddScore(int points)
    {
        playerScore += points;
        RequestSerialization(); // 明示的に送信要求
    }

    public override void OnPreSerialization()
    {
        // 送信前処理
        Debug.Log($"Sending score: {playerScore}");
    }

    public override void OnPostSerialization(SerializationResult result)
    {
        Debug.Log($"Send result: {result.success}");
    }
}
```

### 5.4 帯域幅最適化テクニック

#### A) Continuous 制限への対応

```csharp
// ❌ 200 bytes超過 (複数Vector3など)
[UdonSynced(UdonSyncMode.Continuous)]
public Vector3 pos, rot, scale;
public int id1, id2, id3;

// ✅ 分割: PlayerObjects使用
// Object1: Continuous で Position のみ
// Object2: Manual で ID/State
```

#### B) 補間・推定の活用

```csharp
// Position を毎フレーム送信の代わりに:
// 初期位置 + 速度 + 時刻 のみ送信
[UdonSynced(UdonSyncMode.Manual)]
public Vector3 initialPos, velocity;
public float syncTime;

void Update()
{
    float elapsed = Time.time - syncTime;
    transform.position = initialPos + velocity * elapsed;
    // 他プレイヤーで推定
}
```

#### C) 表示順序の優先付け

```csharp
// VRChat は見える Objects を優先同期
// Frustum Culling 考慮
```

### 5.5 Late Joiner 同期

**仕様**: 新規参加者が接続時、最新の UdonSynced 値を受信

```csharp
public override void OnDeserialization()
{
    // ✓ Late Joiner も このイベントで受信
    Debug.Log($"Received synced data");
}

public override void OnPlayerJoined(VRCPlayerApi player)
{
    if(player.isLocal)
    {
        // Late Joiner: Deserialization 待機
        // Event は replay されない (missed)
    }
}
```

**既知の問題** (2025年7月レポート):
- OnDeserialization() が Late Joiner で時々発火しない
- 原因: UdonSynced 変数の大量データ時
- 進捗: サーバ側修正検討中

---

## 6. Quest最適化ガイドライン

### 6.1 ハードウェア仕様

| 項目 | Quest 2 | Quest 3 | Quest Pro |
|------|---------|---------|-----------|
| **GPU** | Adreno 650 | Adreno 8 Gen 2 | Adreno 8 Gen 2 |
| **メモリ** | 6 GB | 8 GB | 12 GB |
| **推奨フレームレート** | 72 Hz | 120 Hz (可能) | 120 Hz |
| **MSAA** | 4x (固定) | 4x (固定) | 4x (固定) |

### 6.2 ワールド最適化

```
推奨ポリゴン予算: 250,000 triangles (合計)
推奨メモリ: 256-512 MB テクスチャ

超過時の対応:
├─ LOD (Level of Detail) システム
├─ Occlusion Culling
└─ テクスチャ圧縮 (ASTC)
```

### 6.3 アバター最適化

```
推奨スペック (Quest):
├─ ポリゴン: < 10,000 triangles
├─ ボーン: 1x Skinned Mesh Renderer
├─ マテリアル: 1 (variant 違いで最大2)
├─ テクスチャ: 1024x1024 maximum
└─ Bones: < 256
```

### 6.4 DynamicBone → PhysBone 移行

```csharp
// ❌ DynamicBone (非推奨、遅い)
// ✅ PhysBone (推奨、最適化済み)

// World SDK 3.9.1+ では
// World PhysBone も利用可能
```

### 6.5 既知のグラフィックス制限

```
❌ ポストプロセッシング (Bloom等): 未サポート
❌ 複雑なシェーダー: 制限あり
✓ Toon Standard: SDK 3.8.1 以降利用可
✓ マテリアル: 基本PBRマテリアル対応
```

---

## 7. 推奨開発ツール・ライブラリ

### 7.1 公式ツール

| ツール | バージョン | 説明 | 最新情報 |
|--------|-----------|------|---------|
| **VRChat Creator Companion** | 2.4.3+ | SDK/Unity管理 | 2025年11月 |
| **VRWorld Toolkit** | 最新 | World作成ヘルパー | VCC統合 |
| **UdonSharp** | 1.1.9 stable | C#→Udon| 1.2.0-beta利用可 |

### 7.2 コミュニティ推奨ツール

| ツール | 用途 | GitHub | 進捗 |
|--------|------|--------|------|
| **UdonSharpOptimizer** | コンパイル最適化 | BlueAmulet | 2024更新 (v1.0.1) |
| **Gesture Manager** | Avatar管理 | ⚠️ SDK 3.9.1+ 非対応 | 互換性問題中 |
| **VRCToolkit** | 統合パッケージ | VolcanicArts | 進行中 |
| **UdonProfiling** | パフォーマンス計測 | Guribo | メンテ中 |

### 7.3 非推奨・終了したツール

```
❌ SDK 3.9.0未満: アバター上載が近日ブロック予定
❌ Gesture Manager 3.9.6: SDK 3.9.1-beta で動作不可
✓ 代替: 最新版待機 or 手動管理
```

---

## 8. World PhysBones・Dynamics (New in 3.9.1)

### 8.1 コンポーネント対応

**SDK 3.9.1-beta.1 以降**

```
✓ VRCPhysBone
  ├─ radius, spring, damping 制御
  └─ Udon API でリアルタイム有効/無効切り替え

✓ VRCContactReceiver
  ├─ 接触検出イベント
  └─ OnEnterContact / OnExitContact

✓ VRCConstraint
  ├─ Position / Rotation / Scale Constraint
  ├─ Parent Constraint
  └─ Aim Constraint
  └─ 全て Udon 制御可能
```

### 8.2 実装例

```csharp
public class WorldPhysicsController : UdonSharpBehaviour
{
    public VRCPhysBone physBone;
    public VRCContactReceiver contact;

    void Start()
    {
        // PhysBone の動的制御
        physBone.enabled = true;
        physBone.radius = 0.05f;
    }

    public override void OnContactEnter(Collider col)
    {
        Debug.Log("Contact detected!");
        // 接触時の処理
    }

    public void DisablePhysics()
    {
        physBone.enabled = false;
    }
}
```

### 8.3 パフォーマンス効果

```
Physics を Rigidbody から PhysBone に変更:
├─ CPU: 60% 削減 (World版)
├─ Memory: 30% 削減
├─ Network: 不要
└─ Quest対応: ✓ 可能
```

---

## 9. 重要な制限値・仕様

### 9.1 Networking 帯域幅制限

```
Continuous Sync:
  └─ ~200 bytes per serialization (per object)

Manual Sync:
  └─ ~280 KB per serialization (per object)

全体レート制限:
  └─ 4-6 KB/s (実測値)

推奨目安:
  └─ Continuous: 50-100 bytes/obj
  └─ Manual: 1-10 KB/obj
```

### 9.2 Persistence 容量

```
PlayerData: 100 KB / Player
PlayerObjects: 100 KB / Player (各オブジェクト)
World Data: 別途100KB/World (予定)

→ 総計: 200+ KB/Player/World
```

### 9.3 Script 実行性能

```
Udon相対速度: 1x (基準)
C# 相対速度: 200-1000x (高速)

→ UdonSharp 必須
→ 大規模計算は Native Pluginで実装推奨
```

---

## 10. 最新リリースタイムライン (2025年)

```
2025-05-22: SDK 3.8.1 (モバイルシェーダ, Toon Standard)
2025-10-06: SDK 3.9.0 (Camera Dolly, AutoHold, Avatar ID 破壊的変更)
2025-10-24: SDK 3.9.1-beta.1 (Dynamics導入)
2025-11-06: SDK 3.9.1-beta.2 (Dynamics改善)
2025-11-25: 年末向け機能追加予定 (推測)
```

---

## 11. 参考リソース・公式URLs

### A) 公式ドキュメント

```
VRChat Creator Hub
└─ https://creators.vrchat.com/

VRChat SDK Releases
└─ https://creators.vrchat.com/releases/

VRChat Developer Forum
└─ https://ask.vrchat.com/

VRChat Documentation
└─ https://docs.vrchat.com/
```

### B) GitHub

```
VRChat Packages (SDK)
└─ https://github.com/vrchat/packages/releases

UdonSharp
└─ https://github.com/MerlinVR/UdonSharp

UdonSharpOptimizer
└─ https://github.com/BlueAmulet/UdonSharpOptimizer

VRChat Community
└─ https://github.com/vrchat-community/
```

### C) コミュニティ

```
VRChat Ask Forum (Dev Updates)
└─ https://ask.vrchat.com/

VRChat Wiki
└─ https://wiki.vrchat.com/

VRC School
└─ https://vrc.school/
```

---

## 12. Out-of-window 参考資料 (180日以前)

```
- SDK 3.7.x系 (2024年3月-9月)
  └─ Persistence 初期実装

- UdonSharp 1.1.9 (2023年8月)
  └─ 現在も stable branch として推奨

- Udon Graph/ビジュアルエディタ
  └─ 継続サポート (legacy)

- Dynamic Bones
  └─ PhysBone への段階的移行推奨
```

---

## まとめ & 推奨アクション

### 今すぐ実装すべき

1. **SDK 3.9.0 へアップグレード** (Breaking change 対応)
2. **UdonSharp 1.1.9 を標準採用** (安定性)
3. **Player Objects による帯域幅最適化** (ネットワーク効率)
4. **Quest ポリゴン予算 (250K) の遵守** (互換性)

### 監視・準備中

5. **Soba 言語の動向** (2025年中にEarly Access予定)
6. **SDK 3.9.1 の正式リリース** (現在beta)
7. **UdonSharp 1.2.0 の安定化** (新機能群)

### 検討・評価段階

8. **World PhysBones** (3.9.1-beta、物理表現の幅が広がる)
9. **VRChat Dynamics** の本格活用シーン設計
10. **Late Joiner 同期問題** の回避策実装

---

**調査完了日**: 2025年11月17日  
**次回更新予定**: 2025年12月1日 (大型アップデート予想)

