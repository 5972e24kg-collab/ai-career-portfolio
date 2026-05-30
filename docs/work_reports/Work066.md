# Work066: VRMアニメーションの完全In-Place化

Tag: [[Three.js]], [[VRM]], [[Animation]], [[UX]]

## 1. 目的

* Web上で再生するVRMアバター（ジェムちゃん）のモーション切り替え時に発生する「軸ズレ（意図しない位置へのスライド移動）」を解消する。
* 異なる出所（Mixamo、BOOTH等）のVRMAファイルを、Blender等のDCCツールで事前加工することなく、フロントエンドの処理のみでIn-Place（その場再生）化する。
* HipsのX軸（左右）・Z軸（前後）の初期オフセットを補正しつつ、Y軸（高さ）の相対移動は残すことで、ジャンプ等のダイナミックなモーションを維持する。

## 2. システム環境

* **Node:** 開発用クライアントPC / Web Frontend
* **Target:** `@react-three/fiber`, `@pixiv/three-vrm-animation` (`VrmGemchanAvatar.js`)

## 3. 作業ログ・解決プロセス

### 発生した課題

* 待機モーション（`idleLoopScene`）からワンショットモーションへのクロスフェード時、キャラが右手前や右奥に移動してから再生される現象が発生。
* トラックの解析（`console.log`によるHips座標の抽出）により、モーションごとにHips（ルートボーン）の初期座標が原点からズレていること、および移動データが含まれていることが判明。

### 解決策

* `GLTFLoader`でのロード直後、`THREE.AnimationClip`の`KeyframeTrack`配列を走査し、`hips`の`position`トラックを動的に書き換えるヘルパー関数 `applyInPlacePatch` を実装。
* 待機モーションとワンショットモーションの両方に対し、ロード時に1フレーム目のX/Z座標を基準として、全フレームから初期オフセットを差し引くパッチを適用。これにより、モーション素材ごとの開始位置ズレを吸収し、再生開始時の「X=0, Z=0」基準を統一した。

## 4. 成果物

```javascript
// 動的In-Place化パッチ（VrmGemchanAvatar.js）
const applyInPlacePatch = (clip) => {
    clip.tracks.forEach(track => {
        const trackName = track.name.toLowerCase();
        if (trackName.includes('hips') && trackName.includes('position')) {
            const values = track.values;
            if (values.length >= 3) {
                const startX = values[0];
                const startZ = values[2];
                // 全フレームから初期X/Zオフセットを差し引く（Y軸はそのまま）
                for (let i = 0; i < values.length; i += 3) {
                    values[i] -= startX;   // 初期位置Xを0基準に補正する
                    values[i+2] -= startZ; // 初期位置Zを0基準に補正する
                }
            }
        }
    });
};
```
