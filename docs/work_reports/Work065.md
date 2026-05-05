# Work065: アバターフロントエンド基盤の完成
Tag: [[React Three Fiber]], [[Zustand]], [[VRM]], [[Animation]], [[UI/UX]]

## 1. 目的
- Client (ブラウザ) 上で稼働するジェムちゃん（Gemma 3:27b）の3Dアバターに対し、適切なライティング、カメラワーク、アニメーション遷移、およびUI（吹き出し）を実装し、実用的な「AIコマンドセンター」の視覚的基盤を完成させる。

## 2. システム環境
- **Node:** Client ([Host masked] / IntelliJ IDEA) -> 最終デプロイ先: Node B ([Host masked] / Nginx)
- **Target:** `CanvasGemchan.js`, `VrmGemchanAvatar.js`, `App.js`

## 3. 作業ログ・解決プロセス

### 発生した課題と解決策
1. **ライティングの暗さ (逆光問題)**
   - **課題:** 初期設定の平行光源がZ軸奥から当たっており、顔に影が落ちていた。
   - **解決策:** `<directionalLight>` のZ座標をマイナスからプラスへ反転し、`<ambientLight>` と組み合わせたフロントライティングへ変更。

2. **ZustandステートとOrbitControlsの競合によるカメラ座標のズレ**
   - **課題:** ReactのStateでカメラ座標を更新しても、`OrbitControls` の内部ステートと競合し、視点が意図しない方向へズレる現象が発生。
   - **解決策:** Canvas内部に特務コンポーネント `CameraManager` を新設。ZustandのState変更をトリガーに、`camera.position` と `controls.target` を直接上書きし、`controls.update()` で強制同期させる「一方向の命令フロー」を確立した。

3. **ワンショットアニメーション後の硬直**
   - **課題:** 指定したポーズ（土下座、屈伸など）を再生した後、その姿勢のままフリーズしてしまう。
   - **解決策:** `THREE.AnimationMixer` の `finished` イベントをリッスンする機構を実装。ワンショット再生終了後、自動的に `idle_loop` へ0.5秒かけてクロスフェード（線形補間）で復帰するステートマシンを構築。

4. **3D空間における字幕（吹き出し）の追従と動的配置**
   - **課題:** 画面下部の固定UIでは無機質であり、カメラを動かすとキャラクターとセリフの紐づきが薄れる。
   - **解決策:** `@react-three/drei` の `<Html>` を採用し、ワールド座標上にCSSでスタイリングした吹き出しをピン留め。さらに、カメラ状態（全身 / バストアップ）をStateから判定し、三項演算子で吹き出しの3D座標を動的に切り替える宣言的UIを実装。

## 4. 成果物

**動的吹き出し座標制御の実装 (CanvasGemchan.js 抜粋)**
{% raw %}
```javascript
function CanvasGemchan() {
    // ...フック省略...
    const isBustUp = cameraPosition.z === -0.8;
    const balloonPosition = isBustUp ? [-0.35, 1.6, 2.5] : [-0.45, 1.5, 0];

    return (
        <Canvas 45 [cameraPosition.x, camera="{{" cameraPosition.y, cameraPosition.z], fov: position: }}>
            <CameraManager/>
            <ambientLight intensity={1} color={"#ffddcc"}/> 
            <directionalLight position={[2, 2, -3]} intensity={2}/> 
            <VrmGemchanAvatar/> 

            {messageText && (
                <Html center position="{balloonPosition}">
                    
                    {messageText}
                </Html>
            )}
            <OrbitControls enabled="{isCameraControlEnabled}" makeDefault/>
        </Canvas>
    );
}
```
{% endraw %}