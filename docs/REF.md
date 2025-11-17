# VRChatワールド作成 ツール・ライブラリ APIリファレンスマニュアル

**バージョン**: 1.0
**最終更新**: 2025-11-17
**対象**: VRChat SDK 3.9.0 / Unity 2022.3.22f1 / UdonSharp 1.1.9

---

## 目次

1. [概要](#1-概要)
2. [公式ツール・SDK](#2-公式ツールsdk)
   - [2.1 VRChat SDK 3.9.0](#21-vrchat-sdk-390)
   - [2.2 Udon](#22-udon)
   - [2.3 UdonSharp](#23-udonsharp)
   - [2.4 ClientSim](#24-clientsim)
   - [2.5 VRChat Creator Companion](#25-vrchat-creator-companion)
3. [サードパーティライブラリ](#3-サードパーティライブラリ)
   - [3.1 VRWorld Toolkit](#31-vrworld-toolkit)
   - [3.2 UdonUtils](#32-udonutils)
   - [3.3 VUdon](#33-vudon)
4. [動画・音楽システム](#4-動画音楽システム)
   - [4.1 AudioLink](#41-audiolink)
   - [4.2 ProTV](#42-protv)
   - [4.3 VideoTXL](#43-videotxl)
5. [最適化ツール](#5-最適化ツール)
   - [5.1 VRCQuestTools](#51-vrcquesttools)
   - [5.2 Thry's Avatar Performance Tools](#52-thrys-avatar-performance-tools)
6. [外部ツール](#6-外部ツール)
   - [6.1 VRCX](#61-vrcx)
7. [実装ガイド](#7-実装ガイド)
8. [パフォーマンス最適化](#8-パフォーマンス最適化)
9. [トラブルシューティング](#9-トラブルシューティング)
10. [付録](#10-付録)

---

## 1. 概要

### 1.1. VRChatワールド開発エコシステム

VRChatワールド開発は以下の層で構成されています：

```
┌─────────────────────────────────────┐
│   VRChat Creator Companion (VCC)   │ ← パッケージ管理
├─────────────────────────────────────┤
│        Unity 2022.3.22f1           │ ← エディタ
├─────────────────────────────────────┤
│      VRChat SDK 3.9.0 (Worlds)     │ ← 公式SDK
├─────────────────────────────────────┤
│  Udon / UdonSharp / Soba（開発中） │ ← スクリプトシステム
├─────────────────────────────────────┤
│   サードパーティライブラリ群         │ ← 拡張機能
└─────────────────────────────────────┘
```

### 1.2. 2024年の主要な動向

#### 新機能・アップデート
- **VRChat SDK 3.9.0** (2024-10-06): Camera Dolly API、Auto Hold Pickup
- **Soba発表** (2024-11-25): Udon 2に代わる新プログラミング言語
- **Player Objects** (2024-10-11): CyanPlayerObjectPool置き換え
- **VRWorld Toolkit 3.3.2** (2024-11-09): 90以上の診断機能
- **VideoTXL 2.5.0-beta.12** (2024-11-06): イベント向け動画プレイヤー

#### 非推奨・アーカイブ
- **CyanPlayerObjectPool** → VRChat公式Player Objects
- **CyanTrigger** → Udon Graph / UdonSharp
- **CyanEmu** → ClientSim統合

### 1.3. 推奨ツールセット

#### 必須ツール（公式）
1. VRChat SDK 3.9.0
2. UdonSharp 1.1.9（安定版）
3. ClientSim 1.2.7
4. VRChat Creator Companion 2.4.0

#### 強く推奨（サードパーティ）
1. VRWorld Toolkit 3.3.2 - デバッグ・最適化
2. AudioLink 2.1.0 - 音楽同期（音楽ワールド向け）
3. ProTV 3.0-beta / VideoTXL 2.5.0-beta.12 - 動画（イベントワールド向け）
4. VRCQuestTools 2.11.4 - Quest最適化（アバターワールド向け）

---

## 2. 公式ツール・SDK

### 2.1 VRChat SDK 3.9.0

**リリース日**: 2024-10-06
**ライセンス**: VRChat公式
**ドキュメント**: https://creators.vrchat.com/releases/release-3-9-0/

#### 2.1.1 主要機能

##### Camera Dolly API
カメラドリーパスを定義し、ワールド内でカメラを制御可能。

```csharp
using VRC.SDK3.Components;
using UdonSharp;
using UnityEngine;

public class CameraDollyExample : UdonSharpBehaviour
{
    public VRCCameraDollyAnimation dollyAnimation;
    public VRCCameraDollyPath dollyPath;

    public void StartCameraDolly()
    {
        // カメラドリーパスの開始
        dollyAnimation.Play();
    }

    public void StopCameraDolly()
    {
        dollyAnimation.Stop();
    }
}
```

**VRCCameraDollyAnimation**

| メソッド | 説明 | パラメータ | 戻り値 |
|---------|------|-----------|--------|
| `Play()` | ドリーアニメーション開始 | なし | void |
| `Stop()` | ドリーアニメーション停止 | なし | void |
| `Pause()` | ドリーアニメーション一時停止 | なし | void |

**VRCCameraDollyPath**

| プロパティ | 型 | 説明 |
|----------|-----|------|
| `points` | VRCCameraDollyPoint[] | ドリーパスのポイント配列 |
| `looping` | bool | ループ再生するか |

##### Auto Hold Pickup
新しいピックアップモード - トリガーを離してもオブジェクトを保持。

```csharp
using VRC.SDK3.Components;
using UdonSharp;
using UnityEngine;

public class AutoHoldPickupExample : UdonSharpBehaviour
{
    private VRC_Pickup pickup;

    void Start()
    {
        pickup = GetComponent<VRC_Pickup>();
        pickup.AutoHold = VRC_Pickup.AutoHoldMode.AutoHold;
    }
}
```

**VRC_Pickup.AutoHoldMode**

| 値 | 説明 |
|----|------|
| `None` | デフォルト（トリガー離すとドロップ） |
| `AutoHold` | 自動保持（トリガー離しても保持） |

##### VRCCameraSettings API拡張
ClientSim内でStartから利用可能に。

```csharp
using VRC.SDKBase;
using UdonSharp;

public class CameraSettingsExample : UdonSharpBehaviour
{
    void Start()
    {
        // CullingMaskの設定
        VRCCameraSettings cameraSettings = GetComponent<VRCCameraSettings>();
        if (cameraSettings != null)
        {
            cameraSettings.cullingMask = LayerMask.GetMask("Default", "Water");
        }
    }
}
```

#### 2.1.2 Network ID Utility改善
大規模プロジェクトでのUI応答性向上。

#### 2.1.3 破壊的変更
- コンテンツID（`avtr_*`文字列）の割り当て方式変更
- セマンティックバージョニングに従いメジャーバージョン3.9.0に

#### 2.1.4 主要バグフィックス
- ワールドアップロード時のアバターID割り当て問題修正
- Gestureレイヤー内のTransformアニメーション修正
- ClientSimでのVRCCameraSettings例外処理改善

---

### 2.2 Udon

**最終更新**: SDK 3.9.0（2024-10-06）
**ライセンス**: VRChat公式
**ドキュメント**: https://creators.vrchat.com/worlds/udon/

#### 2.2.1 概要
Udonは VRChatの公式スクリプトシステム。イベント駆動型プログラミングモデルを採用。

#### 2.2.2 主要API

##### VRCPlayerApi
プレイヤー情報・操作のためのAPI。

```csharp
using VRC.SDKBase;
using UdonSharp;

public class PlayerAPIExample : UdonSharpBehaviour
{
    void Start()
    {
        // ローカルプレイヤー取得
        VRCPlayerApi localPlayer = Networking.LocalPlayer;

        // プレイヤー情報取得
        string playerName = localPlayer.displayName;
        int playerId = localPlayer.playerId;
        bool isLocal = localPlayer.isLocal;
        bool isMaster = localPlayer.isMaster;

        // プレイヤー位置取得
        Vector3 playerPosition = localPlayer.GetPosition();
        Quaternion playerRotation = localPlayer.GetRotation();

        // プレイヤー速度取得
        Vector3 playerVelocity = localPlayer.GetVelocity();

        // VRトラッキングデータ取得
        VRCPlayerApi.TrackingData headTracking = localPlayer.GetTrackingData(VRCPlayerApi.TrackingDataType.Head);
        Vector3 headPosition = headTracking.position;
        Quaternion headRotation = headTracking.rotation;
    }
}
```

**VRCPlayerApi主要メソッド**

| メソッド | 説明 | パラメータ | 戻り値 |
|---------|------|-----------|--------|
| `GetPosition()` | プレイヤー位置取得 | なし | Vector3 |
| `GetRotation()` | プレイヤー回転取得 | なし | Quaternion |
| `GetVelocity()` | プレイヤー速度取得 | なし | Vector3 |
| `GetTrackingData(TrackingDataType)` | VRトラッキングデータ取得 | TrackingDataType | TrackingData |
| `TeleportTo(Vector3, Quaternion)` | プレイヤーテレポート | 位置、回転 | void |
| `PlayHapticEventInHand(VRC_Pickup.PickupHand, float, float, float)` | ハプティクスフィードバック | 手、時間、振幅、周波数 | void |

**VRCPlayerApi.TrackingDataType**

| 値 | 説明 |
|----|------|
| `Head` | ヘッドトラッキング |
| `LeftHand` | 左手トラッキング |
| `RightHand` | 右手トラッキング |

##### Networking
ネットワーク操作のためのAPI。

```csharp
using VRC.SDKBase;
using UdonSharp;

public class NetworkingExample : UdonSharpBehaviour
{
    void Start()
    {
        // ローカルプレイヤー
        VRCPlayerApi localPlayer = Networking.LocalPlayer;

        // Master Client確認
        bool isMaster = Networking.IsMaster;

        // オーナーシップ
        Networking.SetOwner(localPlayer, gameObject);
        bool isOwner = Networking.IsOwner(gameObject);

        // インスタンス情報
        int instanceOccupants = VRCPlayerApi.GetPlayerCount();
    }
}
```

**Networking主要メソッド**

| メソッド | 説明 | パラメータ | 戻り値 |
|---------|------|-----------|--------|
| `GetOwner(GameObject)` | オブジェクトのオーナー取得 | GameObject | VRCPlayerApi |
| `SetOwner(VRCPlayerApi, GameObject)` | オブジェクトのオーナー設定 | プレイヤー、GameObject | void |
| `IsOwner(GameObject)` | ローカルプレイヤーがオーナーか | GameObject | bool |
| `IsMaster` | ローカルプレイヤーがMaster Clientか | なし | bool（プロパティ） |
| `LocalPlayer` | ローカルプレイヤー取得 | なし | VRCPlayerApi（プロパティ） |

##### VRCInstantiate
オブジェクトのネットワーク同期インスタンス化。

```csharp
using VRC.SDKBase;
using UdonSharp;
using UnityEngine;

public class InstantiateExample : UdonSharpBehaviour
{
    public GameObject prefab;

    public void SpawnObject()
    {
        // ネットワーク同期されたオブジェクト生成
        GameObject instance = VRCInstantiate(prefab);

        // 位置設定
        instance.transform.position = transform.position;
    }
}
```

#### 2.2.3 イベントシステム

##### 標準イベント

```csharp
using UdonSharp;
using VRC.SDKBase;

public class EventExample : UdonSharpBehaviour
{
    // ワールド参加時
    public override void OnPlayerJoined(VRCPlayerApi player)
    {
        Debug.Log($"{player.displayName} joined");
    }

    // ワールド退出時
    public override void OnPlayerLeft(VRCPlayerApi player)
    {
        Debug.Log($"{player.displayName} left");
    }

    // オーナーシップ移行時
    public override void OnOwnershipTransferred(VRCPlayerApi player)
    {
        Debug.Log($"Ownership transferred to {player.displayName}");
    }

    // Pickup時
    public override void OnPickup()
    {
        Debug.Log("Picked up");
    }

    // Drop時
    public override void OnDrop()
    {
        Debug.Log("Dropped");
    }
}
```

##### カスタムイベント

```csharp
using UdonSharp;

public class CustomEventExample : UdonSharpBehaviour
{
    public void MyCustomEvent()
    {
        Debug.Log("Custom event called");
    }

    void Start()
    {
        // カスタムイベント呼び出し
        SendCustomEvent(nameof(MyCustomEvent));

        // 遅延呼び出し
        SendCustomEventDelayedSeconds(nameof(MyCustomEvent), 2.0f);

        // フレーム遅延呼び出し
        SendCustomEventDelayedFrames(nameof(MyCustomEvent), 10);
    }
}
```

#### 2.2.4 Sobaについて（開発中）

**発表日**: 2024-11-25
**ステータス**: 開発中（クローズド/オープンベータ予定）

Sobaは Udon 2の後継として開発が方向転換された新プログラミング言語：

- C#記法でスクリプト作成（UdonSharp類似）
- Common Intermediate Language（CIL）へコンパイル
- Udon Graph/UdonSharpにない機能をサポート
- オブジェクト指向プログラミング
- より高度なC#機能のサポート
- 選択制: Soba / Udon Graph / UdonSharpから選択可能

**参考**: https://ask.vrchat.com/t/on-udon-2-soba-and-why-we-changed-directions/28484

---

### 2.3 UdonSharp

**最新バージョン**: 1.2-beta1（開発中）/ 1.1.9（安定版）
**リリース日**: 2024-11-23（beta1）
**ライセンス**: MIT
**GitHub**: https://github.com/MerlinVR/UdonSharp
**ドキュメント**: https://udonsharp.docs.vrchat.com/

#### 2.3.1 概要
C#ライクな記法でUdonプログラムを作成できるコンパイラ。

#### 2.3.2 基本構文

```csharp
using UdonSharp;
using UnityEngine;
using VRC.SDKBase;

public class BasicUdonSharpExample : UdonSharpBehaviour
{
    // フィールド（インスペクター表示）
    public GameObject targetObject;
    public float moveSpeed = 1.0f;

    // プライベートフィールド
    private Vector3 startPosition;

    void Start()
    {
        startPosition = transform.position;
        Debug.Log("UdonSharp script started!");
    }

    void Update()
    {
        // 移動処理
        transform.position += Vector3.forward * moveSpeed * Time.deltaTime;
    }

    // カスタムイベント
    public void ResetPosition()
    {
        transform.position = startPosition;
    }
}
```

#### 2.3.3 サポートされている型

##### プリミティブ型
- `int`, `float`, `double`, `bool`, `string`, `char`, `byte`, `sbyte`, `short`, `ushort`, `uint`, `long`, `ulong`

##### Unity型
- `Vector2`, `Vector3`, `Vector4`, `Quaternion`, `Color`, `Color32`
- `Rect`, `Bounds`, `Ray`, `RaycastHit`
- `GameObject`, `Transform`, `Rigidbody`, `Collider`, `Renderer`, `Material`, `Texture2D`, etc.

##### VRChat型
- `VRCPlayerApi`, `VRC_Pickup`, `VRCStation`

##### コレクション型（v1.1+）
- `List<T>`, `Dictionary<TKey, TValue>`, `HashSet<T>`, `Queue<T>`, `Stack<T>`

```csharp
using UdonSharp;
using System.Collections.Generic;

public class CollectionExample : UdonSharpBehaviour
{
    private List<int> scores;
    private Dictionary<string, int> playerScores;
    private HashSet<VRCPlayerApi> players;

    void Start()
    {
        // List初期化
        scores = new List<int>();
        scores.Add(100);
        scores.Add(200);

        // Dictionary初期化
        playerScores = new Dictionary<string, int>();
        playerScores["Player1"] = 100;
        playerScores["Player2"] = 200;

        // HashSet初期化
        players = new HashSet<VRCPlayerApi>();
    }
}
```

#### 2.3.4 ネットワーク同期

##### UdonSynced変数

```csharp
using UdonSharp;

[UdonBehaviourSyncMode(BehaviourSyncMode.Manual)]
public class SyncExample : UdonSharpBehaviour
{
    [UdonSynced]
    private int syncedScore = 0;

    [UdonSynced]
    private string syncedPlayerName = "";

    public void UpdateScore(int newScore)
    {
        // オーナーのみが変更可能
        if (!Networking.IsOwner(gameObject))
        {
            Networking.SetOwner(Networking.LocalPlayer, gameObject);
        }

        syncedScore = newScore;
        RequestSerialization(); // 同期リクエスト
    }

    public override void OnDeserialization()
    {
        // 同期データ受信時
        Debug.Log($"Received score: {syncedScore}");
    }
}
```

**BehaviourSyncMode**

| モード | 説明 |
|-------|------|
| `None` | 同期なし |
| `Continuous` | 自動同期（約200バイト制限） |
| `Manual` | 手動同期（約282KB制限） |

##### PlayerData（永続化）

```csharp
using UdonSharp;
using VRC.SDKBase;

public class PersistenceExample : UdonSharpBehaviour
{
    private const string SCORE_KEY = "PlayerScore";
    private const string LEVEL_KEY = "PlayerLevel";

    void Start()
    {
        // データ読み込み
        VRCPlayerApi localPlayer = Networking.LocalPlayer;
        int savedScore = PlayerData.GetInt(localPlayer, SCORE_KEY);
        int savedLevel = PlayerData.GetInt(localPlayer, LEVEL_KEY);

        Debug.Log($"Loaded: Score={savedScore}, Level={savedLevel}");
    }

    public void SaveProgress(int score, int level)
    {
        VRCPlayerApi localPlayer = Networking.LocalPlayer;

        // データ保存
        PlayerData.SetInt(localPlayer, SCORE_KEY, score);
        PlayerData.SetInt(localPlayer, LEVEL_KEY, level);

        Debug.Log("Progress saved!");
    }
}
```

**PlayerData制限**: 100KB（圧縮後）

#### 2.3.5 パフォーマンス最適化

##### GetComponentキャッシング

```csharp
using UdonSharp;

public class OptimizedScript : UdonSharpBehaviour
{
    // キャッシュされたコンポーネント参照
    private Rigidbody rb;
    private Renderer rend;
    private AudioSource audioSource;

    void Start()
    {
        // Start()で一度だけ取得
        rb = GetComponent<Rigidbody>();
        rend = GetComponent<Renderer>();
        audioSource = GetComponent<AudioSource>();
    }

    void Update()
    {
        // キャッシュを使用（GetComponent呼び出しなし）
        rb.velocity = Vector3.forward;
        rend.material.color = Color.red;
    }
}
```

##### Update()回避

```csharp
using UdonSharp;

public class EventDrivenScript : UdonSharpBehaviour
{
    private float checkInterval = 1.0f;
    private float nextCheckTime = 0;

    // ❌ 悪い例：毎フレーム実行
    // void Update() {
    //     CheckStatus();
    // }

    // ✅ 良い例：タイマーベース
    void Update()
    {
        if (Time.time >= nextCheckTime)
        {
            CheckStatus();
            nextCheckTime = Time.time + checkInterval;
        }
    }

    private void CheckStatus()
    {
        // 処理
    }
}
```

#### 2.3.6 制限事項

UdonSharpで使用できないC#機能：
- Async/Await
- LINQ
- 動的型（dynamic）
- リフレクション
- マルチスレッド
- 一部のC# 8.0以降の機能

---

### 2.4 ClientSim

**最新バージョン**: 1.2.7
**リリース日**: 2024-08-22
**ライセンス**: VRChat公式
**GitHub**: https://github.com/vrchat-community/ClientSim
**ドキュメント**: https://clientsim.docs.vrchat.com/

#### 2.4.1 概要
Unity Editor内でVRChatクライアント動作を再現するローカルテスト環境。

#### 2.4.2 主要機能

- ワールド内を歩き回ってテスト
- Pickup/UI/Stationの動作確認
- ポーズメニュー自動クローズ（SDK 3.7.6+）
- VR/デスクトップモード切り替え

#### 2.4.3 使用方法

1. Unity Editorでワールドシーンを開く
2. Playボタンをクリック
3. ClientSimが起動

**操作方法**:
- WASD: 移動
- マウス: 視点変更
- Shift: ダッシュ
- Space: ジャンプ
- E: インタラクト
- Esc: ポーズメニュー

#### 2.4.4 設定

```
VRChat SDK → ClientSim Settings
```

| 設定項目 | 説明 |
|---------|------|
| Hide Menu on Launch | 起動時にメニュー非表示 |
| Enable Desktop Mode | デスクトップモード有効化 |
| Enable VR Mode | VRモード有効化 |

#### 2.4.5 デバッグ機能

```csharp
using UdonSharp;
using VRC.SDKBase;

public class ClientSimDebugExample : UdonSharpBehaviour
{
    void Start()
    {
        // ClientSim環境判定
        #if UNITY_EDITOR
        Debug.Log("Running in ClientSim");
        #endif

        // VRChatクライアント判定
        if (Networking.LocalPlayer != null)
        {
            Debug.Log("Running in VRChat client or ClientSim");
        }
    }
}
```

---

### 2.5 VRChat Creator Companion

**最新バージョン**: 2.4.0
**リリース日**: 2024-12月
**ライセンス**: VRChat公式
**GitHub**: https://github.com/vrchat-community/creator-companion
**ドキュメント**: https://vcc.docs.vrchat.com/

#### 2.5.1 主要機能

- VRChat SDK管理
- Unityプロジェクト管理
- VPM（VRChat Package Manager）統合
- テンプレート提供
- コミュニティパッケージアクセス

#### 2.5.2 v2.4.0の新機能

- 複数フォルダ同時選択機能
- .NET 8へアップグレード（.NET 6から）

#### 2.5.3 プロジェクト作成

1. VCC起動
2. "New" → "World"選択
3. プロジェクト名入力
4. Unityバージョン選択（2022.3.22f1推奨）
5. "Create Project"

#### 2.5.4 パッケージ追加

```
Manage Project → Packages → Browse
```

検索またはキュレーションリストから選択：
- Official（公式パッケージ）
- Curated（VRChat選定コミュニティパッケージ）
- Community（ユーザー追加パッケージ）

#### 2.5.5 カスタムリポジトリ追加

```
Settings → Packages → Add Repository
```

リポジトリURL入力例：
```
https://guribo.github.io/TLP/index.json
```

---

## 3. サードパーティライブラリ

### 3.1 VRWorld Toolkit

**最新バージョン**: 3.3.2
**リリース日**: 2024-11-09
**ライセンス**: MIT
**GitHub**: https://github.com/oneVR/VRWorldToolkit
**Stars**: 446

#### 3.1.1 概要
ワールド最適化・デバッグに特化したツール群。90以上の診断機能を持つWorld Debugger搭載。

#### 3.1.2 主要機能

##### 1. World Debugger
ワールドの一般的な問題を自動検出・提案。

**診断項目（一部）**:
- ライティング設定の問題
- コライダー設定の問題
- AudioSource設定の問題
- Occlusion Culling設定
- Quest互換性チェック

**アクセス方法**:
```
VRWorld Toolkit → World Debugger
```

##### 2. Disable On Build
"DisableOnBuild"タグのGameObjectを自動無効化。

```csharp
// 使用例：テスト用オブジェクトをビルド時に無効化
// GameObjectに"DisableOnBuild"タグを設定
```

##### 3. Post Processing
ワンクリックでPost Processing設定。

**アクセス方法**:
```
VRWorld Toolkit → Helpers → Setup Post Processing
```

##### 4. Quick Functions

**ワールドIDコピー**:
```
VRWorld Toolkit → Helpers → Copy World ID
```

**バッチテクスチャ処理**:
```
VRWorld Toolkit → Helpers → Batch Texture Processing
```

##### 5. Custom Editors
VRChat標準コンポーネントの拡張エディタ。

- VRCMirrorReflection強化
- VRCAvatarPedestal拡張

#### 3.1.3 導入方法

**方法1: VCC（推奨）**
1. VCCを開く
2. Manage Project → Packages
3. "VRWorld Toolkit"を検索
4. "Add"をクリック

**方法2: 手動**
1. https://github.com/oneVR/VRWorldToolkit/releases
2. 最新リリースのUnityPackageダウンロード
3. Unityプロジェクトにインポート

#### 3.1.4 互換性
- Unity 2022.3.x必須
- Version 3.0.0以降Unity 2019非対応

---

### 3.2 UdonUtils

**最新バージョン**: 12.0.0
**リリース日**: 2024-11-09
**ライセンス**: MIT
**GitHub**: https://github.com/Guribo/UdonUtils
**VCC**: https://guribo.github.io/TLP/

#### 3.2.1 概要
Udon開発を支援するユーティリティ集。

#### 3.2.2 主要機能

##### TLPLogger
ロギングシステム。

```csharp
using TLP.UdonUtils.Runtime;
using UdonSharp;

public class LoggerExample : UdonSharpBehaviour
{
    private TLPLogger logger;

    void Start()
    {
        logger = new TLPLogger("MyScript");

        logger.Info("Info message");
        logger.Warning("Warning message");
        logger.Error("Error message");
        logger.Debug("Debug message");
    }
}
```

**TLPLogger API**

| メソッド | 説明 | パラメータ |
|---------|------|-----------|
| `Info(string)` | 情報ログ | メッセージ |
| `Warning(string)` | 警告ログ | メッセージ |
| `Error(string)` | エラーログ | メッセージ |
| `Debug(string)` | デバッグログ | メッセージ |

##### WorldVersionCheck
ワールドバージョン警告システム。

```csharp
using TLP.UdonUtils.Runtime;
using UdonSharp;

public class VersionCheckExample : UdonSharpBehaviour
{
    public WorldVersionCheck versionCheck;
    public int currentVersion = 1;

    void Start()
    {
        versionCheck.CheckVersion(currentVersion);
    }
}
```

##### TLPNetworkTime
サブミリ秒精度のネットワーク時刻。

```csharp
using TLP.UdonUtils.Runtime;
using UdonSharp;

public class NetworkTimeExample : UdonSharpBehaviour
{
    private TLPNetworkTime networkTime;

    void Start()
    {
        networkTime = GetComponent<TLPNetworkTime>();

        // 現在のネットワーク時刻取得
        double currentTime = networkTime.GetNetworkTime();
        Debug.Log($"Network Time: {currentTime}");
    }
}
```

#### 3.2.3 v12.0.0の重要な変更
- **破壊的変更**: Cyan Object Pool依存削除
- VRChat公式Player Objectsの使用を推奨

#### 3.2.4 導入方法

1. VRChat World SDK 3.9.0をインストール
2. VCC Settings → Add Repository
3. `https://guribo.github.io/TLP/index.json`を追加
4. `TLP_Essentials`プレハブをシーンに追加

---

### 3.3 VUdon

**最新更新**: 2024年初頭
**ライセンス**: MIT
**GitHub**: https://github.com/Varneon/VUdon
**Stars**: 62

#### 3.3.1 概要
UdonSharpプレハブエコシステム（UdonEssentialsの後継）。

#### 3.3.2 主要パッケージ

##### Simple Player Settings
プレイヤー設定UI。

```csharp
// 自動的にUI提供、スクリプト不要
```

##### Noclip
Source風のノークリップ移動。

```csharp
// プレハブをシーンに追加するだけで機能
```

##### Playerlist
インスタンス内プレイヤー表示。

**導入方法**:
1. https://github.com/Varneon/VUdon-Playerlist
2. Releasesから最新版ダウンロード
3. Unityにインポート
4. Playerlistプレハブをシーンに追加

##### Logger
デバッグログシステム。

```csharp
using Varneon.VUdon.Logger;
using UdonSharp;

public class VUdonLoggerExample : UdonSharpBehaviour
{
    public UdonLogger logger;

    void Start()
    {
        logger.Log("Info message");
        logger.LogWarning("Warning message");
        logger.LogError("Error message");
    }
}
```

##### Array Extensions
配列操作拡張機能。

##### Event Dispatcher
イベントディスパッチシステム（Update/FixedUpdate/LateUpdate）。

```csharp
using Varneon.VUdon.EventDispatcher;
using UdonSharp;

public class EventDispatcherExample : UdonSharpBehaviour
{
    public UdonEventDispatcher dispatcher;

    void Start()
    {
        // Updateイベント登録
        dispatcher.RegisterUpdate(this, nameof(OnUpdate));
    }

    public void OnUpdate()
    {
        // Update処理
    }
}
```

##### Quick Menu
HUD quick menu。

##### Udonity
ランタイムUnityエディター。

#### 3.3.3 導入方法
各パッケージのGitHubリポジトリから個別にインストール。

**例: Playerlist**
```
1. https://github.com/Varneon/VUdon-Playerlist
2. Releases
3. 最新.unitypackageダウンロード
4. Unityにインポート
```

---

## 4. 動画・音楽システム

### 4.1 AudioLink

**最新バージョン**: 2.1.0
**ライセンス**: オープンソース
**GitHub**: https://github.com/llealloo/audiolink

#### 4.1.1 概要
音楽反応型システム。リアルタイムオーディオ解析によりシェーダー・オブジェクトを音楽に同期。

#### 4.1.2 主要機能

- リアルタイムオーディオ解析
- GPU処理による振幅・周波数データ配信
- アバターシェーダー対応
- 環境オブジェクト対応
- フェーズ情報サポート（v2.1.0）

#### 4.1.3 導入方法

**VCC経由（推奨）**:
1. VCC → Manage Project → Packages
2. "AudioLink"を検索
3. "Add"をクリック

**手動**:
1. https://github.com/llealloo/audiolink/releases
2. 最新UnityPackageダウンロード
3. Unityにインポート

#### 4.1.4 基本使用

```
1. AudioLinkプレハブをシーンに追加
2. AudioSourceを設定
3. AudioLinkコンポーネントを設定
```

#### 4.1.5 シェーダー側での使用

```hlsl
// AudioLinkテクスチャサンプリング
float4 audioLinkData = tex2D(_AudioTexture, ALPASS_DFT + float2(x, y));

// 周波数バンド取得
float bass = audioLinkData.r;      // 低音
float lowMid = audioLinkData.g;    // 中低音
float highMid = audioLinkData.b;   // 中高音
float treble = audioLinkData.a;    // 高音
```

**ALPASS定数**:

| 定数 | 説明 |
|------|------|
| `ALPASS_DFT` | DFT（離散フーリエ変換）データ |
| `ALPASS_AUDIOLINK` | AudioLinkメインデータ |
| `ALPASS_THEME_COLOR0` | テーマカラー0 |
| `ALPASS_THEME_COLOR1` | テーマカラー1 |
| `ALPASS_THEME_COLOR2` | テーマカラー2 |
| `ALPASS_THEME_COLOR3` | テーマカラー3 |

#### 4.1.6 UdonSharp連携

```csharp
using UdonSharp;
using UnityEngine;

public class AudioLinkExample : UdonSharpBehaviour
{
    public Material audioLinkMaterial;

    void Update()
    {
        // AudioLinkデータ取得
        if (audioLinkMaterial != null)
        {
            Texture audioTexture = audioLinkMaterial.GetTexture("_AudioTexture");
            // テクスチャから周波数データ取得
        }
    }
}
```

#### 4.1.7 v2.1.0の新機能

- **Phase Information Support**: ALPASS_DFTセクションのアルファチャンネル経由でフェーズ情報取得
- **パフォーマンス改善**: グローバル文字列処理最適化
- **デフォルト解像度変更**: AudioLinkAvatarプレハブデフォルト360p
- **Unity 6.x互換性修正**

---

### 4.2 ProTV

**最新バージョン**: 3.0.0-beta（LTS: 2.3.15）
**ステータス**: パブリックベータ
**ライセンス**: オープンソース（寄付制）
**GitLab**: https://gitlab.com/techanon/protv
**ドキュメント**: https://protv.dev/

#### 4.2.1 概要
拡張可能な動画プレイヤーシステム。プラグインベースアーキテクチャ。

#### 4.2.2 主要機能

- Media Controls（再生/停止/シーク）
- Playlists & Queues
- 360 Skybox Video
- VoteSkip
- AudioLink統合
- VRCDN & Questネイティブライブストリーム対応

#### 4.2.3 導入方法

**方法1: VCC（推奨）**
1. VCC → Packages → "ProTV"検索
2. "Add"をクリック

**方法2: Gumroad**
1. https://architechvr.gumroad.com/l/protv
2. 無料ダウンロード
3. Unityにインポート

#### 4.2.4 基本セットアップ

```
1. ProTVプレハブをシーンに追加
2. Screen Managerで再生画面設定
3. Media Controllerで操作UI設定
```

#### 4.2.5 プラグイン作成

```csharp
using UdonSharp;
using ArchiTech;

public class CustomProTVPlugin : UdonSharpBehaviour
{
    public ProTV proTV;

    void Start()
    {
        // プラグイン登録
        proTV.RegisterPlugin(this);
    }

    // カスタムイベント
    public void OnVideoStart()
    {
        Debug.Log("Video started");
    }

    public void OnVideoEnd()
    {
        Debug.Log("Video ended");
    }
}
```

#### 4.2.6 API

**ProTV主要メソッド**

| メソッド | 説明 | パラメータ |
|---------|------|-----------|
| `Play()` | 動画再生 | なし |
| `Pause()` | 動画一時停止 | なし |
| `Stop()` | 動画停止 | なし |
| `LoadURL(string)` | URLから動画読み込み | URL |
| `SetTime(float)` | 再生位置設定 | 時間（秒） |

---

### 4.3 VideoTXL

**最新バージョン**: 2.5.0-beta.12
**リリース日**: 2024-11-06
**ライセンス**: MIT
**GitHub**: https://github.com/vrctxl/VideoTXL
**ドキュメント**: https://vrctxl.github.io/Docs/

#### 4.3.1 概要
イベント向け動画プレイヤー。AVPro & Unity Videoバックエンド対応。

#### 4.3.2 主要機能

- Sync & Local動画プレイヤー
- AVPro & Unity Videoバックエンド
- AudioLink & VRSL統合
- Screen Manager（レンダーテクスチャ対応）
- 複数プレハブバリアント

#### 4.3.3 導入方法

1. VCC Settings → Add Repository
2. `https://vrctxl.github.io/VPM/index.json`を追加
3. Packages → "TXL - VideoTXL"をインストール
4. 依存関係"TXL - CommonTXL"が自動インストール

#### 4.3.4 プレハブバリアント

- **Basic Sync**: 基本同期プレイヤー
- **Local AVPro**: ローカルAVProプレイヤー
- **Full Sync**: フル機能同期プレイヤー

#### 4.3.5 API

```csharp
using UdonSharp;
using Texel;

public class VideoTXLExample : UdonSharpBehaviour
{
    public VideoManager videoManager;

    public void PlayVideo(string url)
    {
        videoManager.PlayURL(url);
    }

    public void PauseVideo()
    {
        videoManager.Pause();
    }

    public void StopVideo()
    {
        videoManager.Stop();
    }
}
```

**VideoManager主要メソッド**

| メソッド | 説明 | パラメータ |
|---------|------|-----------|
| `PlayURL(string)` | URL動画再生 | URL |
| `Play()` | 再生 | なし |
| `Pause()` | 一時停止 | なし |
| `Stop()` | 停止 | なし |
| `SetTime(float)` | シーク | 時間（秒） |

---

## 5. 最適化ツール

### 5.1 VRCQuestTools

**最新バージョン**: v2.11.4
**リリース日**: 2024-10-10
**ライセンス**: MIT
**GitHub**: https://github.com/kurotu/VRCQuestTools
**Stars**: 295

#### 5.1.1 概要
アバターをOculus Quest & PICO互換化するUnity Editor拡張。

#### 5.1.2 主要機能

- アバター複製（オリジナル保持）
- Android用テクスチャ生成
- 非対応コンポーネント削除（Constraintsなど）
- PhysBone削減
- 頂点カラー削除
- Unity設定最適化

#### 5.1.3 導入方法

1. VCC Settings → Add Repository
2. `https://kurotu.github.io/vpm-repos/vpm.html`を追加
3. Packages → "VRCQuestTools"をインストール

#### 5.1.4 使用方法

```
1. アバターを選択
2. VRCQuestTools → Convert Avatar for Quest
3. 設定調整
4. "Convert"をクリック
```

#### 5.1.5 最適化目標

- **ポリゴン数**: 10,000トライアングル未満
- **テクスチャ**: 最大1024x1024
- **Skinned Mesh Renderer**: 1つのみ

#### 5.1.6 自動処理

**削除される要素**:
- Constraints
- Final IK
- Dynamic Bone（PhysBoneに変換）
- Cloth
- Light
- Camera

**テクスチャ最適化**:
- Android向けASTC圧縮
- 解像度ダウンスケール

---

### 5.2 Thry's Avatar Performance Tools

**最新バージョン**: 1.3.7
**リリース日**: 2024-08-05
**ライセンス**: MIT
**GitHub**: https://github.com/Thryrallo/VRC-Avatar-Performance-Tools
**VPM**: https://vpm.thry.dev/

#### 5.2.1 概要
VRAM消費量解析・最適化ツール。

#### 5.2.2 主要機能

##### Avatar Evaluator
- VRAM消費量解析
- Grabpass使用数
- Blendshape数
- Animator遷移カウント
- Write defaults検証

##### VRAM Checker
- テクスチャメモリ使用量計算
- アクティブ/全オブジェクトVRAM表示
- テクスチャ・メッシュVRAMサイズリスト
- アバターVRAMサイズフィードバック

#### 5.2.3 導入方法

**方法1: VCC**
1. VCC Settings → Add Repository
2. `https://vpm.thry.dev/`を追加
3. "Thry's Avatar Performance Tools"をインストール

**方法2: OpenUPM**
```
openupm add com.thryrallo.avatar-performance-tools
```

#### 5.2.4 使用方法

```
1. Unity: Thry → Avatar → VRAM
2. アバターを選択
3. "Calculate VRAM"をクリック
```

#### 5.2.5 VRAM計算結果

**表示項目**:
- **Total VRAM**: 総VRAM使用量
- **Texture VRAM**: テクスチャVRAM
- **Mesh VRAM**: メッシュVRAM
- **Material Count**: マテリアル数
- **Texture Count**: テクスチャ数

---

## 6. 外部ツール

### 6.1 VRCX

**最新バージョン**: 2024.12.30
**リリース日**: 2024-12-30
**ライセンス**: オープンソース
**GitHub**: https://github.com/vrcx-team/VRCX

#### 6.1.1 概要
VRChat補助ツール（外部アプリケーション）。フレンド管理・ワールド検索・Discord統合など。

**注意**: 外部ツール（VRChat非公式）。VRChat APIを使用するが、ゲーム改変なし。

#### 6.1.2 主要機能

##### Friend Management
- フレンドリスト管理
- オンラインステータス確認
- 初回追加日・最終会話日追跡
- 一緒に過ごした時間・回数記録
- フレンド名変更追跡
- ノート保存

##### App Launcher
VRChat起動時にOSCアプリ/ボイスチェンジャー自動起動。

##### Search & Organization
- アバター・ユーザー・ワールド・グループ検索
- ローカル無制限ワールドお気に入りリスト
- ゲーム内写真へのワールドデータ保存

##### Discord Integration
- 現在のインスタンス詳細情報Discord表示
- ワールドサムネイル、名前、インスタンスID、プレイヤー数表示

##### Content Management
- アップロード済みアバター/ワールド情報編集
- Unity不要で情報更新可能
- VRChatクラッシュ時自動再起動・前回インスタンス自動再参加

#### 6.1.3 v2024.12.30の新機能

- ユーザーバッジサポート
- Age-gatedインスタンス作成機能
- ローカルバックアップからのフレンド順序復元
- "Auto change status"拡張オプション
- ワンクリックインスタンス作成付きワールドお気に入り

#### 6.1.4 導入方法

1. https://github.com/vrcx-team/VRCX/releases
2. 最新インストーラーダウンロード
3. インストール実行
4. VRCXログイン

#### 6.1.5 使用例

**フレンドノート追加**:
```
1. VRCX → Friends
2. フレンド選択
3. "Notes"タブ
4. メモ入力
```

**ワールドお気に入り**:
```
1. VRCX → Worlds
2. ワールド検索
3. "Add to Favorites"
```

**Discord統合**:
```
1. VRCX Settings → Discord
2. "Enable Discord Rich Presence"有効化
```

---

## 7. 実装ガイド

### 7.1 ワールド開発の基本フロー

#### 1. プロジェクトセットアップ

```
VCC → New → World → プロジェクト作成
↓
Unity Editorで開く
↓
VRChat SDK設定確認
```

#### 2. シーン作成

```
基本オブジェクト配置（地形、建物、装飾）
↓
ライティング設定（ベイクドライティング推奨）
↓
VRChat Descriptorコンポーネント追加
```

#### 3. Udon/UdonSharpスクリプト作成

```csharp
using UdonSharp;
using UnityEngine;
using VRC.SDKBase;

public class WorldScript : UdonSharpBehaviour
{
    void Start()
    {
        Debug.Log("World loaded!");
    }

    public override void OnPlayerJoined(VRCPlayerApi player)
    {
        Debug.Log($"{player.displayName} joined");
    }
}
```

#### 4. ローカルテスト

```
Unity Editorで再生
↓
ClientSimでテスト
↓
問題修正
```

#### 5. ビルド・アップロード

```
VRChat SDK → Build & Publish
↓
ワールド情報入力
↓
アップロード
```

### 7.2 よくある実装パターン

#### インタラクト可能なオブジェクト

```csharp
using UdonSharp;
using UnityEngine;

public class InteractableObject : UdonSharpBehaviour
{
    public override void Interact()
    {
        Debug.Log("Interacted!");

        // オブジェクトの色を変更
        GetComponent<Renderer>().material.color = Random.ColorHSV();
    }
}
```

#### テレポート実装

```csharp
using UdonSharp;
using UnityEngine;
using VRC.SDKBase;

public class TeleportTrigger : UdonSharpBehaviour
{
    public Transform destination;

    private void OnTriggerEnter(Collider other)
    {
        VRCPlayerApi player = Networking.LocalPlayer;
        if (player != null)
        {
            player.TeleportTo(destination.position, destination.rotation);
        }
    }
}
```

#### ドア開閉

```csharp
using UdonSharp;
using UnityEngine;

public class Door : UdonSharpBehaviour
{
    public Transform doorPivot;
    public float openAngle = 90f;
    public float animationTime = 1f;

    private bool isOpen = false;
    private Quaternion closedRotation;
    private Quaternion openRotation;

    void Start()
    {
        closedRotation = doorPivot.localRotation;
        openRotation = closedRotation * Quaternion.Euler(0, openAngle, 0);
    }

    public override void Interact()
    {
        isOpen = !isOpen;
        SendCustomEvent(nameof(AnimateDoor));
    }

    public void AnimateDoor()
    {
        StopAllCoroutines();
        StartCoroutine(AnimateDoorCoroutine());
    }

    private System.Collections.IEnumerator AnimateDoorCoroutine()
    {
        Quaternion startRot = doorPivot.localRotation;
        Quaternion endRot = isOpen ? openRotation : closedRotation;

        float elapsed = 0;
        while (elapsed < animationTime)
        {
            elapsed += Time.deltaTime;
            float t = elapsed / animationTime;
            doorPivot.localRotation = Quaternion.Slerp(startRot, endRot, t);
            yield return null;
        }

        doorPivot.localRotation = endRot;
    }
}
```

#### マルチプレイヤー同期スコアボード

```csharp
using UdonSharp;
using UnityEngine;
using UnityEngine.UI;
using VRC.SDKBase;
using VRC.Udon.Common.Interfaces;

[UdonBehaviourSyncMode(BehaviourSyncMode.Manual)]
public class Scoreboard : UdonSharpBehaviour
{
    [UdonSynced]
    private string[] playerNames = new string[20];

    [UdonSynced]
    private int[] playerScores = new int[20];

    public Text scoreboardText;

    public void UpdateScore(VRCPlayerApi player, int score)
    {
        if (!Networking.IsOwner(gameObject))
        {
            Networking.SetOwner(Networking.LocalPlayer, gameObject);
        }

        int index = player.playerId % 20;
        playerNames[index] = player.displayName;
        playerScores[index] = score;

        RequestSerialization();
        UpdateDisplay();
    }

    public override void OnDeserialization()
    {
        UpdateDisplay();
    }

    private void UpdateDisplay()
    {
        string display = "=== SCOREBOARD ===\n";
        for (int i = 0; i < playerNames.Length; i++)
        {
            if (!string.IsNullOrEmpty(playerNames[i]))
            {
                display += $"{playerNames[i]}: {playerScores[i]}\n";
            }
        }
        scoreboardText.text = display;
    }
}
```

---

## 8. パフォーマンス最適化

### 8.1 Quest最適化ガイドライン

#### ポリゴン数
- **ワールド全体**: 250,000トライアングル以下推奨
- **1オブジェクト**: 10,000トライアングル以下

#### テクスチャ
- **最大解像度**: 1024x1024
- **圧縮形式**: ASTC（Quest標準）
- **ミップマップ**: 有効化

#### ライティング
- **ベイクドライティング**: 優先使用
- **リアルタイムライト**: 最小限（高コスト）
- **ライトマップ**: 適切に設定

#### PhysBones（Quest制限）
- **コンポーネント数**: 最大8
- **影響トランスフォーム**: 最大64
- **Collider**: 最大16

#### オーディオ
- **圧縮**: Vorbis推奨
- **LoadType**: Streaming（大ファイル）、Compressed In Memory（小ファイル）

### 8.2 UdonSharp最適化

#### GetComponent回避

```csharp
// ❌ 悪い例
void Update()
{
    var rb = GetComponent<Rigidbody>();
    rb.velocity = Vector3.forward;
}

// ✅ 良い例
private Rigidbody rb;

void Start()
{
    rb = GetComponent<Rigidbody>();
}

void Update()
{
    rb.velocity = Vector3.forward;
}
```

#### Update()最小化

```csharp
// ❌ 悪い例：毎フレーム実行
void Update()
{
    CheckPlayerStatus();
}

// ✅ 良い例：タイマーベース
private float checkInterval = 1.0f;
private float nextCheckTime = 0;

void Update()
{
    if (Time.time >= nextCheckTime)
    {
        CheckPlayerStatus();
        nextCheckTime = Time.time + checkInterval;
    }
}

// ✅ より良い例：イベント駆動
public void OnPlayerAction()
{
    CheckPlayerStatus();
}
```

#### ネットワーク同期最適化

```csharp
// Manual Syncでバッチング
[UdonBehaviourSyncMode(BehaviourSyncMode.Manual)]
public class OptimizedSync : UdonSharpBehaviour
{
    [UdonSynced] private int value1;
    [UdonSynced] private int value2;
    [UdonSynced] private int value3;

    private bool pendingSync = false;
    private float syncDelay = 0.5f;
    private float nextSyncTime = 0;

    public void UpdateValues(int v1, int v2, int v3)
    {
        value1 = v1;
        value2 = v2;
        value3 = v3;
        pendingSync = true;
    }

    void Update()
    {
        if (pendingSync && Time.time >= nextSyncTime)
        {
            RequestSerialization();
            pendingSync = false;
            nextSyncTime = Time.time + syncDelay;
        }
    }
}
```

### 8.3 ライティング最適化

#### ベイクドライティング設定

```
Window → Rendering → Lighting Settings
↓
Mixed Lighting → Baked Indirect
↓
Lightmapping Settings調整
  - Lightmap Resolution: 20-40（品質と容量のバランス）
  - Lightmap Size: 2048-4096
  - Compress Lightmaps: 有効
↓
Generate Lighting
```

#### リアルタイムライト削減

```
静的オブジェクト → Lightmap Static有効
↓
動的オブジェクトのみLight Probes使用
↓
リアルタイムライト → 最小限（2-3個以下）
```

---

## 9. トラブルシューティング

### 9.1 よくある問題と解決法

#### VRChat SDK関連

**問題**: SDK Control Panelが表示されない
```
解決法:
VRChat SDK → Show Control Panel
```

**問題**: ビルドエラー「Missing Scripts」
```
解決法:
1. Missing Scriptsを探す（Edit → Project Settings → Script Execution Order）
2. 該当オブジェクトからスクリプト削除
3. 再ビルド
```

#### UdonSharp関連

**問題**: UdonSharpコンパイルエラー
```
解決法:
1. UdonSharp → Compile All UdonSharp Programs
2. Console確認
3. エラー箇所修正
4. 再コンパイル
```

**問題**: UdonSynced変数が同期されない
```
解決法:
1. BehaviourSyncModeがManualの場合、RequestSerialization()呼び出し確認
2. オーナーシップ確認（Networking.IsOwner()）
3. 同期変数のデータサイズ確認（Continuous: 200バイト、Manual: 282KB制限）
```

#### ClientSim関連

**問題**: ClientSimで動作しない
```
解決法:
1. ClientSim Settings確認（VRChat SDK → ClientSim Settings）
2. Unity Console確認
3. Udonプログラム再コンパイル
4. Unity Editor再起動
```

### 9.2 非推奨ツールからの移行

#### CyanPlayerObjectPool → VRChat公式Player Objects

**移行手順**:
```
1. VRChat SDK 3.7.2以降にアップグレード
2. CyanPlayerObjectPool削除
3. VRChat公式Player Objectsドキュメント参照
   https://creators.vrchat.com/worlds/udon/player-objects/
4. 新システムで実装
```

#### CyanTrigger → Udon Graph / UdonSharp

**移行手順**:
```
1. CyanTriggerロジック確認
2. Udon Graphまたは UdonSharpで再実装
3. テスト
4. CyanTrigger削除
```

#### CyanEmu → ClientSim

**移行手順**:
```
1. VRChat SDK 3.x にアップグレード（ClientSim自動含まれる）
2. CyanEmu削除
3. ClientSim使用
```

---

## 10. 付録

### 10.1 用語集

| 用語 | 説明 |
|------|------|
| **Udon** | VRChat公式スクリプトシステム |
| **UdonSharp** | C#でUdon記述できるコンパイラ |
| **ClientSim** | Unity内VRChatエミュレータ |
| **VCC** | VRChat Creator Companion（SDK管理ツール） |
| **VPM** | VRChat Package Manager |
| **PlayerData** | プレイヤー永続化データ（100KB制限） |
| **UdonSynced** | ネットワーク同期変数 |
| **Manual Sync** | 手動同期モード（RequestSerialization必要） |
| **Continuous Sync** | 自動同期モード |
| **Master Client** | インスタンスのホストプレイヤー |
| **Late Joiner** | 後から参加したプレイヤー |

### 10.2 リファレンスリンク

#### 公式ドキュメント
- VRChat Creation: https://creators.vrchat.com/
- SDK Releases: https://creators.vrchat.com/releases/
- Udon: https://creators.vrchat.com/worlds/udon/
- UdonSharp: https://udonsharp.docs.vrchat.com/
- ClientSim: https://clientsim.docs.vrchat.com/
- VCC: https://vcc.docs.vrchat.com/

#### GitHub
- VRChat Packages: https://github.com/vrchat/packages
- UdonSharp: https://github.com/MerlinVR/UdonSharp
- ClientSim: https://github.com/vrchat-community/ClientSim
- VCC: https://github.com/vrchat-community/creator-companion

#### サードパーティ
- VRWorld Toolkit: https://github.com/oneVR/VRWorldToolkit
- AudioLink: https://github.com/llealloo/audiolink
- ProTV: https://gitlab.com/techanon/protv
- VideoTXL: https://github.com/vrctxl/VideoTXL
- VRCQuestTools: https://github.com/kurotu/VRCQuestTools
- Thry's Tools: https://github.com/Thryrallo/VRC-Avatar-Performance-Tools
- VUdon: https://github.com/Varneon/VUdon
- UdonUtils: https://github.com/Guribo/UdonUtils
- VRCX: https://github.com/vrcx-team/VRCX

#### コミュニティ
- VRChat Ask Forum: https://ask.vrchat.com/
- VRChat Wiki: https://wiki.vrchat.com/
- VPM Catalog: https://vpm-catalog.vercel.app/

### 10.3 バージョン履歴

| バージョン | 日付 | 変更内容 |
|----------|------|---------|
| 1.0 | 2025-11-17 | 初版作成 |

---

**文書管理情報**
- 作成者：VRChat World開発チーム
- 最終更新：2025-11-17
- ステータス：公開版
- 対象SDK：VRChat SDK 3.9.0
- 対象Unity：2022.3.22f1

---

**このドキュメントは「森のきこりキャンプ」VRChatワールド開発プロジェクトの一部として作成されました。**