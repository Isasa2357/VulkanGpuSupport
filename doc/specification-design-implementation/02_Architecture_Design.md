# VulkanGpuSupport アーキテクチャ・詳細設計書

文書ID: VGS-DESIGN-001  
版: 0.1-draft  
基準日: 2026-07-13

この文書は詳細設計のindexです。設計本文は、GitHub上での閲覧性とCodexへの段階投入を考慮して4分冊にしています。4分冊を合わせた内容が設計書の正本であり、仕様書に従属します。

## 読む順序

1. [02A_Architecture_Core.md](02A_Architecture_Core.md) — 全体構造、repository、CMake、依存、error/ownership、Loader、Context、Queue、SyncPoint、Command Context
2. [02B_Resources_Transfer.md](02B_Resources_Transfer.md) — Buffer/Image/View/Bundle、Resource State、Barrier、Descriptor、Shader/Pipeline、CPU Image、Upload Ring/Batch/Executor
3. [02C_Pool_Processing.md](02C_Pool_Processing.md) — Image Pool、Lease、Deferred Release、Processing型、Processor、resize/YUV数式
4. [02D_Threading_Integration.md](02D_Threading_Integration.md) — Diagnostics、lock hierarchy、Device lost、上位ライブラリ境界、禁止設計、umbrella header、最小利用例

## 文書間の優先順位

```text
01_Specification.md
    > 02_Architecture_Design.md + 02A〜02D
        > 03_Implementation_Plan.md
            > 04_Codex_Implementation_Brief.md
```

Vulkan Specificationと本書が矛盾する場合はVulkan Specificationを優先し、要求IDを維持したまま仕様・設計・testを同時更新します。

## 横断的な固定事項

- C++17 / CMake 3.24+。
- runtime baselineはVulkan 1.3、timeline semaphore、synchronization2。
- external `VkInstance` / `VkPhysicalDevice` / `VkDevice` / `VkQueue`をborrow。
- raw `Vk*`、context-local volk dispatch、VMAを使用。
- Queue host access、Command Pool、Descriptor Pool、GPU completion、Resource lifetimeを別々の同期問題として扱う。
- Imageの暗黙current layoutを持たず、before/after stateを操作境界で明示。
- PoolはLease返却とGPU completionの両方が成立してから再利用。
- Processingはcaller-owned Command Contextへrecordし、Processorは初期化後immutable。
- カメラ固有概念、WSI、SwapChain、D3D interop、non-Vulkan dependencyは本体へ入れない。

実装者は分冊を省略せず、対象Phaseに対応する節と関連する横断節を確認してください。
