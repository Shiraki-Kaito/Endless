# Endless

Unreal Engine 5.6で制作した、部屋を進みながら敵と戦う三人称シューティングゲームです。ドアの先に部屋を読み込み、武器と弾薬を管理しながら探索します。

![開始地点の通路](Docs/Images/start-corridor.jpg)

## ゲームの特徴

- 開始通路と14種類の部屋。平屋、L字・T字、上階・地下につながる部屋を組み合わせます。
- ドアに近づくと次の部屋を先読みし、接続位置・向きを合わせて生成します。前・現在・次の最大3部屋を保持します。
- 敵は巡回、視認・音への反応、追跡、射撃を行います。部屋ごとの活動制御を備えています。
- 武器の拾得・切替、弾薬、時間経過によるリロード、所持上限3枠と交換選択を実装しています。
- Blueprintのキャラクター・UI・アセットと、C++の部屋管理・戦闘処理を組み合わせています。

上記は実装内容です。今回実施した確認と未検証の範囲は、[最終整理・検証記録](Docs/FinalizationReport.md)に分けて記載しています。

| 部屋の探索 | 高低差のある部屋 |
|---|---|
| ![部屋04](Docs/Images/room-04.jpg) | ![上階につながる部屋11](Docs/Images/room-11.jpg) |

![敵との戦闘](Docs/Images/combat.jpg)

画像はこのプロジェクトをUE5.6のPIEで動かして撮影したものです。プレイ画面からEditor部分を切り取った画像と、階段を見渡す紹介用カメラの画像を掲載しています。画像生成やゲーム内容の合成は行っていません。撮影時はDirectX 11を使用しています。

## 操作（キーボード・マウス）

| 操作 | キー |
|---|---|
| 移動 | W / A / S / D、矢印キー |
| 視点移動 | マウス |
| 射撃 | 左クリック |
| 照準 | 右クリック |
| リロード | R |
| ドアの操作・武器拾得・交換確定 | E |
| 武器切替・交換枠の選択 | マウスホイール |
| しゃがむ | C |
| ジャンプ | Space |

武器が3枠埋まっている場合は、Eで交換選択に入り、ホイールで枠を選んでEで確定します。対象から離れると交換を取り消します。入力定義は `Content/TacticalSurvive/Input` にあります。ゲームパッドの割当もありますが、実機での全操作確認は今回行っていません。

## UE5.6で開く

Windows、Unreal Engine **5.6**、C++ビルド環境（Visual Studio 2022のC++ゲーム開発ワークロードとWindows SDK）、Git LFSが必要です。確認に使用したEngineは5.6.1です。

```powershell
git clone --branch feature/room-system-foundation https://github.com/kait0-s/Endless.git
Set-Location Endless
git lfs install
git lfs pull
```

`Endless.uproject` を開き、必要ならC++モジュールをビルドします。起動マップは次のとおりです。

```text
/Game/Endless/Rooms/Start/Lvl_StartCorridor
```

Editorの「プレイ」から開始できます。各部屋は `Content/Endless/Rooms/Room/Lvl_Room_01.umap` ～ `Lvl_Room_14.umap` です。開始通路から起動すると、通常の部屋生成・進行を確認できます。

通常のビルドコマンド（Engineのインストール先は環境に合わせて変更）：

```powershell
$engineRoot = 'C:\Program Files\Epic Games\UE_5.6'
$projectFile = (Resolve-Path .\Endless.uproject).Path
& "$engineRoot\Engine\Build\BatchFiles\Build.bat" EndlessEditor Win64 Development "-Project=$projectFile" -WaitMutex
& "$engineRoot\Engine\Build\BatchFiles\Build.bat" Endless Win64 Development "-Project=$projectFile" -WaitMutex
```

`.uproject` で有効にしているプラグインは **GameplayStateTree** と **ModelingToolsEditorMode**（Editorのみ）です。Enhanced InputなどのEngine機能も使用します。検証スクリプトを使う場合はEngine付属のPython Editor Script Pluginが必要です。Fabの依存アセットはContent内のファイルを使用します。

`.uasset`、`.umap`、掲載画像はGit LFSの対象です。LFSポインターだけの状態では起動できません。`Binaries`、`Intermediate`、`Saved`、`DerivedDataCache`、`Backups`、`plus` はコミット対象に含めません。

## 外部アセット

アセットの権利は各提供者に帰属します。以下はContent内で確認したパッケージ・フォルダー名です。作者名・商品URL・ライセンス原本が確認できないものを推測で補っていません。再配布条件は元の取得元・ライセンスで確認してください。

| 提供元・パッケージ識別名 | 使用内容 |
|---|---|
| Epic Games / Third Person・Mannequins・Starter Content | キャラクター、アニメーション、基礎素材 |
| `Fab/Low_Poly_Weapons_VOL_1`、`HS_WeaponPack_03`、`Laser` | 武器・関連素材 |
| `FreeFurniturePack`、`RestaurantScene` | 家具・室内素材 |
| `Scene_Warehouse`、`Warehouse_Storage` | 倉庫・工業系の素材 |
| `LPLab`、`CR` | 室内小物・キャラクター関連素材 |

`TacticalSurvive` は既存のゲーム実装・アセット群として引き継いでいます。開発メンバーの氏名と担当、各外部パッケージの作者情報は、リポジトリ内で確定できる資料がないため掲載していません。

## 制限と検証状況

一部のPIE終了時クラッシュ、Room 11〜14の生成用メッシュ警告、ゲームに使用しない既存デモ4パッケージの読み込み警告が残っています。標準ビルドとGitHubへの公開は今回の実行環境の書き込み・接続制限を受けており、すべての検証に合格したリリースとは扱っていません。

最終整理の詳細、性能計測条件、ビルド方法、未検証事項は[検証記録](Docs/FinalizationReport.md)を参照してください。DirectX 11での結果をDirectX 12・Lumen・Virtual Shadow Mapsの結果として扱っていません。製品版パッケージと別PCでの再現確認は、Editor内での動作確認とは別の検証です。
