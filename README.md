# VulkanGpuSupport

`VulkanGpuSupport` は、外部所有の Vulkan 1.3 Device/Queue 上で、Resource、Upload、GPU完了付きPool、Compute画像処理を提供するC++17ライブラリです。

このrepositoryは初期設計段階です。実装は次の規範文書に従います。

- [文書セット案内](doc/architecture-and-implementation/README.md)
- [製品仕様書](doc/architecture-and-implementation/01_Specification.md)
- [アーキテクチャ・詳細設計書](doc/architecture-and-implementation/02_Architecture_Design.md)
- [実装手順書](doc/architecture-and-implementation/03_Implementation_Plan.md)
- [Codex向け実装契約](doc/architecture-and-implementation/04_Codex_Implementation_Brief.md)

## 固定方針

- C++17 / CMake 3.24+
- Vulkan 1.3、timeline semaphore、synchronization2
- 外部 `VkInstance` / `VkPhysicalDevice` / `VkDevice` / `VkQueue` を借用
- Vulkan-Headers、volk、VMA
- DXCはbuild時のみ、SPIRV-Toolsとvk-bootstrapはtest/sampleのみ
- Boost、ThreadToolkit/ThreadKit、nlohmann/jsonは使用しない
- WSI/SwapChain、Camera SDK、D3D interopは本体対象外

実装は `03_Implementation_Plan.md` のPhase 0から順に進めます。
