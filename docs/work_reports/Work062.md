# Work062: EVO-X2アイドル時のCPUファン暴走とROCmスピンロック解消
Tag: [[Troubleshooting]], [[ROCm]], [[Ollama]], [[Docker]], [[Strix Halo]]

## 1. 目的
- Node B (EVO-X2) において、巨大モデル (`gpt-oss:120b`) をVRAMに常駐させた際、アイドル状態でもCPUファンが全開になるハードウェア・ソフトウェアの複合障害を解決する。

## 2. システム環境
- **Node:** Node B (EVO-X2 / Ryzen AI Max+ 395)
- **Target:** `[Date masked]_llm_core/docker-compose.yml` (ollama_rocm コンテナ)
- **OS:** Ubuntu 24.04 Native (Kernel 6.8.0-90-generic)

## 3. 作業ログ・解決プロセス
### 発生した課題
- モデルのロード中、推論していないにも関わらずACPI温度センサー (`acpitz-acpi-0`) が97℃を記録し、ファンが最大回転で張り付く。
- `top -H` コマンドで調査した結果、Ollamaの特定スレッドが常時CPUを1コア分（約100%）占有していることが判明。

### 解決策
- AMD ROCm の HSAランタイムが、レイテンシ削減のためにデフォルトでビジーウェイト（無限ポーリング）を行う仕様が原因。
- 以下の環境変数をコンテナに付与し、割り込み待ち（Interrupt）へ挙動を変更。
  - `HSA_ENABLE_INTERRUPT=1`
- スピンロックによる局所的な発熱が収まったことで、ACPIセンサーの異常値も34℃付近へと正常化した。

## 4. 成果物
(docker-compose.yml 追記内容)
```yaml
    environment:
      - HSA_ENABLE_INTERRUPT=1
      - HIP_VISIBLE_DEVICES=0
```