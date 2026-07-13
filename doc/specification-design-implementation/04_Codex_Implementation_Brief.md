# Codex向け VulkanGpuSupport 実装契約

この文書をCodexの作業指示として使用できます。詳細は同directoryの仕様書・設計書・実装手順書が規範です。

## 1. Mission

`VulkanGpuSupport` をC++17/CMakeで実装してください。本ライブラリは、外部所有Vulkan Device上で、汎用Resource、Upload、GPU完了付きPool、Compute画像処理を提供します。カメラ専用実装にしないでください。

## 2. 読む順序

1. `doc/specification-design-implementation/01_Specification.md`
2. `doc/specification-design-implementation/02_Architecture_Design.md`
3. `doc/specification-design-implementation/03_Implementation_Plan.md`
4. 本文書

仕様とVulkan Specificationが矛盾する場合はVulkan Specificationを優先し、文書・testを修正してください。勝手に仕様を省略しないでください。

## 3. Hard constraints

- C++17。
- CMake 3.24+。C++20 designated initializer等を使わない。
- target名 `VulkanGpuSupport::VulkanGpuSupport`。
- namespace `VulkanGpuSupport`。
- Vulkan 1.3 baseline。
- timeline semaphore必須。
- synchronization2必須。
- 外部 `VkInstance` / `VkPhysicalDevice` / `VkDevice` / `VkQueue`をborrow。
- CoreがDeviceを暗黙生成しない。
- raw `Vk*`を基礎にする。
- public APIにVulkan-Hpp型を要求しない。
- Vulkan-Headers + volk + VMAを使用。初期pinはHeaders `vulkan-sdk-1.4.350.1` (`8864cdc`)、volk `1.4.350` (`3ca312a`)、VMA `v3.4.0` (`3aa9212`)。
- DXC `v1.9.2602.24` (`d355aa8`) はbuild時のみ。
- SPIRV-Tools `v2026.2` (`0539c81`)、vk-bootstrap `v1.4.350` (`aee321c`) はtest/sampleのみ。
- Boost禁止。
- ThreadToolkit/ThreadKit禁止。
- nlohmann/json禁止。
- GLFW/SDL/ImGui/OpenCV/FFmpeg禁止。
- 独自Loader禁止。
- 独自GPU memory allocator禁止。
- runtime shader compiler禁止。
- WSI/SwapChain禁止。
- Camera SDK/Camera型禁止。
- D3D interop禁止。

## 4. 絶対に守る設計

1. global `volkLoadDevice`に依存せず、contextごとのinstance/device tableを使う。
2. VMAへVulkan function pointersを明示供給する。
3. `VulkanImage`に暗黙current layoutを持たせない。
4. before/after stateを各Upload/Processing操作へ渡す。
5. Queue host accessを内部mutexまたは注入host lock callbackで直列化する。同じraw Queueを外部コードが使う場合は同じcallbackを共用する。
6. Command Pool/Command Buffer/Descriptor Arenaはrecording epochごとにthread-confined。thread-safe UploadExecutorは独立CommandSlotを貸し出し、completion後にだけ別threadへadoptする。external VMA allocatorは内部同期有効を必須契約とする。
7. PoolはCPU Lease返却かつGPU completion後だけ再利用する。
8. retire pointなしでLeaseが破棄されたらFreeにせずquarantineする。
9. Processorは初期化後immutable。dispatchごとのDescriptor Setを共有しない。
10. 通常pathで `vkDeviceWaitIdle` を呼ばない。
11. v1.0 NV12/P010はR/RGの論理2-plane image。
12. build/test/sample以外でvk-bootstrapを使わない。

## 5. Scope of v1.0

実装するもの:

- Foundation値型、error、checked math
- Loader/dispatch/context/capabilities
- Queue/timeline/syncpoint
- Command context
- Buffer/Image/View/Bundle
- explicit state/barrier
- Descriptor arena
- SPIR-V registry/compute pipeline
- CPU byte/image view
- Buffer/Image upload
- Upload ring/batch/executor
- Image pool/lease/deferred release
- packed conversion/copy
- point/linear resize
- NV12/P010→RGBA
- fused convert+resize
- diagnostics callback/object naming
- tests/samples/install package

実装しないもの:

- Window/SwapChain
- Graphics pipeline
- native multiplanar
- external memory
- Video encode/decode
- camera metadata/drop policy
- advanced effects

## 6. Work sequence

`03_Implementation_Plan.md` のPhase 0から順番に進め、1 Phaseを原則1 PRにしてください。先のPhaseを混ぜないでください。

最初のPhaseで全機能stubを大量作成するのではなく、build/package skeletonだけを作ります。各Phaseでtestとdocumentを同時追加します。

## 7. Required output after each phase

- Changed files
- Requirement IDs
- Public API changes
- Ownership/synchronization decisions
- Build commands
- Test commands/results
- Validation layer result
- Unverified items
- Next phase readiness

## 8. Error handling

- Vulkan failureは `VulkanException`。
- invalid descriptorは `std::invalid_argument`。
- state machine違反は `std::logic_error`。
- timeout/exhaustionはoptional/bool。
- destructorはnoexcept。
- Device lost後はcontext/queue faulted state。
- partial constructionを漏らさない。

## 9. Testing requirement

最低限:

- public header standalone compile
- add_subdirectory/package smoke
- loader/dispatch isolation
- external Queue host-lock coordination
- borrowed external timeline SyncPoint
- concurrent queue submit
- concurrent UploadExecutor command-slot isolation
- command state machine
- VMA resource move/cleanup
- barrier validation
- upload ring randomized stress
- Buffer upload offset/state/readback
- stride付きpacked/NV12/P010 upload
- pool no-reuse-before-completion
- lease quarantine
- processing CPU golden
- validation error 0
- sanitizer CPU tests

GPU環境がない場合、GPU testを成功扱いにしないでください。未実施として報告してください。

## 10. Review stop conditions

次の場合は実装を止め、文書変更案を提示してください。

- Vulkan 1.3では要求を満たせない。
- public ownership契約を変える必要がある。
- non-Vulkan dependency追加が必要に見える。
- implicit state trackingなしではAPIが成立しないと判断した。
- Pool安全性のためblocking destructorが必要に見える。
- DXC binding ABIを固定できない。
- external Device borrowingとVMA統合が両立しない。
- 同一raw Queueを外部コードと安全に共有するhost synchronization契約が成立しない。

局所的なファイル名やprivate helper設計は自律的に決めて構いません。

## 11. Completion definition

1. `01_Specification.md` のv1.0受入条件を全て満たす。
2. 全要求IDがtestまたは明示的根拠へtraceされる。
3. Windows/Linux build。
4. Validation Error 0。
5. install/find_package成功。
6. runtimeにDXC不要。
7. forbidden dependencyなし。
8. docsとpublic headersが一致。

以上を満たすまで「完成」と報告しないでください。
