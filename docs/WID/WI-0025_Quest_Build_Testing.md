# 作業指示書 WI-0025

**作成日**: 2025-11-17
**対象プロジェクト**: 森のきこりキャンプ - VRChatワールド
**フェーズ**: Phase 3（統合テスト・最適化）
**優先度**: 最高
**推定作業日数**: 1日

---

## 1. 作業ID

**WI-0025 / PHASE-03_TASK-0025**

---

## 2. 作業名

**Quest Build & Testing - Questプラットフォームビルドと統合テスト**

---

## 3. 作業目的

Phase 1で実装した全25のWork Instruction（WI-0001〜WI-0024）の統合テストを行い、Quest 2/3/Proプラットフォーム向けの最終ビルドを作成・検証する。これにより、VRChatワールド「森のきこりキャンプ」のPhase 1リリース準備を完了させる。

---

## 4. 対象

### 対象システム
- 森のきこりキャンプ VRChatワールド - Phase 1全機能

### 対象モジュール
- **全モジュール（M-01〜M-16）** - 統合テスト
- **Questプラットフォームビルド** - Android向けビルド設定とアップロード

### 対象ファイルパス
- プロジェクト全体: `Assets/WoodcutterCamp/`
- シーン: `Assets/Scenes/MainWorld.unity`
- ビルド設定: Unity Player Settings / VRChat SDK Builder

### 関連ドキュメント
- `docs/TDD.md` - Section 6.2（WI-0025マッピング）
- `docs/PRDv3.md` - Quest要件仕様
- `docs/FSD.md` - 全機能仕様
- `docs/ref/VRChat_Tools_API_Reference.md` - Build & Test パイプライン

---

## 5. 前提条件

### 環境要件
- Unity 2022.3.22f1がインストール済み
- VRChat SDK 3.9.0がプロジェクトにインポート済み
- UdonSharp 1.1.9がプロジェクトにインポート済み
- VRChat Creator Companion (VCC) 2.4.0以降でプロジェクト管理
- **Android Build Supportモジュール**がUnityにインストール済み（必須）

### プロジェクト状態
- Gitブランチ: `main`または`release/phase-1`
- **WI-0001〜WI-0024がすべて実装・テスト完了済み**
- 全スクリプトがコンパイルエラーなし
- ClientSimでローカルテスト済み

### テスト機材
- **必須**: Quest 2またはQuest 3実機（物理デバイステスト用）
- **推奨**: PC VRヘッドセット（クロスプラットフォームテスト用）
- **オプション**: デスクトップモード（補助的なテスト）

### VRChatアカウント要件
- VRChatアカウントでログイン可能
- ワールドアップロード権限あり（Trust Rank: New User以上）

---

## 6. 入力

### 実装済みモジュール（Phase 0〜Phase 2）
以下の全モジュールが正常動作していることが前提：

#### Phase 0: 基盤実装
- WI-0001: GameManager Singleton
- WI-0002: NetworkSync Batching
- WI-0003: PersistenceManager
- WI-0004: Late Joiner Handling

#### Phase 1: コア機能
- WI-0005: SkillManager
- WI-0006: SkillEffects
- WI-0007: TreeController State
- WI-0008: Tree Falling Animation & Respawn
- WI-0009: Axe Swing Detection
- WI-0010: Damage Calculation
- WI-0011: Hit Effects
- WI-0012: LogSpawner with Object Pool

#### Phase 2: 補助機能
- WI-0013: CoinManager
- WI-0014: Login Bonus
- WI-0015: LogPickup VRC_Pickup
- WI-0016: Delivery Zone
- WI-0017: HUDManager
- WI-0018: HUD Settings
- WI-0019: ShopManager
- WI-0020: CooperativeTracker
- WI-0021: RankingTracker
- WI-0022: LeaderboardUI
- WI-0023: CuttingTable

#### Phase 3: 最適化
- WI-0024: Performance Optimization

---

## 7. 出力

### 成果物
1. **Questビルド済みワールド** - VRChatサーバーにアップロード完了
2. **テストレポート** - 統合テスト結果の記録（本作業指示書に記載）
3. **既知のバグリスト** - 発見されたバグのトラッキングシート
4. **リリースノート** - Phase 1機能概要とユーザーガイド

### 期待される動作
- Quest 2で60fps維持（10人同時接続時）
- Performance Rank: "Good"以上
- 全25機能が正常動作
- ネットワーク帯域: 5KB/秒以下（プレイヤーあたり）
- クリティカルバグなし

---

## 8. 作業手順

### ステップ1: Pre-Build Checklist（ビルド前確認）

#### 1.1 スクリプトコンパイル確認

**操作**:
1. Unity Editorを開く
2. **メニューバー → VRChat SDK → Show Control Panel**
3. **Builder タブ** を開く
4. "Check for Issues"ボタンをクリック

**確認項目**:
- [ ] Console: 0 Errors（エラーがないこと）
- [ ] VRChat SDK Validation: No Critical Issues（重大な問題なし）
- [ ] 全UdonSharpスクリプトがコンパイル済み

**エラー対処**:
- エラーがある場合、該当するWI（WI-0001〜0024）に戻って修正
- コンパイルエラーは**すべて解決必須**

---

#### 1.2 Missing Scripts除去

**操作**:
1. **VRChat SDK → Utilities → Remove Missing Scripts**をクリック
2. 実行確認ダイアログで"Proceed"をクリック
3. Consoleで削除結果を確認

**確認項目**:
- [ ] Missing Scriptsが0件（または削除完了メッセージ）

**説明**:
- Missing Scriptsはビルドエラーの原因となるため必ず除去

---

#### 1.3 Quest非互換コンポーネント確認

**操作**:
Hierarchyとシーン内を目視確認し、以下のQuest非互換要素がないかチェック

**確認項目**:
- [ ] **Post-Processing Stack v2**: 使用していない（Questは未対応）
- [ ] **Standard Shader**: Mobile/Unlitに変更済み
- [ ] **Realtime Lights**: 2個以下（太陽光 + 焚き火のみ）
- [ ] **Reflection Probes**: ベイク済み（Realtime Reflectionなし）
- [ ] **パーティクルシステム**: Max Particles < 500（各エミッター）

**非互換要素の修正方法**:

**Standard Shaderの変更**:
1. Project → Assets/Materials フォルダを開く
2. 各マテリアルを選択
3. Inspector → Shader → Mobile/Diffuse または Mobile/Unlit に変更

**Realtime Lightsの削減**:
1. Hierarchy → Lightsフォルダを開く
2. 不要なライトを削除
3. 必須ライトのみ残す（Directional Light + Campfire Light）

---

#### 1.4 VRChat SDK Validation実行

**操作**:
1. **VRChat SDK → Show Control Panel → Builder タブ**
2. "Build Target: Android"を選択（Questプラットフォーム）
3. **"Validate"ボタン**をクリック

**確認項目**:
- [ ] Validation Result: "All Validations Passed"（緑色）
- [ ] 警告（Warnings）のみであればOK、エラー（Errors）は修正必須

**よくある警告と対処**:

| 警告メッセージ | 対処方法 |
|-------------|---------|
| "Shader not supported on Quest" | Mobile Shaderに変更 |
| "Post-Processing Stack detected" | Post-Processingコンポーネント削除 |
| "Realtime GI enabled" | Lighting Settings → Realtime GI: Off |
| "Occlusion Culling disabled" | Window → Rendering → Occlusion Culling → Bake |

---

### ステップ2: Unity Project Settings for Quest（Quest向けプロジェクト設定）

#### 2.1 Player Settings確認

**操作**:
1. **File → Build Settings**を開く
2. Platform: "Android"を選択
3. "Player Settings..."ボタンをクリック

**設定確認と変更**:

**2.1.1 Color Space**
- **Edit → Project Settings → Player → Other Settings**
- Color Space: **Linear**（Gammaは使用しない）
- 理由: Linearの方が正確な色表現（Questも対応）

**2.1.2 Graphics API**
- **Other Settings → Graphics APIs for Android**
- **Vulkan**のみを残す
- **OpenGLES3を削除**（リストから"-"ボタンで削除）
- 理由: Quest 2/3はVulkanが最適

**2.1.3 Minimum API Level**
- **Other Settings → Minimum API Level**
- **Android 10.0 (API Level 29)** 以上
- 理由: Quest 2の最低要件

**2.1.4 Scripting Backend**
- **Other Settings → Scripting Backend**
- **IL2CPP**（Monoは使用しない）
- 理由: Androidビルド必須

**2.1.5 Target Architectures**
- **Other Settings → Target Architectures**
- **ARM64のみにチェック**
- ARMv7のチェックを外す
- 理由: Quest 2/3は64bit専用

**確認項目**:
- [ ] Color Space: Linear
- [ ] Graphics API: Vulkan only
- [ ] Minimum API Level: Android 10.0+
- [ ] Scripting Backend: IL2CPP
- [ ] Target Architectures: ARM64 only

---

#### 2.2 Quality Settings確認

**操作**:
1. **Edit → Project Settings → Quality**
2. Androidプラットフォーム用設定を確認

**設定確認と変更**:

| 設定項目 | Quest 2 | Quest 3 | 理由 |
|---------|---------|---------|------|
| **Texture Quality** | Full | Full | テクスチャ圧縮で対応 |
| **Anisotropic Textures** | Per Texture | Per Texture | 個別最適化 |
| **Anti Aliasing** | 2x MSAA | 4x MSAA | Quest 3は高性能 |
| **Soft Particles** | Off | On | Quest 3で有効化 |
| **Shadow Quality** | Hard Shadows Only | Soft Shadows | Quest 2は低設定 |
| **Shadow Resolution** | Medium | High | Quest 3は高品質 |
| **Shadow Distance** | 50 | 80 | 描画距離 |
| **Shadow Cascades** | 2 Cascades | 4 Cascades | Quest 3のみ |

**操作手順**:
1. Quality → Levels → "Android"を選択
2. 上記の設定値に変更
3. **Apply**をクリック

**確認項目**:
- [ ] Quest 2向け設定完了（保守的な設定）
- [ ] Quest 3向け設定完了（高品質設定）

---

#### 2.3 VRChat Settings確認

**操作**:
1. **VRChat SDK → Show Control Panel → Builder タブ**
2. "Build Target: Android"を選択

**設定項目**:

**2.3.1 World Name**
- 名前: **森のきこりキャンプ / Forest Woodcutter Camp**
- 文字数制限: 64文字以内

**2.3.2 World Description**
```
【日本語】
Valheim風の協力伐採ワールド！
木を切って丸太を運び、スキルを成長させよう。

【English】
Valheim-inspired cooperative woodcutting world!
Chop trees, carry logs, and grow your skills.

Quest 2/3/Pro対応 | PC VR & Desktop compatible
Phase 1: Core Gameplay
```

**2.3.3 World Capacity**
- **推奨: 16人**（最大20人まで対応）
- 理由: 協力プレイを促進しつつ、Quest性能維持

**2.3.4 Tags**
- Game
- Quest Compatible
- PvE
- Chill/Relaxing
- Japanese

**2.3.5 Content Warnings**
- なし（暴力・性的要素なし）

**2.3.6 Thumbnail Image**
- 解像度: 1200x900 PNG
- 内容: キャンプ全景と木を切っているプレイヤー
- 準備: `docs/WID/assets/thumbnail.png`に配置

**確認項目**:
- [ ] World Name設定完了
- [ ] Description設定完了（日英両方）
- [ ] Capacity: 16人
- [ ] Tags設定完了
- [ ] Thumbnail画像準備完了

---

### ステップ3: Quest Build Process（Questビルド手順）

#### 3.1 Androidプラットフォームへの切り替え

**操作**:
1. **File → Build Settings**を開く
2. "Platform"リストから**Android**を選択
3. **"Switch Platform"ボタン**をクリック

**注意事項**:
- **再インポートに10〜30分かかる**（プロジェクト規模による）
- Unity Editorが応答なしになることがあるが、正常動作（待機必須）
- Console下部に"Importing... (X / Y)"と表示される

**確認項目**:
- [ ] Platform: Androidに切り替わった（Unityアイコンが表示）
- [ ] 再インポート完了（Console: "Importing... Done"）

**トラブルシューティング**:
- 30分以上進まない場合: Unity再起動後に再度Switch Platform

---

#### 3.2 VRChat SDK Build & Test（ローカルテスト）

**操作**:
1. **VRChat SDK → Show Control Panel**を開く
2. **Authentication タブ** → VRChatアカウントでログイン
3. **Builder タブ** → "Build Target: Android"を選択

**Build & Test手順**:
1. **"Offline Testing"セクション**
2. **"Build & Test"ボタン**をクリック
3. ビルド進行状況を確認（Console）

**確認項目**:
- [ ] ビルド成功（Console: "Build Successful"）
- [ ] VRChatクライアントが自動起動
- [ ] ワールドがローカルテストモードで読み込まれる

**ビルドエラー時の対処**:
| エラーメッセージ | 原因 | 対処法 |
|---------------|------|--------|
| "Missing Scripts found" | 削除されたコンポーネント | ステップ1.2に戻る |
| "Shader error" | Quest非対応Shader | ステップ1.3に戻る |
| "Script compilation failed" | スクリプトエラー | 該当WIに戻って修正 |

---

#### 3.3 ローカルテストの実行

**操作**:
VRChatクライアント内で動作確認

**確認項目**:
- [ ] ワールドが正常に読み込まれる
- [ ] 全オブジェクトが表示される
- [ ] UIが表示される（HUD、ランキングボード）
- [ ] 移動できる
- [ ] 斧を拾える

**簡易テスト**:
1. 木を1本伐採
2. 丸太を1本運搬
3. HUDでXPとコイン増加を確認
4. ショップを開く（動作確認のみ）

**確認項目**:
- [ ] 基本的な動作が正常
- [ ] クリティカルエラーなし

**問題があった場合**:
- VRChatを終了
- Unity Editorで該当WIを修正
- ステップ3.2に戻って再ビルド

---

#### 3.4 VRChatへのアップロード

**操作**:
ローカルテストでOKならば、VRChatサーバーにアップロード

**アップロード手順**:
1. **VRChat SDK → Show Control Panel → Builder タブ**
2. **"Online Publishing"セクション**
3. 以下を再確認:
   - World Name
   - Description
   - Capacity: 16
   - Tags
   - Thumbnail画像

4. **"Build & Publish"ボタン**をクリック

**アップロード進行状況**:
- ビルド中: "Building..." → "Uploading..." → "Processing..."
- 所要時間: 5〜15分（ワールドサイズとネットワーク速度による）

**確認項目**:
- [ ] アップロード成功（"Upload Successful"メッセージ）
- [ ] VRChat SDK Control Panel → Content Managerに表示される
- [ ] World Status: "Published"

**アップロード失敗時の対処**:
| エラーメッセージ | 原因 | 対処法 |
|---------------|------|--------|
| "Upload timeout" | ネットワークエラー | 再度アップロード試行 |
| "World too large" | ファイルサイズ超過 | アセット最適化（テクスチャ圧縮） |
| "Authentication failed" | ログイン切れ | Authenticationタブで再ログイン |

---

### ステップ4: Quest Device Testing（Quest実機テスト）

#### 4.1 Quest実機でワールド参加

**前提**:
- Quest 2/3にVRChatアプリがインストール済み
- VRChatアカウントでログイン済み

**操作**:
1. Quest 2/3を装着してVRChatを起動
2. **メニュー → Worlds → My Worlds**
3. **"森のきこりキャンプ"**を選択
4. **"Go"ボタン**で参加

**確認項目**:
- [ ] ワールドが正常にロードされる（ローディング画面）
- [ ] スポーン地点（キャンプ中央）に配置される
- [ ] 全オブジェクトが表示される

---

#### 4.2 Performance Testing（パフォーマンステスト）

**操作**:
Quest内でパフォーマンス統計を確認

**Performance Stats表示方法**:
1. VRChatメニューを開く（Questボタン）
2. **Settings → Performance → Show Performance Stats**
3. HUD左上に統計情報が表示される

**確認項目**:

**4.2.1 FPS（フレームレート）**
- **Quest 2目標**: 60fps以上（常時）
- **Quest 3目標**: 72fps以上（理想は90fps）
- 測定: 5分間プレイして平均FPSを記録

**確認**:
- [ ] 静止時: 60fps以上（Quest 2）/ 72fps以上（Quest 3）
- [ ] 木伐採時: 55fps以上（Quest 2）/ 65fps以上（Quest 3）
- [ ] 10人同時接続時: 50fps以上（Quest 2）/ 60fps以上（Quest 3）

**4.2.2 Frame Time（フレーム時間）**
- **目標**: 16.6ms以下（60fps = 16.6ms）
- **Quest 3**: 13.8ms以下（72fps）

**確認**:
- [ ] 平均Frame Time: 16.6ms以下

**4.2.3 Performance Rank（パフォーマンスランク）**
- VRChat独自の性能評価指標
- **目標**: "Good"以上（"Excellent" > "Good" > "Medium" > "Poor"）

**Performance Rank確認方法**:
1. VRChatメニュー → Worlds → My Worlds → ワールド選択
2. "Performance Rank"欄を確認

**確認**:
- [ ] Performance Rank: "Good"以上

**パフォーマンスランクが低い場合の対処**:
- WI-0024（Performance Optimization）に戻る
- ポリゴン数削減
- テクスチャ圧縮
- LOD設定見直し

---

#### 4.3 Functionality Testing（機能テスト）

**操作**:
Quest実機で全25機能を順次テスト

**テストケース一覧**:

**【Phase 0: 基盤】**
- [ ] WI-0001: GameManager初期化（Console確認）
- [ ] WI-0002: NetworkSync動作（他プレイヤー参加時）
- [ ] WI-0003: PlayerData読み込み（初回ボーナス付与）
- [ ] WI-0004: Late Joiner対応（後から参加時に同期）

**【Phase 1: コア機能】**
- [ ] WI-0005: SkillManagerでXP加算（木を切ってXP獲得）
- [ ] WI-0006: SkillEffectsでダメージボーナス（レベルアップ後）
- [ ] WI-0007: TreeController状態管理（木のHP減少）
- [ ] WI-0008: 倒木アニメーション（木が倒れる）
- [ ] WI-0009: 斧スイング判定（VRトリガー/Desktopクリック）
- [ ] WI-0010: ダメージ計算（スキル効果反映）
- [ ] WI-0011: ヒットエフェクト（パーティクル、サウンド）
- [ ] WI-0012: 丸太スポーン（倒木後に生成）

**【Phase 2: 補助機能】**
- [ ] WI-0013: CoinManagerでコイン加算（伐採報酬）
- [ ] WI-0014: ログインボーナス（初回50コイン付与）
- [ ] WI-0015: 丸太Pickup（掴める、運べる）
- [ ] WI-0016: 納品ゾーン（丸太Drop時にコイン付与）
- [ ] WI-0017: HUD表示（スキルレベル、XP、コイン残高）
- [ ] WI-0018: HUD設定（透明度調整可能）
- [ ] WI-0019: ショップ（斧購入可能）
- [ ] WI-0020: 協力伐採ボーナス（2人以上で+20%）
- [ ] WI-0021: ランキングトラッキング（伐採数記録）
- [ ] WI-0022: リーダーボードUI（TOP3表示）
- [ ] WI-0023: 切断台（丸太→板材変換）

**テスト手順例（基本フロー）**:
1. **木を1本伐採**
   - 斧を拾う（WI-0009）
   - 木を叩く（WI-0010）
   - ヒットエフェクト確認（WI-0011）
   - 木のHP減少（WI-0007）
   - 倒木アニメーション（WI-0008）
   - 丸太生成（WI-0012）
   - XP加算（WI-0005）
   - コイン加算（WI-0013）

2. **丸太を運搬**
   - 丸太をPickup（WI-0015）
   - 納品ゾーンへ運ぶ（WI-0015）
   - Drop（WI-0016）
   - コイン付与（WI-0016）

3. **HUD確認**
   - スキルレベル表示（WI-0017）
   - XPバー表示（WI-0017）
   - コイン残高表示（WI-0017）

4. **ショップ利用**
   - ショップNPCに近づく（WI-0019）
   - 鉄の斧を購入（WI-0019）
   - コイン減算確認（WI-0013）

5. **協力伐採テスト**（2人以上必要）
   - 同じ木を2人で叩く（WI-0020）
   - 協力ボーナス発動（+20%）
   - 金色エフェクト確認（WI-0020）

**確認項目**:
- [ ] 全25機能が正常動作
- [ ] クリティカルバグなし（ワールドクラッシュ、進行不能など）

---

#### 4.4 VR Controller Input Testing（VRコントローラー入力テスト）

**操作**:
Quest実機でVRコントローラー操作を確認

**テストケース**:

**4.4.1 斧スイング**
- Triggerボタン: 斧スイング
- 確認: 木にヒット判定が発生

**4.4.2 丸太Pickup**
- Gripボタン: 丸太を掴む
- Triggerボタン: 丸太をDropしない（AutoHold設定確認）
- 確認: 正常にPickup/Drop可能

**4.4.3 UI操作**
- Aボタン: メニュー開閉
- レーザーポインター: UIボタンクリック
- 確認: ショップUI、HUD設定が操作可能

**4.4.4 移動**
- 左スティック: 移動
- 右スティック: 回転
- 確認: スムーズに移動可能

**確認項目**:
- [ ] 全VR操作が正常動作
- [ ] 誤入力なし（意図しない動作なし）

---

#### 4.5 UI Readability Testing（UI可読性テスト）

**操作**:
Quest実機でUIテキストの読みやすさを確認

**確認項目**:

**4.5.1 HUD（左下）**
- [ ] スキルレベル表示が読める
- [ ] XPバーが見える
- [ ] コイン残高が読める
- [ ] フォントサイズ: 適切（小さすぎない）

**4.5.2 リーダーボード（キャンプ中央）**
- [ ] TOP3のプレイヤー名が読める
- [ ] 伐採数が読める
- [ ] 3m離れた位置から読める

**4.5.3 ショップUI**
- [ ] アイテム名が読める
- [ ] 価格が読める
- [ ] ボタンテキストが読める

**UI改善が必要な場合**:
- WI-0017（HUDManager）に戻る
- フォントサイズを20%増加
- コントラスト強化（背景を暗く）

**確認項目**:
- [ ] 全UIが読みやすい（Quest実機）

---

#### 4.6 Network Sync Testing（ネットワーク同期テスト）

**前提**:
- 2人以上のプレイヤーが必要（自分 + 友人/テスターアカウント）

**操作**:
別のプレイヤーと同じインスタンスに参加

**テストケース**:

**4.6.1 木の状態同期**
1. プレイヤーAが木を叩く
2. プレイヤーBの画面で木のHPが減少するか確認
3. プレイヤーAが木を倒す
4. プレイヤーBの画面で倒木アニメーションが再生されるか確認

**確認**:
- [ ] 木の状態が同期される（1秒以内）

**4.6.2 丸太の同期**
1. プレイヤーAが丸太をPickup
2. プレイヤーBの画面で丸太が移動するか確認

**確認**:
- [ ] 丸太の位置が同期される

**4.6.3 ランキング同期**
1. プレイヤーAが木を10本伐採
2. プレイヤーBのリーダーボードを確認
3. プレイヤーAの伐採数が反映されているか確認

**確認**:
- [ ] ランキングが30秒以内に更新される

**4.6.4 協力伐採同期**
1. プレイヤーAとBが同じ木を叩く
2. 協力ボーナスが両者に発動するか確認

**確認**:
- [ ] 協力ボーナスが同期される

**確認項目**:
- [ ] ネットワーク同期が正常動作
- [ ] 同期遅延: 1秒以内

---

### ステップ5: Cross-Platform Compatibility Testing（クロスプラットフォームテスト）

#### 5.1 PC VR → Questインスタンス参加テスト

**操作**:
1. PC VRプレイヤーがワールドに参加（Questインスタンス）
2. Questプレイヤーと同じインスタンスに入る

**確認項目**:
- [ ] PC VRプレイヤーが正常にロードされる
- [ ] Questプレイヤーの動作が見える
- [ ] PC VRプレイヤーの動作がQuestプレイヤーに見える
- [ ] 協力伐採が正常動作

---

#### 5.2 Quest → PC VRインスタンス参加テスト

**操作**:
1. Questプレイヤーが、PC VRプレイヤーが作成したインスタンスに参加
2. 動作確認

**確認項目**:
- [ ] Questプレイヤーが正常にロードされる
- [ ] クロスプラットフォーム同期が正常動作

---

#### 5.3 Avatar Interaction Testing（アバター相互作用テスト）

**操作**:
異なるサイズのアバターでテスト

**テストケース**:
1. **小さいアバター**（0.5mスケール）で斧を拾う
2. **大きいアバター**（2.0mスケール）で斧を拾う
3. 正常にPickup/Drop可能か確認

**確認項目**:
- [ ] アバターサイズに関わらず斧が使える
- [ ] IK（Inverse Kinematics）が正常動作

---

### ステップ6: Load Testing（負荷テスト）

#### 6.1 Solo Testing（1人テスト）

**操作**:
1人でワールドに参加し、全機能を動作させる

**確認項目**:
- [ ] 1人でも全機能が動作
- [ ] FPS: 60fps以上（Quest 2）
- [ ] メモリ使用量: 正常範囲内

---

#### 6.2 Multi-Player Testing（5人テスト）

**操作**:
5人のプレイヤーでワールドに参加

**確認項目**:
- [ ] 5人同時接続でFPS維持（60fps以上）
- [ ] ネットワーク同期が正常
- [ ] 全員が協力伐採可能

---

#### 6.3 Stress Testing（10人ストレステスト）

**操作**:
10人のプレイヤーでワールドに参加（推奨キャパシティの上限）

**確認項目**:
- [ ] 10人同時接続でFPS維持（50fps以上）
- [ ] ネットワーク帯域: 5KB/秒以下（プレイヤーあたり）
- [ ] 重大なラグなし

**ネットワーク帯域確認方法**:
- VRChat Performance Stats → Network
- Send: < 5KB/s per player

---

#### 6.4 Persistence Testing（永続性テスト）

**操作**:
PlayerDataの永続性を確認

**テストケース**:

**6.4.1 Leave and Rejoin**
1. ワールドで木を10本伐採（XP、コイン獲得）
2. ワールドを退出
3. 再度ワールドに参加
4. HUDでXP、コイン残高を確認

**確認**:
- [ ] XP、コインが保持されている

**6.4.2 24時間後のテスト**
1. ワールドを退出
2. 24時間後に再参加
3. デイリーボーナス（20コイン）が付与されるか確認

**確認**:
- [ ] デイリーボーナスが正常に付与される

---

### ステップ7: Bug Reporting and Iteration（バグ報告と修正）

#### 7.1 Bug Tracking Sheet作成

**操作**:
発見されたバグを記録するトラッキングシートを作成

**Bug Tracking Sheetテンプレート**:

```markdown
# Bug Tracking Sheet - 森のきこりキャンプ Phase 1

## Critical Bugs（クリティカル：修正必須）
| Bug ID | Description | Steps to Reproduce | Expected | Actual | Module (WI) | Status |
|--------|-------------|-------------------|----------|--------|-------------|--------|
| BUG-001 | 例: ワールドクラッシュ | 1. 木を10本伐採 2. ... | 正常動作 | クラッシュ | WI-0012 | Open |

## High Priority Bugs（高優先度：修正推奨）
| Bug ID | Description | Steps to Reproduce | Expected | Actual | Module (WI) | Status |
|--------|-------------|-------------------|----------|--------|-------------|--------|
| BUG-101 | 例: FPS低下 | 1. 10人接続 2. ... | 60fps | 40fps | WI-0024 | Open |

## Medium Priority Bugs（中優先度：修正検討）
| Bug ID | Description | Steps to Reproduce | Expected | Actual | Module (WI) | Status |
|--------|-------------|-------------------|----------|--------|-------------|--------|
| BUG-201 | 例: UI表示ずれ | 1. ショップ開く 2. ... | 中央表示 | 左寄り | WI-0019 | Open |

## Low Priority Bugs（低優先度：次フェーズで対応）
| Bug ID | Description | Steps to Reproduce | Expected | Actual | Module (WI) | Status |
|--------|-------------|-------------------|----------|--------|-------------|--------|
| BUG-301 | 例: サウンド遅延 | 1. 木を叩く 2. ... | 即座 | 0.5秒遅延 | WI-0011 | Open |
```

**確認項目**:
- [ ] Bug Tracking Sheet作成完了
- [ ] 発見されたバグをすべて記録

---

#### 7.2 Critical Bug Fix Workflow（クリティカルバグ修正フロー）

**操作**:
クリティカルバグ（ワールドクラッシュ、進行不能など）が見つかった場合

**修正手順**:
1. Bug IDを付与（例: BUG-001）
2. 該当するWI（Work Instruction）を特定
3. Unity Editorで該当WIのスクリプトを開く
4. バグを修正
5. ClientSimでローカルテスト
6. ステップ3.2に戻ってQuest再ビルド
7. Quest実機で再テスト
8. Bug Tracking Sheetを更新（Status: Fixed）

**確認項目**:
- [ ] クリティカルバグがすべて修正済み
- [ ] 修正後に新たなバグが発生していない（リグレッションテスト）

---

#### 7.3 Known Issues Documentation（既知のバグドキュメント化）

**操作**:
修正しない非クリティカルバグをドキュメント化

**Known Issues例**:
```markdown
# Known Issues - Phase 1

## Minor Issues（軽微な問題）
1. **サウンド遅延** (BUG-301)
   - 症状: 木を叩いた時のサウンドが0.5秒遅延
   - 影響: 低（ゲームプレイに支障なし）
   - 対応: Phase 2で修正予定

2. **UI表示ずれ** (BUG-201)
   - 症状: ショップUIが左寄りに表示される
   - 影響: 低（操作は可能）
   - 対応: Phase 2で修正予定
```

**確認項目**:
- [ ] Known Issuesドキュメント作成完了
- [ ] 非クリティカルバグ < 3件

---

### ステップ8: Release Preparation（リリース準備）

#### 8.1 World Listing Preparation（ワールドリスティング準備）

**操作**:
VRChatワールドリスティング（公開ページ）の最終確認

**確認項目**:

**8.1.1 World Name**
- [ ] 日本語: 森のきこりキャンプ
- [ ] 英語: Forest Woodcutter Camp

**8.1.2 Description**
- [ ] 日英両方記載
- [ ] 主要機能説明（伐採、運搬、スキル成長）
- [ ] Quest対応明記

**8.1.3 Tags**
- [ ] Game
- [ ] Quest Compatible
- [ ] PvE
- [ ] Chill/Relaxing
- [ ] Japanese

**8.1.4 Thumbnail Image**
- [ ] 解像度: 1200x900 PNG
- [ ] 内容: キャンプ全景と木を切っているプレイヤー
- [ ] 明るく魅力的な構図

**8.1.5 Screenshots**
- [ ] 3〜5枚のゲームプレイスクリーンショット
- 推奨内容:
  1. キャンプ全景
  2. 木を伐採しているシーン
  3. 協力伐採（2人以上）
  4. HUD表示（スキル成長）
  5. ショップUI

---

#### 8.2 Documentation Update（ドキュメント更新）

**操作**:
プロジェクトドキュメントを最終更新

**更新対象ドキュメント**:

**8.2.1 README.md（プロジェクトルート）**
```markdown
# 森のきこりキャンプ / Forest Woodcutter Camp

## Phase 1 Release Notes

**Version**: 1.0.0
**Release Date**: 2025-11-17
**Platform**: VRChat (PC VR, Desktop, Quest 2/3/Pro)

### Features
- **Woodcutting System**: Chop trees, earn XP and Coins
- **Skill Progression**: Level up Woodcutting skill (Lv1-10)
- **Economy System**: Earn coins, buy better axes
- **Cooperative Chopping**: +20% bonus when chopping together
- **Leaderboard**: TOP3 daily rankings

### Technical Specs
- Unity 2022.3.22f1
- VRChat SDK 3.9.0
- UdonSharp 1.1.9
- Quest Optimized (60fps on Quest 2)

### Getting Started
1. Visit "森のきこりキャンプ" in VRChat Worlds
2. Grab an axe near the campfire
3. Find a tree and start chopping!

### Known Issues
- [List known minor bugs here]

### Credits
- Developer: [Your Name]
- Music: [Credit if any]
- Assets: [Credit if any]
```

**確認項目**:
- [ ] README.md更新完了

**8.2.2 User Manual（ユーザーマニュアル）**
```markdown
# User Manual - 森のきこりキャンプ

## How to Play

### 1. Grab an Axe
- Desktop: Press "E" key
- VR: Grip button

### 2. Chop Trees
- Desktop: Left click
- VR: Trigger button

### 3. Carry Logs
- Grab logs from fallen trees
- Carry to delivery zone (yellow area)
- Drop to earn coins

### 4. Level Up Skills
- Chop trees to earn XP
- Level up Woodcutting skill
- Unlock damage bonuses

### 5. Buy Better Axes
- Visit the shop (merchant tent)
- Spend coins to buy better axes
- Chop faster and earn more!

### Tips
- Chop trees with others for +20% bonus!
- Daily login bonus: 20 coins
- Max level: Lv10 Woodcutting Master
```

**確認項目**:
- [ ] User Manual作成完了

**8.2.3 Developer Notes（開発者ノート）**
```markdown
# Developer Notes - Architecture Summary

## Module Structure
- M-01 GameManager: Singleton initialization
- M-02 NetworkSync: Manual sync batching
- M-03 PersistenceManager: PlayerData I/O
- M-04 SkillManager: XP and leveling
- M-05 SkillEffects: Damage calculations
- M-06 CoinManager: Economy system
- M-07 ShopManager: Item purchases
- M-08 HUDManager: UI display
- M-09 LeaderboardUI: Rankings
- M-10 TreeController: Tree state machine
- M-11 AxeInteraction: Damage and effects
- M-12 LogSpawner: Object pooling
- M-13 LogPickup: VRC_Pickup system
- M-14 CuttingTable: Processing system
- M-15 CooperativeTracker: Cooperative bonuses
- M-16 RankingTracker: Statistics tracking

## Performance Metrics
- Quest 2 FPS: 60fps (10 players)
- Network Bandwidth: < 5KB/s per player
- PlayerData Size: ~500 bytes
- Performance Rank: "Good"

## Future Improvements (Phase 2)
- Combat System (Bear attacks)
- Camp Rank (instance-wide progression)
- Crafting recipes
- Achievements
```

**確認項目**:
- [ ] Developer Notes作成完了

---

## 9. 完了条件（Doneの条件）

以下の**すべての項目**が満たされた場合、この作業は完了とする:

### 必須条件（すべて達成必須）

#### ビルド成功
- [ ] Androidプラットフォームへの切り替え完了
- [ ] VRChat SDK Validation: All Passed
- [ ] Questビルド成功（エラーなし）
- [ ] VRChatサーバーへのアップロード成功

#### Quest実機テスト
- [ ] Quest 2/3実機でワールド起動成功
- [ ] FPS: 60fps以上（Quest 2、1人時）
- [ ] Performance Rank: "Good"以上
- [ ] 全25機能（WI-0001〜0024）が正常動作
- [ ] クリティカルバグ: 0件

#### ネットワークテスト
- [ ] 2人以上でネットワーク同期テスト完了
- [ ] 協力伐採ボーナスが正常動作
- [ ] ランキング同期が正常動作
- [ ] ネットワーク帯域: 5KB/秒以下（プレイヤーあたり）

#### クロスプラットフォームテスト
- [ ] PC VR ⇔ Quest相互参加テスト完了
- [ ] 異なるアバターサイズでテスト完了

#### 負荷テスト
- [ ] 1人テスト完了
- [ ] 5人テスト完了（FPS維持）
- [ ] 10人ストレステスト完了（FPS 50fps以上）

#### 永続性テスト
- [ ] PlayerData保存・復元テスト完了
- [ ] デイリーボーナステスト完了

### 品質条件

#### パフォーマンス
- [ ] Quest 2: 60fps以上（10人接続時も50fps以上）
- [ ] Quest 3: 72fps以上
- [ ] Frame Time: 16.6ms以下
- [ ] メモリ使用量: 100MB以下

#### 安定性
- [ ] クリティカルバグ: 0件
- [ ] 既知の非クリティカルバグ: 3件以内
- [ ] 5分間プレイでクラッシュなし

#### 互換性
- [ ] Quest 2/3/Pro対応
- [ ] PC VR対応
- [ ] Desktopモード対応

### ドキュメント条件

- [ ] Bug Tracking Sheet作成完了
- [ ] Known Issuesドキュメント作成完了
- [ ] README.md更新完了
- [ ] User Manual作成完了
- [ ] Developer Notes作成完了
- [ ] この作業指示書が完了報告として更新されている（完了日時記載）

---

## 10. テスト観点・テストケース

### テスト種別
- **統合テスト**: 全モジュール連携動作確認
- **パフォーマンステスト**: FPS、Frame Time測定
- **ネットワークテスト**: マルチプレイヤー同期確認
- **クロスプラットフォームテスト**: PC VR ⇔ Quest互換性
- **負荷テスト**: 10人同時接続
- **永続性テスト**: PlayerData保存・復元

### テストケース一覧（再掲）

#### TC-001: 基本ゲームループ
**手順**:
1. ワールド参加
2. 斧を拾う
3. 木を伐採（HP = 0まで）
4. 丸太をPickup
5. 納品ゾーンへDrop
6. HUDでXP、コイン増加確認

**期待結果**:
- [ ] 全手順が正常動作
- [ ] XP: +10（小さい木）
- [ ] コイン: +3（伐採）+ 3〜30（運搬距離）

---

#### TC-002: スキル成長
**手順**:
1. 木を10本伐採（XP: 10 × 10 = 100）
2. HUDでレベルアップ確認（Lv1 → Lv2）

**期待結果**:
- [ ] レベルアップ演出表示
- [ ] ダメージボーナス: +5%

---

#### TC-003: ショップ購入
**手順**:
1. コインを100枚集める
2. ショップで鉄の斧購入
3. 装備

**期待結果**:
- [ ] コイン: -100
- [ ] 鉄の斧装備完了
- [ ] ダメージ: +50%

---

#### TC-004: 協力伐採
**手順**（2人必要）:
1. プレイヤーAとBが同じ木を叩く（3秒以内）
2. 協力ボーナス発動確認

**期待結果**:
- [ ] 金色エフェクト表示
- [ ] ダメージ: +20%
- [ ] XP: +20%

---

#### TC-005: ランキング
**手順**:
1. 木を10本伐採
2. リーダーボードを確認
3. 自分の名前とスコアが表示されるか確認

**期待結果**:
- [ ] TOP3に自分の名前
- [ ] 伐採数: 10本

---

#### TC-006: 永続性
**手順**:
1. 木を5本伐採（XP: 50、コイン: 15）
2. ワールド退出
3. 再参加
4. HUD確認

**期待結果**:
- [ ] XP: 50（保持）
- [ ] コイン: 15（保持）

---

#### TC-007: Quest性能
**手順**:
1. Quest 2でワールド参加
2. Performance Stats表示
3. 5分間プレイ

**期待結果**:
- [ ] FPS: 60fps以上（平均）
- [ ] Frame Time: 16.6ms以下
- [ ] Performance Rank: "Good"以上

---

#### TC-008: ネットワーク同期
**手順**（2人必要）:
1. プレイヤーAが木を伐採
2. プレイヤーBの画面確認

**期待結果**:
- [ ] 木の状態が同期（1秒以内）
- [ ] 倒木アニメーションが両者で再生

---

## 11. 成果物

### 作成ファイル
1. **Questビルド済みワールド**
   - VRChatサーバーにアップロード完了
   - World ID: `wrld_xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`

2. **Bug Tracking Sheet**
   - パス: `docs/WID/assets/Bug_Tracking_Sheet.md`
   - 内容: 発見されたバグの一覧

3. **Known Issues Document**
   - パス: `docs/WID/assets/Known_Issues.md`
   - 内容: 非クリティカルバグの一覧

### 更新ファイル
1. **README.md**
   - パス: `/README.md`
   - 内容: Phase 1リリースノート追加

2. **User Manual**
   - パス: `docs/User_Manual.md`
   - 内容: ゲームプレイガイド

3. **Developer Notes**
   - パス: `docs/Developer_Notes.md`
   - 内容: アーキテクチャサマリ

### テストレポート
- この作業指示書にテスト結果を記載（完了報告欄）

---

## 12. 備考

### 注意点

#### Quest最適化の重要性
- Questは PC VRより性能が低いため、最適化が重要
- WI-0024（Performance Optimization）が完了していることが前提
- 60fps維持が最優先（ユーザー体験に直結）

#### ネットワーク帯域制限
- VRChatのネットワーク帯域制限: 10KB/秒
- 本プロジェクト目標: 5KB/秒以下（50%使用）
- Manual Sync活用で帯域削減（Continuous Sync回避）

#### PlayerData容量制限
- VRChat PlayerData制限: 100KB/Player/World
- 本プロジェクト使用量: 約500バイト（0.5%）
- 余裕があるため、Phase 2でデータ追加可能

#### Late Joiner対応
- WI-0004（Late Joiner Handling）が正常動作していることが前提
- 後から参加したプレイヤーも同期データを受信

### 次のステップ

この作業完了後、以下のアクションを実行:

#### Phase 1リリース
1. **ワールド公開設定変更**
   - VRChat SDK Control Panel → Content Manager
   - Visibility: "Public"に変更
   - Release Notes記載

2. **コミュニティ告知**
   - VRChat公式Discord
   - Twitter/X
   - VRChat World Forumに投稿

#### Phase 2準備
1. **フィードバック収集**
   - 初週のプレイヤーフィードバックを収集
   - Bug報告を受け付け

2. **Phase 2機能設計**
   - Combat System（熊襲撃イベント）
   - Camp Rank（インスタンス共有成長）
   - Carrying/Craftingスキル追加

### トラブルシューティング

#### 問題1: Androidプラットフォーム切り替えに失敗
**症状**: "Switch Platform"が進まない

**原因**: Android Build Supportがインストールされていない

**解決法**:
1. Unity Hub → Installs → Unity 2022.3.22f1 → 歯車アイコン → Add Modules
2. "Android Build Support"にチェック
3. インストール完了後に再試行

---

#### 問題2: Questビルドに失敗（Shader Error）
**症状**: "Shader 'Standard' not supported on Quest"

**原因**: Quest非対応のShaderを使用

**解決法**:
1. ステップ1.3に戻る
2. 全マテリアルをMobile/Diffuseに変更
3. 再ビルド

---

#### 問題3: Quest実機でFPS低下（30fps以下）
**症状**: Quest 2で30fps前後

**原因**: ポリゴン数過多、LOD未設定

**解決法**:
1. WI-0024（Performance Optimization）に戻る
2. ポリゴン数を50%削減（LOD設定）
3. テクスチャを512x512に縮小
4. 再ビルド・再テスト

---

#### 問題4: ネットワーク同期が遅い（5秒以上）
**症状**: プレイヤー間で状態同期に5秒以上かかる

**原因**: Manual Syncの頻度が低すぎる

**解決法**:
1. WI-0002（NetworkSync Batching）に戻る
2. 同期頻度を2秒間隔に変更（現在5秒の場合）
3. 再テスト

---

#### 問題5: PlayerDataが保存されない
**症状**: ワールド再参加時にXP、コインがリセット

**原因**: PersistenceManagerのSave処理が呼ばれていない

**解決法**:
1. WI-0003（PersistenceManager）に戻る
2. `_SavePlayerData()`メソッドの呼び出しを確認
3. OnPlayerLeft時に保存処理を追加

---

#### 問題6: VRChatアップロードタイムアウト
**症状**: "Upload timeout"エラー

**原因**: ネットワーク不安定、ファイルサイズ過大

**解決法**:
1. インターネット接続を確認
2. ワールドファイルサイズを確認（100MB以下推奨）
3. テクスチャ圧縮でサイズ削減
4. 再アップロード

---

### 参考資料

#### VRChat公式ドキュメント
- **VRChat SDK 3.9.0**: https://creators.vrchat.com/releases/release-3-9-0/
- **Quest Content Optimization**: https://creators.vrchat.com/platforms/android/quest-content-optimization/
- **World Debugging**: https://creators.vrchat.com/worlds/debugging/

#### プロジェクト内ドキュメント
- `docs/TDD.md` - モジュール設計詳細
- `docs/PRDv3.md` - Phase 1詳細仕様
- `docs/FSD.md` - 機能仕様書
- `docs/ref/VRChat_Tools_API_Reference.md` - VRChat API詳細

#### 外部リソース
- **Unity 2022.3 LTS Documentation**: https://docs.unity3d.com/2022.3/
- **UdonSharp Documentation**: https://udonsharp.docs.vrchat.com/
- **Quest Development Guide**: https://developer.oculus.com/documentation/unity/

---

## テスト結果記録欄

### Quest 2実機テスト結果

**実施日時**: _______________
**テスター**: _______________
**デバイス**: Quest 2

#### パフォーマンス測定
- [ ] 静止時FPS: ______ fps
- [ ] 木伐採時FPS: ______ fps
- [ ] 10人接続時FPS: ______ fps
- [ ] 平均Frame Time: ______ ms
- [ ] Performance Rank: ______

#### 機能テスト
- [ ] WI-0001〜0024全機能動作: 合格/不合格
- [ ] クリティカルバグ: _____ 件
- [ ] 非クリティカルバグ: _____ 件

#### 備考
```
[テスト中に気づいた点を記載]
```

---

### Quest 3実機テスト結果

**実施日時**: _______________
**テスター**: _______________
**デバイス**: Quest 3

#### パフォーマンス測定
- [ ] 静止時FPS: ______ fps
- [ ] 木伐採時FPS: ______ fps
- [ ] 10人接続時FPS: ______ fps
- [ ] 平均Frame Time: ______ ms
- [ ] Performance Rank: ______

#### 備考
```
[テスト中に気づいた点を記載]
```

---

### クロスプラットフォームテスト結果

**実施日時**: _______________
**テスター**: _______________

#### PC VR → Quest
- [ ] 相互参加: 成功/失敗
- [ ] 同期動作: 正常/異常

#### Quest → PC VR
- [ ] 相互参加: 成功/失敗
- [ ] 同期動作: 正常/異常

---

### 負荷テスト結果

**実施日時**: _______________

#### 1人テスト
- [ ] FPS: ______ fps
- [ ] 結果: 合格/不合格

#### 5人テスト
- [ ] FPS: ______ fps
- [ ] 結果: 合格/不合格

#### 10人ストレステスト
- [ ] FPS: ______ fps
- [ ] ネットワーク帯域: ______ KB/s/player
- [ ] 結果: 合格/不合格

---

**作業完了報告欄**

- [ ] 作業完了日時: __________________
- [ ] 作業者: __________________
- [ ] レビュアー: __________________
- [ ] レビュー完了日時: __________________
- [ ] リリース承認: __________________

---

**文書管理情報**
- 作成日: 2025-11-17
- バージョン: 1.0
- ステータス: 実装準備完了
- 次回レビュー: テスト完了後
