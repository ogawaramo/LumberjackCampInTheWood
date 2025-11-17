# 作業指示書（Work Instruction）

## 1. 作業ID
**WI-0011**

## 2. 作業名
**Implement Hit Effects (Particles and Sound) - ヒットエフェクトの実装（パーティクル・サウンド・ダメージ表示）**

## 3. 作業目的
木を斧で叩いた際の視覚・聴覚フィードバックを実装し、プレイヤーが「気持ちよく」伐採できる体験を提供する。通常ヒット・クリティカルヒット・協力伐採の3種類のエフェクトを実装し、Quest環境での最適化も同時に行う。

## 4. 対象

* **対象システム**: 森のきこりキャンプ (Woodcutter Camp) - VRChatワールド
* **対象モジュール**: M-11 AxeInteraction（斧インタラクション）
* **対象ファイルパス**: `Assets/WoodcutterCamp/Scripts/Gameplay/Woodcutting/AxeInteraction.cs`
* **関連ファイル**:
  * ParticleSystemプレハブ: `Assets/WoodcutterCamp/Prefabs/Effects/`
  * AudioClip: `Assets/WoodcutterCamp/Audio/SFX/`
  * ダメージテキストUI: `Assets/WoodcutterCamp/Prefabs/UI/DamageText.prefab`

## 5. 前提条件

* **環境**:
  * Unity 2022.3.22f1が起動可能な状態であること
  * VRChat SDK 3.9.0とUdonSharp 1.1.9がインストール済みであること
  * プロジェクト `WoodcutterCamp` がUnityで開かれていること

* **ブランチ**: `develop` ブランチから新ブランチ `feature/WI-0011-hit-effects` を作成して作業すること

* **依存作業の完了状況**:
  * **WI-0010 (Implement Damage Calculation)** が完了していること
    - `AxeInteraction.cs` の `CalculateDamage()` メソッドが実装済み
    - 通常ダメージ・クリティカル・協力ボーナスの判定ロジックが動作していること
    - `TreeController.cs` の `TakeDamage(int damage, VRCPlayerApi attacker)` メソッドが実装済み

* **初期状態**:
  * `AxeInteraction.cs` にダメージ計算ロジックが実装されているが、エフェクト再生処理は未実装
  * Unityシーン `WoodcutterCamp.unity` が存在し、木オブジェクトと斧オブジェクトが配置されている

## 6. 入力

* **ダメージ計算の結果**:
  * `int calculatedDamage`: 計算されたダメージ量（10〜200の範囲）
  * `bool isCritical`: クリティカルヒットかどうか
  * `bool isCooperative`: 協力伐採ボーナスが適用されているかどうか

* **ヒット位置情報**:
  * `Vector3 hitPosition`: 斧が木に衝突した位置（ワールド座標）
  * `Vector3 hitNormal`: 衝突地点の法線ベクトル（パーティクルの方向決定に使用）

* **音源設定**:
  * 通常ヒット音: `hit_normal.wav`（70%音量、20m減衰）
  * クリティカルヒット音: `hit_critical.wav`（80%音量、20m減衰）
  * 協力伐採ヒット音: `hit_cooperative.wav`（70%音量、20m減衰）

## 7. 出力

* **視覚エフェクト**:
  * 通常ヒット: 木片パーティクル（茶色）5個が飛散
  * クリティカルヒット: 赤い衝撃波パーティクル10個が放射状に広がる
  * 協力伐採: 金色のキラキラパーティクル7個が上昇

* **サウンド再生**:
  * ヒット時に対応する効果音が3D空間オーディオで再生される
  * 同時に複数のヒット音が再生可能（AudioSource.PlayOneShot使用）

* **ダメージテキスト表示**:
  * 木の上部（+1.5m）にダメージ数値が表示される
  * 通常: 白色テキスト、サイズ1.0倍
  * クリティカル: 赤色テキスト、サイズ1.5倍
  * 1秒間表示後、フェードアウト（0.5秒）

## 8. 作業手順

### 8.1. 準備作業

#### 8.1.1. ブランチ作成
1. Gitクライアント（Git Bash / SourceTree / VSCodeなど）を起動
2. 以下のコマンドを実行してブランチを作成・切り替え:
   ```bash
   cd /mnt/d/Dropbox/Dev/CLI/WCF
   git checkout develop
   git pull origin develop
   git checkout -b feature/WI-0011-hit-effects
   ```

#### 8.1.2. 前提条件の確認
1. Unityエディタでプロジェクトを開く
2. Hierarchyウィンドウで `WoodcutterCamp` シーンが開かれていることを確認
3. Project ウィンドウで `Assets/WoodcutterCamp/Scripts/Gameplay/Woodcutting/AxeInteraction.cs` を開く
4. 以下のメソッドが存在することを確認:
   ```csharp
   private int CalculateDamage(bool isCritical, bool isCooperative)
   ```
5. TreeController.cs に以下のメソッドが存在することを確認:
   ```csharp
   public void TakeDamage(int damage, VRCPlayerApi attacker)
   ```

---

### 8.2. ParticleSystem の作成（通常ヒット）

#### 8.2.1. 通常ヒットパーティクルの設定
1. Unityエディタで `GameObject > Effects > Particle System` を選択
2. 作成されたオブジェクトの名前を `HitParticle_Normal` に変更
3. Inspectorで以下のように設定:

   **Main Module（メインモジュール）**:
   - Duration: `0.3`秒
   - Looping: `false`（チェックを外す）
   - Start Lifetime: `0.5`秒
   - Start Speed: `3`〜`5`（ランダム）
   - Start Size: `0.05`〜`0.1`（ランダム）
   - Start Color: 茶色 `RGB(139, 90, 43)` ※木片の色
   - Gravity Modifier: `1.0`（重力適用）
   - Max Particles: `5`

   **Emission Module（放出モジュール）**:
   - Rate over Time: `0`（無効化）
   - Bursts（バースト）:
     - Time: `0`秒
     - Count: `5`個
     - Cycles: `1`
     - Interval: `0.01`秒

   **Shape Module（形状モジュール）**:
   - Shape: `Cone`（円錐）
   - Angle: `30`度
   - Radius: `0.1`
   - Emit from: `Base`
   - Randomize Direction: `0.3`（少しランダム化）

   **Renderer Module（レンダラーモジュール）**:
   - Render Mode: `Billboard`（カメラに常に向く）
   - Material: デフォルトのParticle Material使用
   - Cast Shadows: `Off`（Quest最適化）
   - Receive Shadows: `Off`（Quest最適化）

4. Projectウィンドウで `Assets/WoodcutterCamp/Prefabs/Effects/` フォルダに移動
5. HierarchyからプレハブフォルダにHitParticle_Normalをドラッグ&ドロップしてPrefab化
6. Hierarchyから元のオブジェクトを削除

---

### 8.3. ParticleSystem の作成（クリティカルヒット）

#### 8.3.1. クリティカルヒットパーティクルの設定
1. Unityエディタで `GameObject > Effects > Particle System` を選択
2. 作成されたオブジェクトの名前を `HitParticle_Critical` に変更
3. Inspectorで以下のように設定:

   **Main Module**:
   - Duration: `0.5`秒
   - Looping: `false`
   - Start Lifetime: `0.7`秒
   - Start Speed: `5`〜`8`（クリティカルなので速い）
   - Start Size: `0.1`〜`0.2`（大きめ）
   - Start Color: 赤色 `RGB(255, 50, 50)` ※衝撃波の色
   - Gravity Modifier: `0`（重力なし）
   - Max Particles: `10`

   **Emission Module**:
   - Rate over Time: `0`
   - Bursts:
     - Time: `0`秒
     - Count: `10`個
     - Cycles: `1`

   **Shape Module**:
   - Shape: `Sphere`（球体）
   - Radius: `0.2`
   - Emit from: `Volume`
   - Randomize Direction: `0.5`（大きくランダム化）

   **Color over Lifetime Module（有効化）**:
   - Gradient（グラデーション）設定:
     - 0%: 赤色 Alpha 100%
     - 100%: 赤色 Alpha 0%（フェードアウト）

   **Size over Lifetime Module（有効化）**:
   - Curve（カーブ）設定:
     - 0%: Size 1.0
     - 100%: Size 0.2（徐々に小さくなる）

   **Renderer Module**:
   - Render Mode: `Billboard`
   - Material: デフォルトのParticle Material
   - Cast/Receive Shadows: `Off`

4. `Assets/WoodcutterCamp/Prefabs/Effects/` にPrefab化
5. Hierarchyから元のオブジェクトを削除

---

### 8.4. ParticleSystem の作成（協力伐採）

#### 8.4.1. 協力伐採パーティクルの設定
1. Unityエディタで `GameObject > Effects > Particle System` を選択
2. 作成されたオブジェクトの名前を `HitParticle_Cooperative` に変更
3. Inspectorで以下のように設定:

   **Main Module**:
   - Duration: `0.6`秒
   - Looping: `false`
   - Start Lifetime: `1.0`秒
   - Start Speed: `2`〜`4`（上昇する）
   - Start Size: `0.08`〜`0.15`（キラキラ）
   - Start Color: 金色 `RGB(255, 215, 0)` ※ゴールド
   - Gravity Modifier: `-0.5`（上昇効果）
   - Max Particles: `7`

   **Emission Module**:
   - Rate over Time: `0`
   - Bursts:
     - Time: `0`秒
     - Count: `7`個
     - Cycles: `1`

   **Shape Module**:
   - Shape: `Cone`
   - Angle: `15`度（上向き）
   - Radius: `0.15`
   - Emit from: `Base`

   **Color over Lifetime Module（有効化）**:
   - Gradient:
     - 0%: 金色 Alpha 100%
     - 80%: 金色 Alpha 100%
     - 100%: 金色 Alpha 0%（最後にフェード）

   **Rotation over Lifetime Module（有効化）**:
   - Angular Velocity: `45`〜`90`度/秒（キラキラ回転）

   **Renderer Module**:
   - Render Mode: `Billboard`
   - Material: デフォルトのParticle Material
   - Cast/Receive Shadows: `Off`

4. `Assets/WoodcutterCamp/Prefabs/Effects/` にPrefab化
5. Hierarchyから元のオブジェクトを削除

---

### 8.5. AudioClip のインポート設定

#### 8.5.1. サウンドファイルの配置
1. Windowsエクスプローラーで `Assets/WoodcutterCamp/Audio/SFX/` フォルダを開く
2. 以下の音声ファイルを配置（プロジェクトで用意されている音源を使用）:
   - `hit_normal.wav` - 通常ヒット音（「コン」という木を叩く音）
   - `hit_critical.wav` - クリティカルヒット音（「キン！」という金属的な音）
   - `hit_cooperative.wav` - 協力伐採音（「キラーン」という明るい音）

#### 8.5.2. AudioClipのインポート設定
1. Unityエディタに戻り、Projectウィンドウで各AudioClipを選択
2. Inspectorで以下のように設定（各ファイル共通）:

   **Import Settings（インポート設定）**:
   - Force To Mono: `false`（ステレオ維持）
   - Load In Background: `true`
   - Ambisonic: `false`
   - **Load Type**: `Decompress On Load`（Quest最適化: メモリ使用量削減）
   - **Compression Format**:
     - PC: `Vorbis`（Quality 70%）
     - Android (Quest): `ADPCM`（低遅延・低CPU負荷）
   - **Sample Rate Setting**: `Preserve Sample Rate`
   - Preload Audio Data: `true`（再生開始を高速化）

3. 各ファイルごとに「Apply」ボタンをクリックして設定を保存

---

### 8.6. DamageText UI Prefab の作成

#### 8.6.1. WorldSpace Canvas の作成
1. Hierarchyウィンドウで右クリック > `UI > Canvas` を選択
2. 作成されたCanvasの名前を `DamageTextCanvas` に変更
3. Inspectorで以下のように設定:

   **Canvas コンポーネント**:
   - Render Mode: `World Space`（3D空間に配置）
   - Event Camera: （設定不要、VRChatが自動設定）
   - Sorting Layer: `Default`
   - Order in Layer: `10`（他のUIより前面）

   **Rect Transform**:
   - Width: `200`
   - Height: `100`
   - Scale: `0.01, 0.01, 0.01`（ワールドスケールで小さく表示）

#### 8.6.2. Text (TextMeshPro) の追加
1. DamageTextCanvas を右クリック > `UI > Text - TextMeshPro` を選択
   - 初回の場合、TMP Import Windowが表示されるので「Import TMP Essentials」をクリック
2. 作成されたTextの名前を `DamageText` に変更
3. Inspectorで以下のように設定:

   **Rect Transform**:
   - Anchors: `Center`（中央配置）
   - Pos X, Y, Z: `0, 0, 0`
   - Width: `200`
   - Height: `100`

   **TextMeshPro - Text (UI) コンポーネント**:
   - Text: `-12`（ダメージ数値のプレースホルダー）
   - Font: `LiberationSans SDF`（デフォルトフォント）
   - Font Style: `Bold`（太字）
   - Font Size: `72`
   - Alignment: `Center` / `Middle`（中央揃え）
   - Vertex Color: 白色 `RGB(255, 255, 255)`
   - Enable Auto Sizing: `false`
   - Extra Settings > Outline:
     - Enable: `true`
     - Thickness: `0.2`
     - Color: 黒色 `RGB(0, 0, 0)` Alpha 100%（視認性向上）

#### 8.6.3. Canvas Group の追加（フェードアウト用）
1. DamageTextCanvas を選択
2. Inspector下部の「Add Component」をクリック
3. `Canvas Group` を検索して追加
4. 以下のように設定:
   - Alpha: `1.0`（初期状態は不透明）
   - Interactable: `false`
   - Block Raycasts: `false`（クリック無効）

#### 8.6.4. Prefab化
1. Projectウィンドウで `Assets/WoodcutterCamp/Prefabs/UI/` フォルダに移動
2. HierarchyからDamageTextCanvasをドラッグ&ドロップしてPrefab化
3. Hierarchyから元のオブジェクトを削除

---

### 8.7. AxeInteraction.cs の実装（エフェクト再生処理）

#### 8.7.1. フィールド変数の追加
1. VSCode または Visual Studioで `AxeInteraction.cs` を開く
2. クラスの先頭に以下のフィールド変数を追加:

```csharp
[Header("Hit Effects")]
[SerializeField] private GameObject hitParticleNormalPrefab;
[SerializeField] private GameObject hitParticleCriticalPrefab;
[SerializeField] private GameObject hitParticleCooperativePrefab;
[SerializeField] private GameObject damageTextPrefab;

[Header("Hit Sounds")]
[SerializeField] private AudioClip hitSoundNormal;
[SerializeField] private AudioClip hitSoundCritical;
[SerializeField] private AudioClip hitSoundCooperative;
[SerializeField] private AudioSource audioSource;

[Header("Effect Settings")]
[SerializeField] private float damageTextDuration = 1.0f;
[SerializeField] private float damageTextFadeTime = 0.5f;
[SerializeField] private float damageTextYOffset = 1.5f;
```

#### 8.7.2. PlayHitEffect メソッドの実装
3. 既存の `OnCollisionEnter()` メソッドの後に、以下のメソッドを追加:

```csharp
/// <summary>
/// ヒットエフェクトを再生する
/// </summary>
/// <param name="hitPosition">ヒット位置（ワールド座標）</param>
/// <param name="hitNormal">衝突法線ベクトル</param>
/// <param name="damage">ダメージ量</param>
/// <param name="isCritical">クリティカルヒットか</param>
/// <param name="isCooperative">協力伐採か</param>
private void PlayHitEffect(Vector3 hitPosition, Vector3 hitNormal, int damage, bool isCritical, bool isCooperative)
{
    // 1. パーティクル再生
    PlayHitParticle(hitPosition, hitNormal, isCritical, isCooperative);

    // 2. サウンド再生
    PlayHitSound(isCritical, isCooperative);

    // 3. ダメージテキスト表示
    ShowDamageText(hitPosition, damage, isCritical);
}

/// <summary>
/// ヒットパーティクルを再生する
/// </summary>
private void PlayHitParticle(Vector3 position, Vector3 normal, bool isCritical, bool isCooperative)
{
    GameObject particlePrefab = null;

    // パーティクルの種類を選択
    if (isCooperative)
    {
        particlePrefab = hitParticleCooperativePrefab;
    }
    else if (isCritical)
    {
        particlePrefab = hitParticleCriticalPrefab;
    }
    else
    {
        particlePrefab = hitParticleNormalPrefab;
    }

    // Null チェック
    if (particlePrefab == null)
    {
        Debug.LogError("[AxeInteraction] Hit particle prefab is null");
        return;
    }

    // パーティクルを生成
    GameObject particleObj = Instantiate(particlePrefab, position, Quaternion.LookRotation(normal));

    // 自動削除（Duration + 1秒後）
    Destroy(particleObj, 2.0f);
}

/// <summary>
/// ヒット音を再生する
/// </summary>
private void PlayHitSound(bool isCritical, bool isCooperative)
{
    AudioClip soundClip = null;
    float volume = 0.7f;

    // サウンドの種類を選択
    if (isCooperative)
    {
        soundClip = hitSoundCooperative;
        volume = 0.7f;
    }
    else if (isCritical)
    {
        soundClip = hitSoundCritical;
        volume = 0.8f;
    }
    else
    {
        soundClip = hitSoundNormal;
        volume = 0.7f;
    }

    // Null チェック
    if (soundClip == null)
    {
        Debug.LogError("[AxeInteraction] Hit sound clip is null");
        return;
    }

    if (audioSource == null)
    {
        Debug.LogError("[AxeInteraction] AudioSource is null");
        return;
    }

    // 3D空間オーディオで再生（PlayOneShot: 同時再生可能）
    audioSource.PlayOneShot(soundClip, volume);
}

/// <summary>
/// ダメージテキストを表示する
/// </summary>
private void ShowDamageText(Vector3 basePosition, int damage, bool isCritical)
{
    // Null チェック
    if (damageTextPrefab == null)
    {
        Debug.LogError("[AxeInteraction] DamageText prefab is null");
        return;
    }

    // テキスト表示位置（木の上部）
    Vector3 textPosition = basePosition + Vector3.up * damageTextYOffset;

    // ダメージテキストを生成
    GameObject damageTextObj = Instantiate(damageTextPrefab, textPosition, Quaternion.identity);

    // TextMeshPro コンポーネントを取得
    TextMeshProUGUI textComponent = damageTextObj.GetComponentInChildren<TextMeshProUGUI>();
    if (textComponent == null)
    {
        Debug.LogError("[AxeInteraction] TextMeshProUGUI component not found in DamageText prefab");
        Destroy(damageTextObj);
        return;
    }

    // ダメージ数値を設定
    textComponent.text = $"-{damage}";

    // クリティカルの場合は赤色・大きめに
    if (isCritical)
    {
        textComponent.color = new Color(1.0f, 0.2f, 0.2f, 1.0f); // 赤色
        textComponent.fontSize = textComponent.fontSize * 1.5f;
    }
    else
    {
        textComponent.color = Color.white; // 白色
    }

    // フェードアウトアニメーション開始
    StartCoroutine(FadeOutDamageText(damageTextObj));
}

/// <summary>
/// ダメージテキストをフェードアウトさせる
/// </summary>
private IEnumerator FadeOutDamageText(GameObject damageTextObj)
{
    CanvasGroup canvasGroup = damageTextObj.GetComponent<CanvasGroup>();
    if (canvasGroup == null)
    {
        Debug.LogError("[AxeInteraction] CanvasGroup component not found");
        Destroy(damageTextObj);
        yield break;
    }

    // 1秒間表示
    yield return new WaitForSeconds(damageTextDuration);

    // 0.5秒かけてフェードアウト
    float elapsedTime = 0f;
    float startAlpha = canvasGroup.alpha;

    while (elapsedTime < damageTextFadeTime)
    {
        elapsedTime += Time.deltaTime;
        float alpha = Mathf.Lerp(startAlpha, 0f, elapsedTime / damageTextFadeTime);
        canvasGroup.alpha = alpha;
        yield return null;
    }

    // 完全に透明になったら削除
    Destroy(damageTextObj);
}
```

#### 8.7.3. OnCollisionEnter メソッドの修正
4. 既存の `OnCollisionEnter()` メソッド内で、ダメージ計算後にエフェクト再生を呼び出す:

```csharp
void OnCollisionEnter(Collision collision)
{
    // 木のタグチェック
    if (collision.gameObject.CompareTag("Tree"))
    {
        // ダメージ計算（既存実装）
        int damage = CalculateDamage(out bool isCritical, out bool isCooperative);

        // 木にダメージを与える（既存実装）
        TreeController treeController = collision.gameObject.GetComponent<TreeController>();
        if (treeController != null)
        {
            treeController.TakeDamage(damage, Networking.LocalPlayer);
        }

        // ヒット位置と法線を取得
        Vector3 hitPosition = collision.contacts[0].point;
        Vector3 hitNormal = collision.contacts[0].normal;

        // ★ エフェクト再生（新規追加）
        PlayHitEffect(hitPosition, hitNormal, damage, isCritical, isCooperative);
    }
}
```

5. ファイルを保存（Ctrl + S）

---

### 8.8. Unity エディタでの設定

#### 8.8.1. AxeInteraction GameObject への参照設定
1. Unityエディタに戻る
2. Hierarchyウィンドウで斧オブジェクト（例: `Axe_Wood`）を選択
3. Inspectorで `AxeInteraction` スクリプトコンポーネントを確認
4. 以下のフィールドに対応するアセットをドラッグ&ドロップで設定:

   **Hit Effects セクション**:
   - Hit Particle Normal Prefab: `Assets/WoodcutterCamp/Prefabs/Effects/HitParticle_Normal.prefab`
   - Hit Particle Critical Prefab: `Assets/WoodcutterCamp/Prefabs/Effects/HitParticle_Critical.prefab`
   - Hit Particle Cooperative Prefab: `Assets/WoodcutterCamp/Prefabs/Effects/HitParticle_Cooperative.prefab`
   - Damage Text Prefab: `Assets/WoodcutterCamp/Prefabs/UI/DamageText.prefab`

   **Hit Sounds セクション**:
   - Hit Sound Normal: `Assets/WoodcutterCamp/Audio/SFX/hit_normal.wav`
   - Hit Sound Critical: `Assets/WoodcutterCamp/Audio/SFX/hit_critical.wav`
   - Hit Sound Cooperative: `Assets/WoodcutterCamp/Audio/SFX/hit_cooperative.wav`
   - Audio Source: （次のステップで設定）

#### 8.8.2. AudioSource コンポーネントの追加
1. Axe_Wood オブジェクトを選択した状態で、Inspectorの「Add Component」をクリック
2. `Audio Source` を検索して追加
3. AudioSource コンポーネントを以下のように設定:

   **AudioSource 設定**:
   - AudioClip: （空欄のまま）※PlayOneShotで動的に再生
   - Play On Awake: `false`（チェックを外す）
   - Loop: `false`
   - Volume: `0.7`
   - Pitch: `1.0`
   - Spatial Blend: `1.0`（完全に3D音響）
   - Doppler Level: `0`（ドップラー効果なし）
   - Min Distance: `1`メートル
   - Max Distance: `20`メートル（PRD仕様に準拠）
   - Rolloff Mode: `Logarithmic Rolloff`（自然な減衰）

4. Inspector の `AxeInteraction` コンポーネントに戻る
5. **Audio Source** フィールドに、同じオブジェクトの AudioSource コンポーネントをドラッグ&ドロップ

#### 8.8.3. Effect Settings の調整
6. `AxeInteraction` コンポーネントの **Effect Settings** セクションで以下を確認:
   - Damage Text Duration: `1.0`秒（表示時間）
   - Damage Text Fade Time: `0.5`秒（フェードアウト時間）
   - Damage Text Y Offset: `1.5`メートル（木の上部に表示）

7. 設定完了後、シーンを保存（Ctrl + S）

---

### 8.9. テスト実行

#### 8.9.1. Unity エディタ内でのテスト
1. Unityエディタで `Play` ボタンをクリック
2. シーン内で斧を装備して木に近づく
3. 以下の動作を確認:
   - 木を叩いたときに **通常ヒット音** が再生される
   - **木片パーティクル** が5個飛散する
   - 木の上部に **白色のダメージ数値**（例: `-12`）が表示される
   - 1.5秒後にダメージテキストがフェードアウトする

#### 8.9.2. クリティカルヒットのテスト
1. クリティカルが発生するまで木を叩く（ランダム発生）
2. クリティカル発生時、以下を確認:
   - **「キン！」という特殊音**（hit_critical.wav）が再生される
   - **赤い衝撃波パーティクル** が10個放射状に広がる
   - **赤色・1.5倍サイズのダメージ数値**（例: `-24`）が表示される

#### 8.9.3. 協力伐採のテスト
1. 別のプレイヤーと同じ木を3秒以内に攻撃（テストではダミープレイヤーでシミュレート）
2. 協力伐採発動時、以下を確認:
   - **「キラーン」という音**（hit_cooperative.wav）が再生される
   - **金色のキラキラパーティクル** が7個上昇する
   - ダメージテキストが表示される

#### 8.9.4. Quest 最適化の確認
1. Unityエディタで `Window > Analysis > Profiler` を開く
2. Playモードで木を連続で叩く
3. 以下を確認:
   - **パーティクル総数** が30個を超えていないこと（Profiler > Particles）
   - **AudioSourceのCPU使用率** が5%以下であること
   - **FPS** が60fpsを維持していること（Quest環境想定）

---

## 9. 完了条件（Doneの条件）

以下のすべての条件を満たすこと:

### 9.1. 機能実装の完了
- [ ] 通常ヒット時に木片パーティクル5個が再生される
- [ ] クリティカルヒット時に赤い衝撃波パーティクル10個が再生される
- [ ] 協力伐採時に金色のキラキラパーティクル7個が再生される
- [ ] 各ヒットタイプに対応する効果音（normal/critical/cooperative）が3D空間オーディオで再生される
- [ ] ダメージ数値が木の上部に表示される（通常: 白色、クリティカル: 赤色・1.5倍サイズ）
- [ ] ダメージテキストが1秒表示後、0.5秒でフェードアウトする

### 9.2. Quest 最適化の完了
- [ ] パーティクル総数が30個以下に抑えられている
- [ ] AudioSourceの Max Distance が20mに設定されている
- [ ] AudioClipのCompression Formatが Android（Quest）向けに `ADPCM` に設定されている
- [ ] ParticleSystemの Cast Shadows / Receive Shadows が `Off` に設定されている

### 9.3. コード品質の完了
- [ ] `PlayHitEffect()`, `PlayHitParticle()`, `PlayHitSound()`, `ShowDamageText()` の各メソッドが実装されている
- [ ] すべてのフィールド変数に適切なコメントが記載されている
- [ ] Null チェックが実装されており、エラーログが出力される
- [ ] コードに警告（Warning）が0件である

### 9.4. テストの完了
- [ ] Unity Playモードで通常ヒット・クリティカル・協力伐採のすべてのエフェクトが正常に再生される
- [ ] 複数の木を連続で叩いても、パーティクルとサウンドが正常に再生される
- [ ] ダメージテキストが正確な位置（木の上部 +1.5m）に表示される
- [ ] Profilerでパフォーマンスが確認され、FPSが60を維持している

---

## 10. テスト観点／テストケース

### 10.1. テスト種別
- **ユニットテスト**: 各メソッド（PlayHitParticle, PlayHitSound, ShowDamageText）が正常に動作するか
- **結合テスト**: AxeInteraction ↔ TreeController の連携が正常に機能するか
- **パフォーマンステスト**: Quest環境でFPS 60を維持できるか

### 10.2. テストケース

#### 正常系
| テストID | テスト内容 | 期待結果 |
|---------|-----------|---------|
| TC-0011-01 | 通常ヒット時のパーティクル再生 | 木片パーティクル5個が飛散する |
| TC-0011-02 | クリティカルヒット時のパーティクル再生 | 赤い衝撃波パーティクル10個が放射状に広がる |
| TC-0011-03 | 協力伐採時のパーティクル再生 | 金色のキラキラパーティクル7個が上昇する |
| TC-0011-04 | 通常ヒット音の再生 | hit_normal.wav が70%音量、20m減衰で再生される |
| TC-0011-05 | クリティカルヒット音の再生 | hit_critical.wav が80%音量で再生される |
| TC-0011-06 | 協力伐採音の再生 | hit_cooperative.wav が70%音量で再生される |
| TC-0011-07 | 通常ダメージテキスト表示 | 白色テキスト、サイズ1.0倍で表示される |
| TC-0011-08 | クリティカルダメージテキスト表示 | 赤色テキスト、サイズ1.5倍で表示される |
| TC-0011-09 | ダメージテキストのフェードアウト | 1秒表示後、0.5秒でフェードアウトする |
| TC-0011-10 | 複数の木を連続攻撃 | すべての木で正常にエフェクトが再生される |

#### 異常系
| テストID | テスト内容 | 期待結果 |
|---------|-----------|---------|
| TC-0011-E01 | Prefabが設定されていない | エラーログ「Hit particle prefab is null」が出力され、処理がスキップされる |
| TC-0011-E02 | AudioClipが設定されていない | エラーログ「Hit sound clip is null」が出力され、処理がスキップされる |
| TC-0011-E03 | AudioSourceが設定されていない | エラーログ「AudioSource is null」が出力され、処理がスキップされる |
| TC-0011-E04 | DamageText Prefabに TextMeshProUGUI がない | エラーログ「TextMeshProUGUI component not found」が出力され、オブジェクトが削除される |
| TC-0011-E05 | パーティクル総数が30個を超える | 古いパーティクルから自動削除される（Destroy 2秒後） |

#### パフォーマンステスト
| テストID | テスト内容 | 目標値 | 確認方法 |
|---------|-----------|-------|---------|
| TC-0011-P01 | FPS維持 | 60fps以上 | Unity Profiler > FPS |
| TC-0011-P02 | パーティクル総数 | 30個以下 | Profiler > Particles |
| TC-0011-P03 | AudioSource CPU使用率 | 5%以下 | Profiler > Audio |
| TC-0011-P04 | メモリ使用量 | +10MB以下 | Profiler > Memory |

---

## 11. 成果物

### 11.1. 変更ファイル
- `Assets/WoodcutterCamp/Scripts/Gameplay/Woodcutting/AxeInteraction.cs`
  - `PlayHitEffect()`, `PlayHitParticle()`, `PlayHitSound()`, `ShowDamageText()`, `FadeOutDamageText()` メソッドを追加
  - フィールド変数追加（Prefab参照、AudioClip参照、エフェクト設定）

### 11.2. 追加ファイル（Prefab）
- `Assets/WoodcutterCamp/Prefabs/Effects/HitParticle_Normal.prefab`
- `Assets/WoodcutterCamp/Prefabs/Effects/HitParticle_Critical.prefab`
- `Assets/WoodcutterCamp/Prefabs/Effects/HitParticle_Cooperative.prefab`
- `Assets/WoodcutterCamp/Prefabs/UI/DamageText.prefab`

### 11.3. 追加ファイル（Audio）
- `Assets/WoodcutterCamp/Audio/SFX/hit_normal.wav`（インポート設定変更）
- `Assets/WoodcutterCamp/Audio/SFX/hit_critical.wav`（インポート設定変更）
- `Assets/WoodcutterCamp/Audio/SFX/hit_cooperative.wav`（インポート設定変更）

### 11.4. 更新したドキュメント
- 本作業指示書（WI-0011_Hit_Effects.md）を `docs/WID/` に配置

---

## 12. 備考

### 12.1. 注意点
- **TextMeshProのインポート**: 初回使用時に「TMP Importer」が表示されるので、必ず「Import TMP Essentials」を実行すること
- **Coroutineの使用**: UdonSharpでは一部のCoroutine機能に制限があるため、フェードアウトが正常に動作しない場合はUpdate()ベースの実装に切り替えること
- **3D音響の設定**: AudioSourceの Spatial Blend を必ず `1.0` に設定すること（0だと2D音響になり、距離減衰が効かない）
- **パーティクル自動削除**: Instantiateしたパーティクルは `Destroy(particleObj, 2.0f)` で自動削除されるため、手動管理不要

### 12.2. Quest 最適化に関する制約
- パーティクル総数は **30個以下** に抑えること（PRD仕様）
- AudioClipのCompression FormatはAndroid向けに **ADPCM** を使用すること（低遅延・低CPU負荷）
- ParticleSystemの Cast Shadows / Receive Shadows は **Off** にすること（Quest性能向上）
- AudioSourceの Max Distance は **20m** に設定すること（PRD Section 2.8.1準拠）

### 12.3. 今後の拡張予定
- Phase 2で追加予定のエフェクト:
  - 倒木時の大規模パーティクル演出（木片・葉っぱ大量）
  - カメラシェイク（VRでは控えめに）
  - 地面衝撃エフェクト（土煙）
- これらは別のWI（WI-0017: Tree Falling Animation）で実装予定

### 12.4. 参考資料
- **PRD.md Section 2.8.1**: サウンドデザイン仕様（音量・距離減衰の詳細）
- **FSD.md FNC-001**: 伐採システムの詳細仕様（エフェクトの種類と条件）
- **TDD.md Section 5.11**: M-11 AxeInteraction モジュール定義（F-38: Play Hit Effects）
- **Unity Manual - Particle System**: https://docs.unity3d.com/Manual/PartialSysReference.html
- **Unity Manual - Audio Source**: https://docs.unity3d.com/Manual/class-AudioSource.html
- **TextMeshPro Documentation**: https://docs.unity3d.com/Packages/com.unity.textmeshpro@latest

---

**作業指示書作成日**: 2025-11-17
**推定作業時間**: 0.5日（4時間）
**優先度**: 中
**依存作業**: WI-0010 (Damage Calculation)
**後続作業**: WI-0012 (Implement LogSpawner)
