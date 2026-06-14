# Work019: Llama 4 Scout導入と実戦投入試験

Tag: [[Node B]], [[Llama 4]], [[Benchmark]], [[Ollama]]

## 1. 目的

* GMKtec EVO-X2 の広大なVRAM (96GB) を最大限に活用できる「重量級かつ高速」な次世代モデルを発掘する。
* 従来の主力モデル `Llama 3.3 70B` に対する性能差（IQ, Speed, EQ）を検証し、リプレースの可否を判断する。

## 2. システム環境

* **Node:** Node B (GMKtec EVO-X2)
* **OS:** Ubuntu 24.04 LTS (Native)
* **Kernel:** Linux 6.8.0 (OEM Optimized)
* **Hardware:** Ryzen AI Max+ 395 / RAM 128GB (VRAM Allocation: **96GB**)
* **Target Model:** `llama4:16x17b` (Llama 4 Scout / 17B active parameters・16 experts・109B total parametersのMoEモデル)

  * **Quantization:** Q4_K_M
  * **Size:** Approx 65 GB（検証時のローカル表示）

## 3. 検証プロセスと結果

### 3.1 基礎スペック比較 (vs Llama 3.3 70B)

| 項目              | Llama 3.3 70B | Llama 4 (16x17b) | 評価                               |
| :-------------- | :------------ | :--------------- | :------------------------------- |
| **VRAM消費**      | 約 42 GB       | **約 65 GB**      | 96GBのVRAM割当内に収まり、高い利用率となった。      |
| **Gen Speed**   | 5.52 t/s      | **16.27 t/s**    | **約3倍高速**。本環境ではMoE構成が寄与した可能性がある。 |
| **Prompt Eval** | 155.87 t/s    | **291.28 t/s**   | **約1.9倍高速**。プロンプト処理時間が大きく短縮された。  |

### 3.2 定性評価（簡易IQ/EQ設問）

以下は正式なベンチマークではなく、特定の設問および生成条件に対する応答を比較した簡易的な定性評価である。

1. **物理演算（コップと鉄球）**

   * **Llama 3.3:** 「鉄が磁力で紙にくっつく」という、設問の条件と整合しない回答（Hallucination）を生成。
   * **Llama 4:** 「重力により紙の上に落ちて接触している」と、この設問では妥当な回答を生成。
2. **文脈理解（京都ぶぶ漬け）**

   * **Llama 4:** 意味は理解しているが、「茶碗を出させてから帰る」という**鬼畜な行動**を提案。意図は理解していたものの、提案した行動にはずれが見られ、EQ（空気読み）はやや天然。
3. **創造性・高Temperature設定試験 (Temperature 2.0)**

   * **Llama 4:** 酩酊状態になると「銀河一のアイドル」を自称し、謎の必殺技**『チャオチャンス発動』**を叫ぶなど、本試行では高いエンターテインメント性を示す出力となった。

## 4. 結論・成果

* 検証時点の個人研究環境では、**Llama 4 Scout (16x17b) を「絶対王者 (God Tier)」と評価し、主力モデルとして正式採用。**
* `Llama 3.3 70B` は、検証時点では予備モデル（Legacy）へ移行。
* 約65GB規模のQ4_K_Mモデルを96GBのVRAM割当内に配置し、EVO-X2の実機で16.27 t/sの生成速度を確認した。単体GPUのVRAMがモデルサイズを下回る環境では、モデル全体をGPUメモリ上に配置することが難しい規模である。

## 5. 関連コマンド

```bash
# Model Pull
ollama pull llama4:16x17b

# Run Benchmark (Example)
ollama run llama4:16x17b --verbose
```

※モデル本体およびモデル重みは本ポートフォリオには含めない。利用時はLlama 4 Community Licenseおよび配布元の利用条件を確認する。
