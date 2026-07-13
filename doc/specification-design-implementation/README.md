# VulkanGpuSupport 設計・実装ドキュメント

文書版: 0.1-draft  
基準日: 2026-07-13  
対象リポジトリ: `Isasa2357/VulkanGpuSupport`

## この文書セットの目的

本書群は、`VulkanGpuSupport` をこのチャットの文脈なしに実装できるよう、製品仕様、アーキテクチャ、公開API契約、実装順序、テスト条件、Codex向け作業規約を固定するものです。

`VulkanGpuSupport` はカメラ専用ライブラリではありません。CPU画像、動画デコーダ出力、推論前処理、シミュレーション結果などを、Vulkanリソースへ非同期アップロードし、Compute処理し、GPU完了を考慮して再利用するための汎用基盤です。カメラライブラリは、その一利用者として本ライブラリを使用します。

## 文書一覧

1. [01_Specification.md](01_Specification.md) — 製品仕様、要求、非目標、受入条件
2. [02_Architecture_Design.md](02_Architecture_Design.md) — モジュール、公開API、所有権、同期、スレッド安全性、依存統合
3. [03_Implementation_Plan.md](03_Implementation_Plan.md) — PR単位の実装手順、テスト、CI、リリース条件
4. [04_Codex_Implementation_Brief.md](04_Codex_Implementation_Brief.md) — Codexへそのまま渡せる実装契約

## 固定済みの主要判断

- 言語は C++17、ビルドは CMake。
- v1.0 の実行基準は Vulkan 1.3、timeline semaphore、synchronization2。
- 本体は外部で生成済みの `VkInstance` / `VkPhysicalDevice` / `VkDevice` / `VkQueue` を借用する。
- 本体が `VkInstance` や `VkDevice` を暗黙生成しない。
- Vulkan API呼び出しは raw `Vk*` handle を基礎とし、公開APIに Vulkan-Hpp 型を要求しない。
- 必須Vulkan依存は Vulkan-Headers、volk、Vulkan Memory Allocator (VMA)。
- DXCはHLSLからSPIR-Vを生成するビルド時ツール。実行時依存にはしない。
- SPIRV-Toolsとvk-bootstrapは開発・テスト・サンプル限定。
- Boost、ThreadToolkit / ThreadKit、nlohmann/json、OpenCV、FFmpeg、GLFW、SDL、ImGuiなどの非Vulkan依存を本体へ追加しない。
- 独自Vulkan Loader、独自GPUメモリアロケータ、独自shader compilerを実装しない。
- ライブラリ独自価値は Queue/Command/Sync、Resource、Upload、Pool/Lease、GPU lifetime、Compute Processing。
- `VulkanImage` 自体に暗黙の「現在layout」を保持しない。操作ごとに before/after state を明示する。
- Poolは、CPU上でLeaseが返却され、かつ指定されたGPU完了点が完了した後だけResourceを再利用する。
- v1.0のNV12/P010は、Vulkan native multiplanar imageではなく論理2-plane imageとして扱う。
- WSI/SwapChain、描画エンジン、カメラSDK、D3D interop、DMA-BUF import、Vulkan Videoはv1.0対象外。

## 採用依存の基準版

初期実装では、API/ABIの組み合わせを固定するため、次の確認済みtag/commitを基準にします。runtime要件はVulkan 1.3ですが、build時header/loaderはVulkan 1.4.350系の整合した組を使用します。

| 依存 | 基準tag | commit | 用途 |
|---|---|---|---|
| Vulkan-Headers | `vulkan-sdk-1.4.350.1` | `8864cdc` | 必須、PUBLIC header |
| volk | `1.4.350` | `3ca312a` | 必須、PRIVATE loader/dispatch |
| Vulkan Memory Allocator | `v3.4.0` | `3aa9212` | 必須、PRIVATE memory allocation |
| DirectXShaderCompiler | `v1.9.2602.24` | `d355aa8` | build時のみ、stable release |
| SPIRV-Tools | `v2026.2` | `0539c81` | test/build時のみ |
| vk-bootstrap | `v1.4.350` | `aee321c` | sample/test時のみ |

更新は依存ごとの独立PRで行い、Vulkan-Headersとvolkは同じVulkan header revisionへ揃えます。floating branchは使用しません。

## 実装の開始順

`03_Implementation_Plan.md` の Phase 0 から順番に実装してください。後続Phaseは、先行Phaseの完了条件を満たすまで着手しません。特に、Processingを先に実装してQueue、Resource State、Descriptor lifetimeを後付けする進め方は禁止します。

## 一次資料

実装時は以下の一次資料を正とします。

- Vulkan Specification: https://docs.vulkan.org/spec/latest/
- Vulkan Guide: https://docs.vulkan.org/guide/latest/
- Vulkan-Headers: https://github.com/KhronosGroup/Vulkan-Headers
- volk: https://github.com/zeux/volk
- Vulkan Memory Allocator: https://gpuopen-librariesandsdks.github.io/VulkanMemoryAllocator/html/
- DirectX Shader Compiler SPIR-V backend: https://github.com/microsoft/DirectXShaderCompiler/blob/main/docs/SPIR-V.rst
- SPIRV-Tools: https://github.com/KhronosGroup/SPIRV-Tools
- vk-bootstrap: https://github.com/charles-lunarg/vk-bootstrap

Vulkan Valid Usage ID、構造体要件、同期要件について本書とVulkan仕様が矛盾した場合、Vulkan仕様を優先し、差異を文書とテストへ反映してください。
