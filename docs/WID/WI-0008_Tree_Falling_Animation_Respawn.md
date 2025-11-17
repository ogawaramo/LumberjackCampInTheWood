# 作業指示書:WI-0008 Tree Falling Animation and Respawn

**作成日**: 2025-11-17
**Phase**: Phase 1
**Priority**: 高
**Estimated Days**: 1日

---

## 1. 作業ID

**WI-0008**

---

## 2. 作業名

Tree Falling Animation and Respawn - 倒木アニメーションとリスポーン処理の実装

---

## 3. 作業目的

VRChatワールド「森のきこりキャンプ」における倒木時の視覚的演出とリスポーン処理を実装する。**UdonSharpの制約（Coroutine不可）を考慮し、SendCustomEventDelayedFrames/Secondsを活用**して、滑らかな倒木アニメーション、サウンドエフェクト、パーティクルエフェクト、およびリスポーンタイマーを実現する。Valheimライクな気持ちよい伐採体験を提供する。

---

## 4. 対象

### 対象システム
- VRChat World: 森のきこりキャンプ (Woodcutter Camp)

### 対象モジュール
- **Module**: M-10 TreeController
- **Functions**:
  - F-32: Start Falling Animation（倒木アニメーション開始）
  - F-33: Respawn Tree（木のリスポーン）

### 対象ファイルパス
- 更新対象: `Assets/WoodcutterCamp/Scripts/Gameplay/Woodcutting/TreeController.cs`
- 新規作成: `Assets/WoodcutterCamp/Prefabs/Effects/TreeFallingEffects.prefab`
- 新規作成: `Assets/WoodcutterCamp/Prefabs/Effects/TreeRespawnEffects.prefab`

---

## 5. 前提条件

### 開発環境
- Unity 2022.3.22f1 LTS
- VRChat SDK 3.9.0
- UdonSharp 1.1.9
- 作業ブランチ: `feature/WI-0008-tree-animation`（mainから分岐）

### 依存モジュール
- **WI-0007**: TreeController State Machine（木の状態管理が実装済み）
  - TreeController.csが存在し、基本的な状態遷移が動作している
  - StartFalling(), _OnFallingComplete(), RespawnTree()のメソッドスタブが存在

### 初期状態
- TreeController.csでHP=0時に`StartFalling()`が呼び出される
- 現在は簡易的な状態遷移のみで、アニメーションは未実装
- リスポーンタイマーの基本処理は実装済み

---

## 6. 入力

### メソッド呼び出し
```csharp
// TreeController.csから呼び出される
StartFalling(VRCPlayerApi attacker);
```
- `attacker` (VRCPlayerApi): 最後の一撃を与えたプレイヤー（倒木方向計算に使用）

### ScriptableObjectデータ
```json
{
  "treeType": "Pine | Oak | Ancient",
  "respawnTime": 180.0 | 300.0 | 600.0,  // 秒
  "fallingDuration": 2.0,                  // 倒木アニメーション時間（秒）
  "fallingRotationAngle": 90.0             // 倒木時の回転角度（度）
}
```

---

## 7. 出力

### アニメーション実行
- 2秒間かけて木が滑らかに倒れる（Quaternion.Lerpによる回転）
- 倒木方向: プレイヤー位置の反対方向（Valheim方式）
- フレームベース更新: SendCustomEventDelayedFrames使用（60fps = 120フレーム）

### サウンド再生
1. **ミシミシ音**: HP 30%以下で再生（倒木前の警告音）
2. **バキッ音**: 倒木アニメーション開始時
3. **ドサッ音**: 地面接触時（アニメーション完了時）

### パーティクルエフェクト
1. **木片パーティクル**: 倒木開始時（木の幹から放射状に10個）
2. **葉っぱパーティクル**: 倒木中（ランダムに散る、20個）
3. **土煙パーティクル**: 地面接触時（衝撃位置から上昇、30個）
4. **リスポーンパーティクル**: リスポーン時（緑色の光、根元から上昇、15個）

### リスポーンタイマー
- Pine: 180秒（3分）
- Oak: 300秒（5分）
- Ancient: 600秒（10分）
- タイマー完了時に`_RespawnTree()`を自動呼び出し

---

## 8. 作業手順

### 手順1: TreeController.csの倒木方向計算実装
**所要時間**: 30分

1. **CalculateFallingDirectionメソッドを実装**
   ```csharp
   /// <summary>
   /// 倒木方向を計算（Valheim方式）
   /// プレイヤー位置の反対方向に倒れる
   /// </summary>
   private Vector3 CalculateFallingDirection(VRCPlayerApi attacker)
   {
       if (attacker == null || !attacker.IsValid())
       {
           Debug.LogWarning("[TreeController] Invalid attacker. Using default direction (forward)");
           return transform.forward;
       }

       // プレイヤー位置から木の位置へのベクトル
       Vector3 toTree = transform.position - attacker.GetPosition();
       toTree.y = 0; // 水平方向のみ

       // その反対方向に倒れる
       Vector3 fallingDirection = toTree.normalized;

       // 異常値チェック
       if (fallingDirection.magnitude < 0.1f)
       {
           Debug.LogWarning("[TreeController] Calculated direction is near-zero. Using forward.");
           return transform.forward;
       }

       Debug.Log($"[TreeController] Tree {treeID} falling direction: {fallingDirection}");
       return fallingDirection;
   }
   ```

### 手順2: 倒木アニメーション処理の実装（SendCustomEventDelayedFrames使用）
**所要時間**: 2.5時間

1. **アニメーション用の変数追加**
   ```csharp
   // ========== Falling Animation Variables ==========
   [Header("Falling Animation Settings")]
   [SerializeField] private float fallingDuration = 2.0f;  // アニメーション時間（秒）
   [SerializeField] private int animationTotalFrames = 120; // 60fps × 2秒 = 120フレーム

   private bool isAnimating = false;
   private Vector3 fallingDirection;
   private Quaternion startRotation;
   private Quaternion endRotation;
   private int currentAnimationFrame = 0;
   ```

2. **StartFallingメソッドを更新**
   ```csharp
   private void StartFalling(VRCPlayerApi attacker)
   {
       Debug.Log($"[TreeController] Tree {treeID} HP reached 0. Starting falling animation.");

       // 状態遷移
       treeState = STATE_FALLING;

       // 倒木方向の計算
       fallingDirection = CalculateFallingDirection(attacker);

       // 回転の開始・終了状態を計算
       startRotation = transform.rotation;

       // 倒木方向を向いた状態で90度傾ける
       Quaternion lookRotation = Quaternion.LookRotation(fallingDirection, Vector3.up);
       endRotation = lookRotation * Quaternion.Euler(90f, 0f, 0f);

       // アニメーション開始
       isAnimating = true;
       currentAnimationFrame = 0;

       // サウンド再生（バキッ音）
       PlaySound(crackSound, 1.0f);

       // パーティクル再生（木片）
       if (woodChipsParticle != null)
       {
           woodChipsParticle.Play();
       }

       // アニメーション更新を開始（次のフレームで呼び出し）
       SendCustomEventDelayedFrames(nameof(_UpdateFallingAnimation), 1);

       // ネットワーク同期
       if (networkSync != null)
       {
           networkSync._RequestStatUpdate(this);
       }
       else
       {
           RequestSerialization();
       }
   }
   ```

3. **_UpdateFallingAnimationメソッドを実装**
   ```csharp
   /// <summary>
   /// 倒木アニメーションのフレーム更新処理
   /// UdonSharp制約によりCoroutine不可のため、SendCustomEventDelayedFramesで代替
   /// </summary>
   public void _UpdateFallingAnimation()
   {
       if (!isAnimating) return;

       // 進捗率を計算（0.0 → 1.0）
       float progress = (float)currentAnimationFrame / (float)animationTotalFrames;

       // イージング関数適用（Ease-In-Out Cubic）
       float easedProgress = EaseInOutCubic(progress);

       // 回転を適用（Quaternion.Lerp使用）
       transform.rotation = Quaternion.Lerp(startRotation, endRotation, easedProgress);

       // 葉っぱパーティクルを中間地点で再生
       if (currentAnimationFrame == animationTotalFrames / 2 && leavesParticle != null)
       {
           leavesParticle.Play();
       }

       // フレームカウント増加
       currentAnimationFrame++;

       // アニメーション継続判定
       if (currentAnimationFrame < animationTotalFrames)
       {
           // 次のフレームで再度呼び出し（約16ms後 = 60fps）
           SendCustomEventDelayedFrames(nameof(_UpdateFallingAnimation), 1);
       }
       else
       {
           // アニメーション完了
           _OnFallingComplete();
       }
   }

   /// <summary>
   /// イージング関数（Ease-In-Out Cubic）
   /// </summary>
   private float EaseInOutCubic(float t)
   {
       if (t < 0.5f)
       {
           return 4.0f * t * t * t;
       }
       else
       {
           float f = (2.0f * t - 2.0f);
           return 0.5f * f * f * f + 1.0f;
       }
   }
   ```

4. **_OnFallingCompleteメソッドを実装**
   ```csharp
   public void _OnFallingComplete()
   {
       if (!Networking.LocalPlayer.isMaster) return;

       Debug.Log($"[TreeController] Tree {treeID} falling animation complete. State → Fallen");

       isAnimating = false;

       // 状態をFallenに遷移
       treeState = STATE_FALLEN;

       // 最終回転を設定（誤差補正）
       transform.rotation = endRotation;

       // 地面接触音（ドサッ音）
       PlaySound(thudSound, 1.0f);

       // 土煙パーティクル
       if (dustParticle != null)
       {
           dustParticle.Play();
       }

       // リスポーンタイマー開始
       respawnTimer = respawnTime;
       SendCustomEventDelayedSeconds(nameof(_UpdateRespawnTimer), 1.0f);

       // ビジュアル状態更新
       UpdateVisualState();

       // LogSpawnerに倒木完了を通知
       NotifyLogSpawner();

       // ネットワーク同期
       if (networkSync != null)
       {
           networkSync._RequestStatUpdate(this);
       }
       else
       {
           RequestSerialization();
       }
   }
   ```

### 手順3: サウンド再生処理の実装
**所要時間**: 1時間

1. **AudioSource設定とサウンド再生メソッド**
   ```csharp
   // ========== Audio Settings ==========
   [Header("Audio Settings")]
   [SerializeField] private AudioSource audioSource;
   [SerializeField] private AudioClip creakingSound;       // ミシミシ音
   [SerializeField] private AudioClip crackSound;          // バキッ音
   [SerializeField] private AudioClip thudSound;           // ドサッ音
   [SerializeField] private AudioClip respawnSound;        // リスポーン音

   void Start()
   {
       // 既存の初期化処理...

       // AudioSourceの設定
       if (audioSource == null)
       {
           audioSource = GetComponent<AudioSource>();
           if (audioSource == null)
           {
               audioSource = gameObject.AddComponent<AudioSource>();
           }
       }

       audioSource.spatialBlend = 1.0f;        // 3D音源
       audioSource.rolloffMode = AudioRolloffMode.Linear;
       audioSource.minDistance = 5.0f;         // 5m以内は最大音量
       audioSource.maxDistance = 50.0f;        // 50mで聞こえなくなる
       audioSource.dopplerLevel = 0.0f;        // ドップラー効果なし

       Debug.Log("[TreeController] AudioSource configured");
   }

   /// <summary>
   /// サウンドを再生
   /// </summary>
   private void PlaySound(AudioClip clip, float volume)
   {
       if (audioSource == null || clip == null) return;

       audioSource.PlayOneShot(clip, volume);
       Debug.Log($"[TreeController] Playing sound: {clip.name}");
   }
   ```

2. **HP 30%以下でミシミシ音を再生**
   ```csharp
   public void _TakeDamage(int damage, VRCPlayerApi attacker)
   {
       // 既存のバリデーション処理...

       // ダメージ適用
       int previousHP = currentHP;
       currentHP = Mathf.Max(0, currentHP - damage);
       lastAttackerPlayerID = attacker.playerId;

       Debug.Log($"[TreeController] Tree {treeID} took {damage} damage. HP: {previousHP} → {currentHP}");

       // HP 30%以下でミシミシ音を再生
       float hpPercentage = (float)currentHP / (float)maxHP;
       if (hpPercentage <= 0.3f && hpPercentage > 0)
       {
           PlaySound(creakingSound, 0.7f);
       }

       // HP=0判定
       if (currentHP <= 0)
       {
           StartFalling(attacker);
       }
       else
       {
           // ダメージエフェクト表示（ローカル処理）
           ShowDamageEffect(damage);
       }

       // ネットワーク同期
       if (networkSync != null)
       {
           networkSync._RequestStatUpdate(this);
       }
   }
   ```

### 手順4: パーティクルシステムの設定
**所要時間**: 2時間

1. **ParticleSystem参照の追加**
   ```csharp
   // ========== Particle Effects ==========
   [Header("Particle Effects")]
   [SerializeField] private ParticleSystem woodChipsParticle;   // 木片
   [SerializeField] private ParticleSystem leavesParticle;      // 葉っぱ
   [SerializeField] private ParticleSystem dustParticle;        // 土煙
   [SerializeField] private ParticleSystem respawnParticle;     // リスポーン
   ```

2. **Unity Editorでパーティクル作成**
   - パス: `Assets/WoodcutterCamp/Prefabs/Effects/TreeFallingEffects.prefab`

   **木片パーティクル（WoodChipsParticle）**:
   ```
   Main Module:
   - Duration: 0.5秒
   - Looping: OFF
   - Start Lifetime: 0.5〜1.0秒（ランダム）
   - Start Speed: 2〜5 m/s（ランダム）
   - Start Size: 0.05〜0.15m（ランダム）
   - Start Color: 茶色（#8B4513）
   - Max Particles: 10

   Emission Module:
   - Rate over Time: 0
   - Bursts: Time=0, Count=10（一度に10個放出）

   Shape Module:
   - Shape: Sphere
   - Radius: 0.5m
   - Emit from: Volume

   Velocity over Lifetime:
   - Linear: (0, -2, 0) （重力効果）

   Renderer Module:
   - Render Mode: Billboard
   - Material: Default-Particle
   ```

   **葉っぱパーティクル（LeavesParticle）**:
   ```
   Main Module:
   - Duration: 1.5秒
   - Start Lifetime: 1.0〜2.0秒
   - Start Speed: 1〜3 m/s
   - Start Size: 0.1〜0.2m
   - Start Color: 緑色（#228B22）〜黄色（#FFD700）のグラデーション
   - Max Particles: 20

   Emission Module:
   - Bursts: Time=0, Count=20

   Shape Module:
   - Shape: Cone
   - Angle: 30度
   - Radius: 1.0m

   Rotation over Lifetime:
   - Angular Velocity: -180〜180度/秒（ランダム回転）
   ```

   **土煙パーティクル（DustParticle）**:
   ```
   Main Module:
   - Duration: 1.0秒
   - Start Lifetime: 0.8〜1.2秒
   - Start Speed: 0.5〜2 m/s
   - Start Size: 0.3〜0.8m
   - Start Color: 茶色半透明（#8B4513, Alpha=0.5）
   - Max Particles: 30

   Emission Module:
   - Bursts: Time=0, Count=30

   Shape Module:
   - Shape: Circle
   - Radius: 1.0m

   Size over Lifetime:
   - Size Curve: 0.0 → 1.0 → 0.5（膨張→縮小）
   ```

   **リスポーンパーティクル（RespawnParticle）**:
   ```
   Main Module:
   - Duration: 2.0秒
   - Start Lifetime: 1.5〜2.0秒
   - Start Speed: 0.5〜1.5 m/s
   - Start Size: 0.1〜0.3m
   - Start Color: 緑色発光（#00FF00, Emission=2.0）
   - Max Particles: 15

   Emission Module:
   - Rate over Time: 10個/秒

   Shape Module:
   - Shape: Cone
   - Angle: 10度（ほぼ垂直上昇）
   ```

3. **Quest最適化設定**
   - すべてのパーティクルでMax Particles制限を厳守
   - 合計パーティクル数: 10 + 20 + 30 + 15 = 75個
   - Quest向けビルド時は各パーティクル数を30%削減:
     - 木片: 7個
     - 葉っぱ: 14個
     - 土煙: 21個
     - リスポーン: 10個
     - 合計: 52個（許容範囲内）

### 手順5: リスポーン処理の実装
**所要時間**: 1.5時間

1. **リスポーンタイマー更新処理**
   ```csharp
   /// <summary>
   /// リスポーンタイマーの更新処理（1秒ごと）
   /// SendCustomEventDelayedSecondsで再帰呼び出し
   /// </summary>
   public void _UpdateRespawnTimer()
   {
       // Master Clientのみ実行
       if (!Networking.LocalPlayer.isMaster) return;

       // Fallen状態かつタイマーが残っている場合のみ更新
       if (treeState == STATE_FALLEN && respawnTimer > 0)
       {
           respawnTimer -= 1.0f;

           Debug.Log($"[TreeController] Tree {treeID} respawn timer: {respawnTimer}s remaining");

           if (respawnTimer <= 0)
           {
               // リスポーン実行
               _RespawnTree();
           }
           else
           {
               // 1秒後に再度更新
               SendCustomEventDelayedSeconds(nameof(_UpdateRespawnTimer), 1.0f);
           }
       }
   }
   ```

2. **_RespawnTreeメソッドを実装**
   ```csharp
   /// <summary>
   /// 木のリスポーン処理
   /// </summary>
   public void _RespawnTree()
   {
       // Master Clientのみ実行
       if (!Networking.LocalPlayer.isMaster) return;

       Debug.Log($"[TreeController] Tree {treeID} respawning...");

       // 状態とHPをリセット
       currentHP = maxHP;
       treeState = STATE_STANDING;
       respawnTimer = 0f;
       lastAttackerPlayerID = -1;
       isAnimating = false;
       currentAnimationFrame = 0;

       // 回転をリセット（直立状態）
       transform.rotation = Quaternion.identity;

       // ビジュアル状態を更新
       UpdateVisualState();

       // リスポーンエフェクト再生
       PlayRespawnEffect();

       // ネットワーク同期
       if (networkSync != null)
       {
           networkSync._RequestStatUpdate(this);
       }
       else
       {
           RequestSerialization();
       }

       Debug.Log($"[TreeController] Tree {treeID} respawned with HP={currentHP}");
   }

   /// <summary>
   /// リスポーンエフェクトを再生
   /// </summary>
   private void PlayRespawnEffect()
   {
       // リスポーンパーティクル再生
       if (respawnParticle != null)
       {
           respawnParticle.Play();
           Debug.Log("[TreeController] Respawn particle effect played");
       }

       // リスポーン音再生
       PlaySound(respawnSound, 0.8f);
   }
   ```

### 手順6: AudioClipの設定
**所要時間**: 1時間

1. **サウンドファイルの準備**
   - パス: `Assets/WoodcutterCamp/Audio/Tree/`

   **必要なサウンド**:
   | サウンド名 | ファイル | 長さ | 音量 | 備考 |
   |----------|---------|------|------|------|
   | ミシミシ音 | tree_creaking.wav | 2秒 | 0.7 | HP 30%以下で再生 |
   | バキッ音 | tree_crack.wav | 0.5秒 | 1.0 | 倒木開始時 |
   | ドサッ音 | tree_thud.wav | 1秒 | 1.0 | 地面接触時 |
   | リスポーン音 | tree_respawn.wav | 2秒 | 0.8 | リスポーン時 |

2. **AudioClipのインポート設定**
   ```
   Load Type: Decompress On Load（小さいファイル用）
   Preload Audio Data: ON
   Compression Format:
   - PC/Android: Vorbis（高品質）
   - Quest: ADPCM（低負荷）
   Quality: 70%
   Sample Rate Setting: Preserve Sample Rate
   ```

### 手順7: Prefab設定とインテグレーション
**所要時間**: 1.5時間

1. **TreeFallingEffects.prefabの作成**
   - 空のGameObjectを作成
   - 以下のコンポーネントを追加:
     - ParticleSystem（木片）
     - ParticleSystem（葉っぱ）
     - ParticleSystem（土煙）

2. **TreeRespawnEffects.prefabの作成**
   - 空のGameObjectを作成
   - ParticleSystem（リスポーン用）を追加

3. **既存TreePrefabへの統合**
   - PineTree.prefab, OakTree.prefab, AncientTree.prefabを開く
   - TreeFallingEffects.prefabを子オブジェクトとして追加
   - TreeRespawnEffects.prefabを子オブジェクトとして追加
   - TreeControllerのInspectorで各ParticleSystem参照を設定
   - TreeControllerのInspectorで各AudioClipを設定

4. **AudioSourceの追加**
   - 各TreePrefabにAudioSourceコンポーネントを追加（未追加の場合）
   - TreeControllerのInspectorで`audioSource`参照を設定

### 手順8: VR酔い防止対策の実装
**所要時間**: 30分

1. **カメラシェイクの制限（Phase 1では未実装）**
   ```csharp
   /// <summary>
   /// カメラシェイク（Phase 1では未実装）
   /// VR酔い防止のため、実装はPhase 2以降
   /// </summary>
   private void TriggerCameraShake()
   {
       // Phase 1では実装しない（VR酔い防止のため）
       // Phase 2で実装予定
       Debug.Log("[TreeController] Camera shake skipped (Phase 1)");
   }
   ```

2. **アニメーション速度の調整**
   - 倒木時間を2秒確保（最低1秒以上）
   - 急激な動きを避けるためEase-In-Out Cubicを使用
   - VRユーザーには特に配慮（カメラシェイクなし）

### 手順9: テストケースの作成
**所要時間**: 1時間

1. **手動テスト手順書の作成**
   - パス: `docs/Testing/WI-0008_TestProcedure.md`

   **テストケースリスト**:
   - TC-001: 倒木アニメーション（基本）
   - TC-002: 倒木方向の確認
   - TC-003: ミシミシ音の再生
   - TC-004: リスポーンタイマー（Pine）
   - TC-005: リスポーンタイマー（Oak）
   - TC-006: リスポーンタイマー（Ancient）
   - TC-007: Quest環境でのパフォーマンス
   - TC-008: Late Joiner対応

---

## 9. 完了条件（Doneの条件）

### 機能要件
- [ ] 倒木アニメーションが滑らかに動作する
  - 2秒間でQuaternion.Lerpによる回転
  - Ease-In-Out Cubicのイージング適用
  - プレイヤーと反対方向に倒れる
  - SendCustomEventDelayedFramesを使用（60fps = 120フレーム）
- [ ] サウンドが適切なタイミングで再生される
  - ミシミシ音: HP 30%以下
  - バキッ音: 倒木開始時
  - ドサッ音: 地面接触時
  - リスポーン音: リスポーン時
- [ ] パーティクルエフェクトが正常に表示される
  - 木片パーティクル: 倒木開始時
  - 葉っぱパーティクル: 倒木中
  - 土煙パーティクル: 地面接触時
  - リスポーンパーティクル: リスポーン時
- [ ] リスポーンタイマーが正確に動作する
  - Pine: 180秒、Oak: 300秒、Ancient: 600秒
  - SendCustomEventDelayedSecondsを使用
- [ ] Quest最適化が完了している
  - パーティクル総数52個以下（Quest版）
  - AudioClipはADPCM圧縮
  - 60fps維持

### テスト要件
- [ ] ClientSimで全テストケース（TC-001〜TC-008）が合格
- [ ] Questビルドで60fps維持を確認

### ドキュメント要件
- [ ] TreeController.csにXMLコメント記載済み
- [ ] 手動テスト手順書（WI-0008_TestProcedure.md）作成済み
- [ ] README.mdに倒木アニメーションの説明を追加

---

## 10. テスト観点・テストケース

### テスト種別
- 機能テスト（ClientSim使用）
- パフォーマンステスト（Quest Build）
- ネットワーク同期テスト（Late Joiner対応）

### 正常系テストケース

| No. | テスト項目 | 入力 | 期待結果 |
|-----|----------|------|---------|
| TC-001 | 倒木アニメーション基本動作 | HP=0、attacker有効 | 2秒でプレイヤーと反対方向に倒れる |
| TC-002 | 倒木方向計算（北側攻撃） | attacker位置=木の北側 | 木が南側に倒れる |
| TC-003 | 倒木方向計算（無効なattacker） | attacker=null | transform.forward方向に倒れる |
| TC-004 | バキッ音再生 | 倒木開始時 | crackSoundが再生される |
| TC-005 | ドサッ音再生 | アニメーション完了時 | thudSoundが再生される |
| TC-006 | ミシミシ音再生 | HP 30% → 20% | creakingSoundが再生される |
| TC-007 | 木片パーティクル | 倒木開始時 | 10個の木片が放出される |
| TC-008 | 葉っぱパーティクル | アニメーション中間 | 20個の葉が散る |
| TC-009 | 土煙パーティクル | 地面接触時 | 30個の土煙が発生 |
| TC-010 | リスポーン（Pine） | 180秒経過 | HP=30、state=Standing |
| TC-011 | リスポーン（Oak） | 300秒経過 | HP=60、state=Standing |
| TC-012 | リスポーン（Ancient） | 600秒経過 | HP=100、state=Standing |
| TC-013 | リスポーンパーティクル | リスポーン時 | 緑色のパーティクル15個 |

### 異常系テストケース

| No. | テスト項目 | 入力 | 期待結果 |
|-----|----------|------|---------|
| TC-101 | アニメーション中の追加攻撃 | state=Falling、damage=10 | ダメージ無視、Warning出力 |
| TC-102 | AudioClip未設定 | crackSound=null | サウンドスキップ、アニメーション正常 |
| TC-103 | ParticleSystem未設定 | woodChipsParticle=null | パーティクルスキップ、アニメーション正常 |
| TC-104 | リスポーンタイマー中のMaster移行 | Master Client退出 | 新Masterがタイマー継続 |

### パフォーマンステスト

| No. | テスト項目 | 条件 | 期待結果 |
|-----|----------|------|---------|
| TC-201 | Quest環境での5本同時倒木 | Quest 2、5本同時 | 60fps維持 |
| TC-202 | パーティクル数制限 | 各木のパーティクル合計 | 52個以下（Quest版） |
| TC-203 | 30本全倒木時のパフォーマンス | 30本を10秒以内に全倒し | FPS低下<10% |

---

## 11. 成果物

### 新規作成ファイル
- `Assets/WoodcutterCamp/Prefabs/Effects/TreeFallingEffects.prefab`
- `Assets/WoodcutterCamp/Prefabs/Effects/TreeRespawnEffects.prefab`
- `Assets/WoodcutterCamp/Audio/Tree/tree_creaking.wav`
- `Assets/WoodcutterCamp/Audio/Tree/tree_crack.wav`
- `Assets/WoodcutterCamp/Audio/Tree/tree_thud.wav`
- `Assets/WoodcutterCamp/Audio/Tree/tree_respawn.wav`

### 更新ファイル
- `Assets/WoodcutterCamp/Scripts/Gameplay/Woodcutting/TreeController.cs`
  - CalculateFallingDirection()追加
  - StartFalling()更新
  - _UpdateFallingAnimation()追加（SendCustomEventDelayedFrames使用）
  - _OnFallingComplete()実装
  - _UpdateRespawnTimer()実装（SendCustomEventDelayedSeconds使用）
  - _RespawnTree()実装
  - PlaySound()追加
  - PlayRespawnEffect()追加
  - EaseInOutCubic()追加
- `Assets/WoodcutterCamp/Prefabs/Trees/PineTree.prefab`
- `Assets/WoodcutterCamp/Prefabs/Trees/OakTree.prefab`
- `Assets/WoodcutterCamp/Prefabs/Trees/AncientTree.prefab`
  - TreeFallingEffects.prefab統合
  - TreeRespawnEffects.prefab統合
  - AudioSource追加（未追加の場合）

### テストドキュメント
- `docs/Testing/WI-0008_TestProcedure.md`（手動テスト手順書）

### 更新ドキュメント
- `README.md`（倒木アニメーションの説明追加）
- `docs/TDD.md`（F-32, F-33実装完了記録）

---

## 12. 備考

### 注意点

1. **UdonSharpの制約対応**
   - **Coroutineは使用不可** → `SendCustomEventDelayedFrames`で代替
   - アニメーション更新は1フレームごとに`_UpdateFallingAnimation()`を呼び出し
   - 60fps想定で120フレーム = 2秒のアニメーション
   - リスポーンタイマーは`SendCustomEventDelayedSeconds`で1秒ごと更新

2. **Quaternion.Lerpの使用**
   - Unityの標準的な回転補間機能を使用
   - VRChatのUdonでも正常動作
   - イージング関数でより自然な動きを実現

3. **パーティクル数の最適化**
   - PC版: 合計75個（木片10 + 葉20 + 土煙30 + リスポーン15）
   - Quest版: 合計52個（各30%削減）
   - Quest Buildでは自動的にパーティクル数を削減

4. **VR酔い防止**
   - カメラシェイクはPhase 1では未実装
   - 倒木アニメーションは2秒以上確保（急激な動きを避ける）
   - イージング関数で滑らかな動きを実現

5. **Master Client検証**
   - リスポーン処理はMaster Clientのみ実行
   - タイマー更新もMaster Clientのみ
   - Late Joiner入場時はOnDeserializationで状態復元

### 制約事項

1. **Phase 1スコープ**
   - カメラシェイクは未実装（VR酔い防止のため）
   - 完全な物理シミュレーションなし（簡易的な回転のみ）
   - 地形との干渉判定なし

2. **サウンドファイル**
   - 適切なフリー音源または自作音源を使用
   - 商用利用可能なライセンス確認必須
   - 推奨サイト: Freesound.org、OpenGameArt.org

3. **パフォーマンス制約**
   - Quest 2環境で60fps維持必須
   - パーティクル総数52個以下（Quest版）
   - AudioClipはADPCM圧縮（Quest版）

### 依存関係

**このWIの完了後に開始できるWI**:
- WI-0009: Implement AxeInteraction Detection（斧との相互作用）
- WI-0012: Implement LogSpawner（倒木後の丸太生成）

**このWIが依存するWI**:
- WI-0007: TreeController State Machine（必須・完了済み）

### 参考資料

- TDD.md - Module M-10 TreeController（設計仕様）
- FSD.md - FNC-001 Section 3.1.4-5（倒木・リスポーン仕様）
- PRDv3.md - Section 2.1.3（倒木演出の詳細）
- Unity Documentation - ParticleSystem: https://docs.unity3d.com/2022.3/Documentation/Manual/ParticleSystems.html
- Unity Documentation - Quaternion.Lerp: https://docs.unity3d.com/2022.3/Documentation/ScriptReference/Quaternion.Lerp.html
- UdonSharp - SendCustomEventDelayedFrames: https://udonsharp.docs.vrchat.com/events/#sendcustomeventdelayedframes
- UdonSharp - SendCustomEventDelayedSeconds: https://udonsharp.docs.vrchat.com/events/#sendcustomeventdelayedseconds

---

**作成者**: VRChat World Development Team
**レビュアー**: （実装完了後にコードレビュー実施）
**承認者**: （テスト完了後に承認）
