# 最終整理・検証記録（2026-10-27）

## 公開状況

ブランチは `feature/room-system-foundation`、接続先は `origin`（`https://github.com/kait0-s/Endless.git`）です。作業前のHEADは `28bc57793ea53cc2842713107a0eaf02558873b6` です。

この実行環境では `.git` への書き込みが禁止され、`git add` が `Unable to create .../.git/index.lock: Permission denied` で拒否されます。GitHubへのHTTPS接続も失敗しています。ステージ、最終コミット、push、リモート先行状態の確認は完了していません。作業ツリーと既存の変更は保持しています。

## 実施した変更

- 通常Shadow Mapの最大解像度を2048から1024へ変更しました。影・照明そのものは残しています。影の細部の解像度が下がる品質上の変更です。DirectX 11で計測し、DX12のVirtual Shadow Mapsには適用されません。
- `EndlessWeaponInteraction.cpp` の武器交換確定処理を `ConfirmWeaponExchange` に分離しました。失敗時の弾薬・所持状態の復元と処理順序は維持しています。
- 所持上限を `AEndlessPlayerCharacter::MaxOwnedWeapons` にまとめ、取得判定・交換判定・画面表示から参照します。
- 操作案内の更新でドア・拾得物を二度探索していた箇所を、一度の探索結果を使う形にしました。フレームをまたぐキャッシュやBlueprintからC++への移設は行っていません。
- READMEをゲーム紹介、操作、UE5.6起動・ビルド方法、依存プラグイン・外部アセット、実画面、制限を含む内容に更新しました。

## 性能計測

UE5.6.1 / Win64 Development Editor / AMD Radeon (TM) Graphics / DirectX 11 / RenderOffscreenで計測しました。固定乱数シードでRoom 01を3インスタンス読み込み、敵4体、同じプレイヤー位置とカメラを使用しました。追加2室は離れた場所に配置したロード負荷の試験であり、隣接3室の同時可視を示すものではありません。接続した部屋の動作は別の生成・通過テストで確認します。

Editorウィンドウの要求サイズは1280×720、同じレイアウトで後に取得したPIE描画領域は847×550、VSyncとFPS上限は無効です。1280×720の製品版ゲームの性能とは比較できません。12秒のウォームアップ後、12秒ずつ2回をCSV Profilerで記録しました。ビルドと重なった最初の診断計測は最終比較から除外しました。

| 指標 | 変更前 | 変更後 |
|---|---:|---:|
| FPS（1000 / 平均FrameTime） | 25.71 | 26.59 |
| Game Thread | 17.00 ms | 16.70 ms |
| GPU | 38.82 ms | 37.53 ms |
| ShadowDepths（GPU） | 3.68 ms | 2.04 ms |

変更後の同一プロセスでも影解像度を2048→1024→1024→2048の順で各12秒ずつ比較しました。2048側の平均は25.58 FPS / ShadowDepths 3.63ms、1024側は25.97 FPS / 2.33msでした。影の深度描画は約36%減りましたが、総FPSの差は小さく、測定間の変動もあります。CPUの改善は確認できたとは扱いません。

AIのDecision/Action、Animation、TimerManager、AsyncLoadingも計測しています。AIの更新周期、部屋生成方式、アニメーション品質、同期ロード方式を変更する根拠は得られず、これらの大規模な変更は見送りました。生データと集計はローカルの `Saved/Finalization_20261026/*_normal*.csv`、`shadow_abba_*.csv`、`performance-summary.json` にあります。

## ビルドと動作検証

変更後にビルドしたDLLで、次の確認を行いました。検証スクリプト・原記録は `Saved/Finalization_20261026` に保持しています。レコード数には終了マーカーなどを含むため、仕様項目数とは異なります。

| 対象 | 結果と制約 |
|---|---|
| EndlessEditor / Endless（Development / Win64） | 変更前後ともソースから成功。変更後は34.50秒 / 67.22秒。下記のローカル出力用補助を使用 |
| Blueprint / AnimBlueprint | 依存グラフ内17件のコンパイルを実行。コンパイラエラーなし |
| 全ゲーム用マップ | 開始通路とRoom 01〜14、15件をロード・MAP CHECK。エラー0。Room 11〜14は各1件のNULL StaticMesh警告あり |
| 部屋の生成・回転・接続・ドア | 14部屋すべてで実生成、ドア通過・帰還を確認。0/90/180/270度の回転を14部屋に分配して検証。接続位置・角度の誤差0。全部屋×全角度の組合せ試験ではない |
| 連続ストリーミング | 8回前進と後戻り、最大3部屋、敵20体の制御、回復・武器ドロップとアンロード、所持状態の維持を確認 |
| 歩行・階段 | 初回の直進テストはRoom 09/11/13でドア等に阻まれ6レコード失敗。既存の通行経路に沿う追試では3部屋とも歩行・足音成功。上下階4部屋の階段通行も成功 |
| 戦闘・武器 | 初回58/60。遮蔽物のある配置で旋回・命中判定2件が失敗。遮蔽物のない一時テストマップで60/60成功。8武器、弾薬、リロード待機、所持上限、交換、照準と回避、ダメージを含む |
| 敵AI | 同室警戒共有、隣室への情報遮断、ドア境界、4部屋の階段経路を確認。初回20レコード中19成功。残る1件は古いテストが「閉塞時は調査先も下階に移る」と期待していた。現行コードは調査先を保持して再試行するため、追加試験で閉塞中の待機・上階目標の保持・障害物撤去後の移動再開を確認し4/4成功。ゲーム側の仕様変更なし |
| 同一部屋の複数インスタンス | 6/6成功。同じアセットの別Level間での警戒分離とアンロード後の掃除を確認 |
| Windows Cook | 対象15マップで終了コード0、95.03秒、集計0エラー・7警告。初回は600秒の上限で停止し、キャッシュを再利用した再試行で完了。Cook済みゲームパッケージ481件は既存追跡ファイルと追加選定7件ですべて充足 |
| 入力・照明 | Input Mappingの設定を確認し、PIEで移動・インタラクト・射撃等の処理を検証。DX11の実画面で照明・影を確認。物理キーボード全キー、ゲームパッド実機、DX12の照明は未検証 |

Room 11〜14の警告は `EndlessRoomPropPoint` の生成メッシュです。C++では確率に応じてメッシュを空にし、BeginPlayで候補を生成する構成です。警告を消すためだけにマップや生成確率を変更していません。先の途中報告の「全マップで警告なし」は誤りで、この表を最終結果とします。

複数のPIEプロセスが、結果JSONを出力した後の終了処理で `0xC0000005` になりました。Room 14の接続試験を単独で再実行すると終了コード0でしたが、NavigationSystemのCDOに関するensureが残っています。起動時の無名 `LogAutomationTest: Error: Condition failed` も検出しています。動作レコードが成功していても、これらをプロセス全体の正常終了として扱っていません。過去のローカル記録にも終了クラッシュがあり、今回の変更が原因とは確定していません。

DX12、製品版パッケージ起動、別PCでのクリーン取得・再現、長時間プレイは未検証です。

Cookの警告7件はSSL証明書ストア、ゲーム用マップ対象外の既存追跡ファイル `HS_WeaponPack_03/Demo/Maps` 4パッケージの読み込み不可、分離したCook出力先の自動掃除警告2件です。起動時にはTrace Serverの起動失敗警告もあります。デモ4件はLFS実体の一致を確認していますが、このEngineでロードできません。ゲーム用15マップのCookには含まれていません。不要と断定してアセットを削除したり、警告を隠したりしていません。Cook成功を製品版パッケージのプレイ成功とは扱っていません。

通常の `Build.bat` はユーザーフォルダーのUnrealBuildTool設定・MarketplaceRules出力への書き込み拒否で停止しました。既存の `Saved/Codex/EnvironmentAI_20261007/BuildHost/BuildHost.dll` を使い、出力をプロジェクトの `Saved/BuildUser` に向ける検証では、変更前後とも `EndlessEditor` と `Endless` のソースビルドに成功しています。

Editorビルドでは、この補助がインストール済みEngineの `BlueprintGraph → UMG` 循環依存宣言をプロセス内だけで補っています。Engineファイルは変更していません。MSVC 14.43はインストール済みUE5.6の推奨14.38と異なる警告があります。この結果を補助なしのクリーン環境でのビルド成功とは扱いません。

## ファイル分類と依存関係

| 分類 | 処置 |
|---|---|
| `.uproject`、Source、Config、既存Content、既存Docs | 既存の追跡状態を維持。必要箇所だけ修正 |
| 実行時依存として必要な未追跡アセット | Asset Registryのハード・ソフト参照、C++・Configの参照、Cook対象15マップから7件を追加対象に選定 |
| その他の未追跡アセット | 未使用と断定せず保持。名前やFab由来であることを削除根拠にしない |
| Binaries / Intermediate / Saved / DerivedDataCache / Backups / plus / ログ | `.gitignore` によりコミット対象外。復元資料や検証結果を含むため一括削除しない |
| ルートの `ystem updates` | Git差分統計の誤出力と内容を確認。6,723 bytesを削除。元データとSHA-256はバックアップに保存 |

追加対象の依存アセットは次の7件、合計183,446,985 bytesです。

```text
Content/Endless/Characters/Player/SKM_Player.uasset
Content/Endless/Environment/Props/SM_Industrial_steel_shelving_assembled_01.uasset
Content/Warehouse_Storage/Materials/Instances/MI_Warehouse_11.uasset
Content/Warehouse_Storage/Materials/M_Warehouse_11.uasset
Content/Warehouse_Storage/Textures/T_Warehouse_Basecolor_11.uasset
Content/Warehouse_Storage/Textures/T_Warehouse_Normal_11.uasset
Content/Warehouse_Storage/Textures/T_Warehouse_ORM_11.uasset
```

Asset Registryで523パッケージを調査しました。`CR/Demo/Meshes/SK_Mannequin` から存在しない `Characters/Mannequins/Meshes/SKM_Manny` と `SKM_Quinn` へのソフト参照2件が残っています。ハード参照ではありませんが、完全に参照欠落がないとは報告しません。実際に使用するプレイヤーメッシュとその依存は別途選定しています。

HEADのLFS対象935件はポインターと作業ツリー実体のSHA-256・サイズが一致し、`git lfs fsck --pointers HEAD` と `--objects HEAD` も成功しました。追加対象7件もバイナリ先頭・SHA-256と `filter=lfs` / `-text` を確認しました。新規ファイルのLFSオブジェクト登録・リモート転送は、ステージ・pushの制限により未完了です。

## README画像

| リポジトリ内画像 | 内容 | サイズ |
|---|---|---:|
| `Docs/Images/start-corridor.jpg` | 開始通路のPIE画面 | 40,369 bytes |
| `Docs/Images/room-04.jpg` | 間仕切りのある部屋と敵 | 58,944 bytes |
| `Docs/Images/room-11.jpg` | 上階への階段、PIE内の紹介用カメラ | 80,983 bytes |
| `Docs/Images/combat.jpg` | 接続後の部屋で敵と対峙するプレイ画面 | 67,418 bytes |

合計247,714 bytes。実画面をJPEG品質85で保存し、プレイ画面はEditor部分を除く切り抜きを行いました。架空画像・内容の合成はありません。相対リンクと画像の内容を確認し、4件とも既存の `*.jpg` のGit LFS設定（`filter=lfs`、`-text`）に一致しています。生成元・切り抜き矩形・ハッシュはローカルの `readme-images.json` に記録しています。

## コミット対象と公開の停止理由

追加・変更対象20件を [FinalizationFiles.txt](FinalizationFiles.txt) に明示しました。内訳は既存テキスト5件、依存アセット7件、JPEG4件、報告・計測・検証・一覧4件です。既存の `.uproject`、Source、Config、Content、`.gitignore`、`.gitattributes` 等はそのまま追跡対象に残ります。既存の `Config/DefaultEditor.ini` の状態は保全し、今回の選定には含めていません。

選定した作業ツリー実体は約184MBです。新規LFS対象の実体合計は183,694,699 bytes（アセットとJPEG）です。これはGitのポインターファイルのサイズではありません。パス別の実体サイズ・SHA-256・属性検査結果はローカルの `Saved/Finalization_20261026/final-file-review.json` に保存します。生成物・バックアップ・plus・ログは選定に含めていません。

この環境では `.git` が読み取り専用のため、明示パスによるステージが `index.lock: Permission denied` で失敗します。ステージ済み一覧・差分・削除一覧は空です。新規LFSポインターを含むステージ後の完全な検査、最終コミット、通常pushは未実施です。GitHub接続の再試行もポート443への接続失敗となり、リモート先行の有無は確認できません。既存HEADを最終版としてpushしていません。force push・reset・cleanは行っていません。

したがって、今回の編集と選定アセットは未コミットのままです。選定外の未追跡ファイル3,360件も保持しています。公開を再開できる環境では、この一覧と差分を再確認し、リモート状況を取得してからステージ・検査・コミット・通常pushを行う必要があります。想定メッセージは `Finalize Endless: measured shadow optimization, weapon refactor and release documentation` です。

計測値は [PerformanceResults.json](PerformanceResults.json)、検証の成功・失敗・終了コードは [ValidationResults.json](ValidationResults.json) に保存しています。

## 復元方法

作業前の履歴は `Backups/Finalization_20261026/repository.bundle` に保存し、`git bundle verify` が成功しています。元のソース・設定・READMEは同フォルダーの `files` にあります。元の状態・未追跡一覧は `status.txt`、`untracked.txt`、`inventory.json` で確認できます。bundleはGit履歴のバックアップで、LFS実体全体の複製ではありません。

今回Git履歴は変更していません。今回の変更を取り消す場合は、まず現在の編集を別途保存し、次の5ファイルだけを `files` 内の同名ファイルから戻してください。

```text
Config/DefaultEngine.ini
README.md
Source/Endless/Private/EndlessPlayerWeapons.cpp
Source/Endless/Private/EndlessWeaponInteraction.cpp
Source/Endless/Public/EndlessPlayerWeapons.h
```

今回作成した `Docs/Images` の掲載画像と `Docs/FinalizationReport.md`、`PerformanceResults.json`、`ValidationResults.json`、`FinalizationFiles.txt` は、後から加えた編集がないことを確認して取り除きます。誤出力ファイルも元の状態に戻す必要があれば `files/ystem updates` からコピーできます。PIEで更新されるセーブデータの元コピーは `Backups/Finalization_20261026/SaveGames` にあります。必要ならEditorを終了してから `Saved/SaveGames` の対応ファイルへ戻せます。Contentの移動・削除は行っていないため、16GBのアセットを複製し直す必要はありません。既存の `Config/DefaultEditor.ini` は作業対象にせず保持しました。`git reset --hard` や `git clean` は不要です。
