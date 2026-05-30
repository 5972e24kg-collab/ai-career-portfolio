# Work067: 外部モーションアセットのVRMA変換パイプライン構築

Tag: [[Unity]], [[VRM]], [[R3F]], [[Avatar]], [[Motion]]

## 1. 目的

* ジェムちゃん（Gemma 3:27b）のReact Three Fiber (R3F) 空間における肉体表現を拡張する。
* Mixamo等のFBXモーション、および市販・配布されているUnityPackage形式のモーションアセットを、R3F環境で再生可能な `.vrma` フォーマットへ変換するパイプラインを確立する。

## 2. システム環境

* **Node:** Client (開発用クライアントPC / [local IP masked])
* **Target:** Unity Editor (2022.3.74f1 LTS)
* **Plugins:** UniVRM v0.131.0, AnimationClipToVrmaSample

## 3. 作業ログ・解決プロセス

### 発生した課題

* Unity 6系ではUniVRMの動作が不安定になる懸念があり、安定した変換環境の選定が必要だった。
* UniVRM標準の `VRM10Animation` コンポーネントおよびエクスポート機能では、FBX内の `AnimationClip` を直接 `.vrma` として書き出すUIが存在せず、BVH形式等からの変換に限定されていた。

### 解決策

* VRChat/VRM界隈のエコシステムと最も親和性が高く安定している **Unity 2022.3 LTS (2022.3.74f1)** をあえて採用し、「肉体生成チャンバー」としてのベースキャンプを構築。
* FBXのボーンを `Humanoid` 規格へリグ設定（チートシートによるマッピング確認）し、正規化。
* 標準機能の不足を補うため、拡張スクリプト `AnimationClipToVrmaSample` を導入。これにより、内部的な仮想Humanoid骨格を用いたベイク処理が可能となり、Projectウィンドウから `AnimationClip` を右クリックして一発で `.vrma` へエクスポートするパイプラインが開通した。

## 4. 成果物

* Mixamo由来のサンプルモーションを `.vrma` 形式へ変換し、Reactテスト環境での動作確認を完了。
* 過去に購入して保管していた市販のUnityPackage形式モーション（計30種以上）が `.vrma` へコンバート可能であることが判明し、ジェムちゃんの動作語彙を大きく拡張できる見込みが得られた。
