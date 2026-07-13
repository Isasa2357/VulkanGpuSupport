# VulkanGpuSupport 仕様・設計・実装ガイド

## 1. このフォルダの目的

このフォルダは、`VulkanGpuSupport` をこのチャットや口頭説明へ依存せず実装できるようにするための、実装上の正本です。

対象文書:

1. [`01_Specification.md`](01_Specification.md)  
   製品目的、適用範囲、依存関係、機能要件、非機能要件、受入条件を定義する。
2. [`02_ArchitectureDesign.md`](02_ArchitectureDesign.md)  
   モジュール境界、所有権、同期、公開API案、内部アルゴリズム、CMake・テスト構成を定義する。
3. [`03_ImplementationPlan.md`](03_ImplementationPlan.md)  
   実装フェーズ、変更ファイル、検証項目、完了条件、Codex向け作業規則を定義する。

実装者は上記3文書を順に読むこと。文書間で記述が競合する場合は、次の優先順位を適用する。

```text
01_Specification.md
    > 02_ArchitectureDesign.md
        > 03_ImplementationPlan.md
```

仕様を変更する場合は、先に上位文書を更新し、その後に下位文書と実装を追従させる。

## 2. 固定済みの設計判断

本計画では、以下を変更済み前提として扱う。

- ライブラリ名は `VulkanGpuSupport` とする。
- C++17で実装する。
- 公開APIはraw Vulkan handleを受け渡せる形にする。
- ライブラリ本体はカメラ、動画、ウィンドウ、GUI、D3Dへ依存しない。
- Vulkan以外の外部ライブラリへ依存しない。
- Boost、ThreadToolkit、nlohmann/jsonは使用しない。
- Vulkan function loadingは`volk`へ委譲する。
- Vulkan device memory allocationはVulkan Memory Allocator（VMA）へ委譲する。
- HLSLからSPIR-Vへの変換はDXCへ委譲し、実行時コンパイルは必須にしない。
- SPIR-V検証はSPIRV-Toolsを開発・テスト時だけ使用する。
- Instance／PhysicalDevice／Device生成は本体の責務にしない。
- `vk-bootstrap`はサンプル・統合テスト専用の任意依存とし、本体へリンクしない。
- Vulkan-Hppは使用しない。
- shaderc／glslangは使用しない。
- Window System Integration、Swapchain、Presentationはv1.xの対象外とする。
- v1.0の実行基準はVulkan 1.3、timeline semaphore、synchronization2とする。
- 最新のVulkan-Headersを使用しても、組み込みSPIR-Vのターゲット環境は`vulkan1.3`とする。
- 本体は外部で生成済みの`VkInstance`、`VkPhysicalDevice`、`VkDevice`、`VkQueue`を借用する。
- Vulkanオブジェクトの所有期間は、Vulkanの親子関係とGPU完了条件の両方を満たすこと。
- Image layoutを`VulkanImage`内部で暗黙追跡しない。処理ごとに明示する。
- Queue submitのホスト側外部同期は`VulkanQueue`が内部mutexで直列化する。
- CommandPool、CommandBuffer、DescriptorArenaはスレッド拘束型とする。
- Resource PoolはCPU lease返却だけで再利用せず、GPU完了を確認してから再利用する。
- 公開型へカメラ固有のtimestamp、frame number、camera ID、drop policyを入れない。

## 3. 規範用語

文書内では以下をRFC 2119相当の意味で使用する。

- **MUST / 必須**: 実装・利用時に必ず満たす。
- **MUST NOT / 禁止**: 実装・利用時に行ってはならない。
- **SHOULD / 推奨**: 合理的な理由がない限り従う。
- **SHOULD NOT / 非推奨**: 合理的な理由がない限り避ける。
- **MAY / 任意**: 実装判断または利用者判断で選択できる。

## 4. 実装開始前の確認

実装を開始するコミットでは、最低限次を追加する。

- ルート`README.md`
- `CMakeLists.txt`
- `cmake/Dependencies.cmake`
- `include/VulkanGpuSupport/`
- `src/`
- `shaders/`
- `test/`
- `sample/`
- `doc/`
- `.gitignore`
- `.clang-format`
- `CMakePresets.json`

ライセンスはリポジトリ所有者が決定する。実装者が推測でライセンス文書を追加してはならない。

## 5. 参照する一次資料

実装時は、古いブログや非公式サンプルより次を優先する。

- Khronos Vulkan Specification / Vulkan Guide
- Khronos Vulkan-Headers
- zeux/volk公式READMEおよびheader
- GPUOpen Vulkan Memory Allocator公式READMEおよびDoxygen
- Microsoft DirectXShaderCompilerのSPIR-V文書
- Khronos SPIRV-Tools
- vk-bootstrap公式README（サンプル・テストだけ）

依存バージョン更新時は、API互換性だけでなくVulkan header version、volkの生成元header version、VMAの対応Vulkan versionを揃えて確認する。
