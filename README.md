# 森のきこりキャンプ - Woodcutter's Forest Camp

**VRChatワールド開発プロジェクト** - Valheimの爽快な木こり体験をVRで再現し、みんなで協力しながら会話を楽しむ新感覚ワールド

[![VRChat SDK](https://img.shields.io/badge/VRChat_SDK-3.9.0-blue)](https://vrchat.com/)
[![Unity](https://img.shields.io/badge/Unity-2022.3.22f1-green)](https://unity.com/)
[![UdonSharp](https://img.shields.io/badge/UdonSharp-1.1.9-orange)](https://github.com/vrchat-community/UdonSharp)
[![Platform](https://img.shields.io/badge/Platform-PC%20%7C%20Quest-purple)](https://vrchat.com/)

## プロジェクト概要

「森のきこりキャンプ」は、VRChatで体験できる協力作業型ワールドです。斧を振って木を倒し、丸太を運び、板材に加工する...シンプルな作業を通じて自然と会話が生まれ、プレイヤー同士の繋がりが深まります。

### なぜこのプロジェクトが面白いのか

**1. Valheimライクな気持ちよさ**
- 斧で木を叩く → 倒れる → 丸太が出る、というシンプルで爽快なゲームサイクル
- 物理演算による倒木の迫力ある演出
- 木のHPが減るたびに「ミシミシ」ときしむリアルな音響

**2. 協力プレイの必然性**
- 2人以上で同じ木を切ると「協力伐採ボーナス」発動（ダメージ+20%、XP+20%）
- 太い木は協力しないと効率が悪い設計
- 運搬や加工中の待ち時間が自然な会話タイムに

**3. 成長と収集の楽しみ**
- Woodcuttingスキル（Lv1〜10）で徐々に強くなる実感
- きこりコインで購入できる6種類の斧スキン
- デイリーランキングで「今日の伐採王」を競う
- レベル10到達で特殊エフェクトとマスター称号

**4. まったり雑談しながら作業**
- 平均滞在時間20〜30分の程よい体験
- 作業の合間に焚き火を囲んで休憩
- ステータスアイコンで他プレイヤーの活動が一目瞭然

## ゲームプレイ動画（Coming Soon）

[プレイ動画のサムネイル画像]
*開発中のプレイ動画を近日公開予定*

## 主要機能

### 伐採システム
- **3種類の木**: 小さい木（Pine）、中くらいの木（Oak）、大きな木（Ancient）
- **斧のアップグレード**: 木の斧 → 鉄の斧 → 黒鋼の斧
- **協力伐採**: 複数人で同じ木を切ると金色のエフェクトとボーナス発動
- **倒木演出**: Valheim風の物理演算による迫力ある倒木

### スキル成長システム
```
Woodcutting スキル
├── Lv1: 初心者きこり
├── Lv3: 丸太生成数+1（10%確率）
├── Lv5: クリティカル率5%、鉄の斧解禁
├── Lv8: クリティカル率10%
└── Lv10: マスターきこり称号、特殊エフェクト
```

### 経済システム
- **きこりコイン**: 伐採・運搬・加工でコイン獲得
- **ショップ**: 斧スキン、作業帽、運搬ベルトなどを購入
- **デイリーボーナス**: 毎日ログインで20コイン

### ソーシャル機能
- **デイリーランキング**: 「本日の伐採王TOP3」を競う
- **ステータスアイコン**: 作業中のプレイヤーの状態を可視化
- **エモート**: 手を振る、拍手、ガッツポーズなど

## 開発への参加方法

### 必要な環境
- Unity 2022.3.22f1
- VRChat Creator Companion
- VRChat SDK 3.9.0
- UdonSharp 1.1.9
- Git

### セットアップ手順

1. **リポジトリをクローン**
```bash
git clone https://github.com/[your-username]/woodcutter-forest-camp.git
cd woodcutter-forest-camp
```

2. **Unityでプロジェクトを開く**
```
Unity Hub → Add → クローンしたフォルダを選択
Unity 2022.3.22f1で開く
```

3. **VRChat Creator Companionでパッケージをインポート**
```
VCC起動 → プロジェクトを追加 → SDK/UdonSharpを最新に更新
```

4. **シーンを開く**
```
Assets/Scenes/WoodcutterCamp.unity を開く
```

### 開発で協力してほしいこと

#### すぐに参加できるタスク

**初心者歓迎タスク**
- [ ] 効果音の追加・改善（斧ヒット音、倒木音など）
- [ ] パーティクルエフェクトの調整（木片、葉っぱ、土煙）
- [ ] チュートリアル看板のテキスト改善
- [ ] エモートアニメーションの追加

**中級者向けタスク**
- [ ] 新しい斧スキンのモデリング
- [ ] ランキングボードのUI改善
- [ ] 板材を使った装飾アイテムの追加（椅子、テーブルなど）
- [ ] Quest 2向けのパフォーマンス最適化

**上級者向けタスク**
- [ ] UdonSharpスクリプトの最適化（Update回避、イベント駆動化）
- [ ] ネットワーク同期の改善（帯域幅削減）
- [ ] PlayerData永続化システムの拡張
- [ ] Phase 2機能の設計・実装（熊襲撃システムなど）

### コントリビューションガイドライン

1. **Issue作成**: バグ報告や機能提案は[Issues](https://github.com/[your-username]/woodcutter-forest-camp/issues)へ
2. **ブランチ作成**: `feature/your-feature-name` or `fix/bug-description`
3. **コミットメッセージ**: 日本語OK、わかりやすく記述
4. **プルリクエスト**: mainブランチへPR作成、レビューを待つ

## 技術仕様

### ドキュメント構成

このリポジトリの設計・実装方針は `docs/` 以下のドキュメントで管理しています。  
開発を進める前に、少なくとも PRD / FSD / TDD / TSD には一度目を通してください。

- [`docs/PLAN.md`](docs/PLAN.md)
  - プロジェクト全体のコンセプト・体験ゴール・マップ構成・MVP/拡張ロードマップをまとめた企画レベルの計画書です。
  - 「森のきこりキャンプ」がどんなワールドになるのかを短時間で把握したいときに参照します。

- [`docs/PRD.md`](docs/PRD.md)
  - Phase 1 の **プロダクト要件定義書 (Product Requirements Document)**。ターゲット、KPI、ゲームデザインの詳細（木・斧・XP・コインテーブルなど）を定義しています。
  - 実装に迷ったときは「プレイヤーにどんな体験を提供したいか」をここで確認します。

- [`docs/FSD.md`](docs/FSD.md)
  - 機能仕様書 (Functional Specification Document)。
    FNC-001〜006（伐採・運搬・加工・成長・経済・協力）の各機能について、振る舞い・シナリオ・正常系/異常系を具体的に記述しています。
  - UdonSharp 実装前に「この機能は何をすべきか？」を確認するための一次情報です。

- [`docs/TSD.md`](docs/TSD.md)
  - 技術仕様書 (Technical Specification Document)。
    UdonSharp / PlayerData / UdonSynced を前提としたアーキテクチャ、データモデル、同期戦略、パフォーマンス要件を定義しています。
  - 新しいスクリプトやネットワーク同期を追加する際は、ここに沿って設計してください。

- [`docs/TDD.md`](docs/TDD.md)
  - Technical Design Document。
    モジュール (M-01〜) と機能 (F-xx) を実装単位にブレイクダウンし、Work Instruction（作業指示書）に落とし込むための中間設計です。
  - 「どのモジュールがどの責務を持つか」「どのWIを先に実装すべきか」を確認する場です。

- [`docs/REF.md`](docs/REF.md)
  - VRChat SDK / UdonSharp /周辺ツールのリファレンスマニュアル。
    プロジェクト内でよく使う API やツールのリンク・要約をまとめています。
  - 公式ドキュメントを毎回検索する手間を減らすためのカタログとして使います。

- [`docs/TECHNICAL_DETAILS.md`](docs/TECHNICAL_DETAILS.md) / [`docs/VRChat_TechResearch_2025.md`](docs/VRChat_TechResearch_2025.md) / [`docs/VRChat_Tools_API_Reference.md`](docs/VRChat_Tools_API_Reference.md)
  - VRChat・UdonSharp の最新仕様・制約・ベストプラクティスの調査結果。
  - SDK や UdonSharp のバージョンアップを検討する際や、「この構文は UdonSharp で使えるか？」を確認したいときに参照します。

#### Work Instruction ([`docs/WID/`](docs/WID/))

実際の Unity 作業は、[`docs/WID/`](docs/WID/) 以下の **作業指示書 (Work Instruction / WI)** に分解されています。
各ファイルは 1 つのタスク（1〜2日分）に対応し、「何を・どの順番で・どう確認するか」が具体的に書かれています。

- [`docs/WID/WI-0001_GameManager.md`](docs/WID/WI-0001_GameManager.md)
  - **M-01 GameManager** の実装手順書。
    `GameManager.cs` の作成、プレハブ化、シーン配置、ログ確認までの手順と Done 条件が定義されています。
  - Phase 0（基盤実装）の最初のタスクであり、他のモジュール（NetworkSync, PersistenceManager など）はこの GameManager を前提に作られます。

全25個の作業指示書（WI-0001〜WI-0025）が Phase 0〜3 に分けて定義されています。
**Unity 上で作業を始める前に、必ず該当 WI を開いて「前提条件・手順・完了条件」を確認する**運用を想定しています。

### アーキテクチャ

``` 
WoodcutterCamp/
├── Core/                    # 基盤システム
│   ├── GameManager         # ゲーム全体管理
│   ├── NetworkSync         # UdonSynced抽象化
│   └── PersistenceManager  # PlayerData管理
├── Gameplay/               # ゲームロジック
│   ├── Woodcutting/       # 伐採システム
│   ├── Carrying/          # 運搬システム
│   └── Processing/        # 加工システム
└── UI/                    # ユーザーインターフェース
```

### パフォーマンス目標
- **Quest 2**: 60fps維持、メモリ使用量100MB以下
- **同時接続**: 最大20人、推奨4〜8人
- **ネットワーク**: 帯域幅10KB/秒以下

### SOLID原則による設計
- 単一責任原則: 各モジュールは単一の機能に特化
- オープン/クローズド原則: 新機能追加時の既存コード変更最小化
- 依存性逆転原則: 抽象化レイヤーによる疎結合

## 開発ロードマップ

### Phase 1 (現在) - 基本システム
- [x] 伐採システム実装
- [x] 運搬システム実装
- [x] 加工システム実装
- [x] スキル成長システム
- [x] 経済システム
- [x] ソーシャル機能
- [ ] バグ修正とバランス調整
- [ ] ワールド公開

### Phase 2 - 拡張機能
- [ ] 熊襲撃イベントシステム
- [ ] Camp Rank（拠点レベル）
- [ ] 太丸太協力運搬
- [ ] 高度なクラフトレシピ
- [ ] 個人キャンプエリア

### Phase 3 - コミュニティ機能
- [ ] グローバルランキング
- [ ] シーズンイベント
- [ ] プレイヤー間トレード
- [ ] ギルドシステム

## コミュニティ

### Discord サーバー
開発者とプレイヤーが集まるコミュニティ: [参加リンク]（準備中）

### 定期プレイテスト
毎週土曜日 21:00 JST - みんなで一緒に木を切ろう！

### 開発ブログ
開発の裏話や技術解説: [ブログリンク]（準備中）

## よくある質問

**Q: VRヘッドセットがなくても遊べますか？**
A: はい！デスクトップモードに完全対応しています。マウスとキーボードで快適にプレイできます。

**Q: Quest 2でも動きますか？**
A: もちろんです！Quest 2で60fps維持できるよう最適化しています。

**Q: プログラミング経験がなくても貢献できますか？**
A: 大歓迎です！3Dモデル作成、効果音制作、アイデア提案、テストプレイなど、様々な形で貢献できます。

**Q: 一人でも楽しめますか？**
A: 一人でもスキル成長やコイン収集を楽しめますが、協力プレイでより楽しくなる設計です。

## ライセンス

このプロジェクトは [MIT License](LICENSE) のもとで公開されています。

## 謝辞

- VRChat コミュニティの皆様
- UdonSharp 開発チーム
- Valheim、Palia、Deep Rock Galactic の開発者様（インスピレーションをありがとう）
- すべてのコントリビューターとテスター

---

**さあ、一緒に最高のきこりキャンプを作りましょう！**

斧を手に取り、木を倒し、仲間と語らう。シンプルだけど奥深い、みんなで作り上げるVRChat ワールド。
あなたの参加をお待ちしています。

[ワールドへ行く（公開後にリンク追加）] | [Issueを作成](https://github.com/[your-username]/woodcutter-forest-camp/issues/new) | [Discordに参加（準備中）]
