# VulkanGpuSupport 製品仕様書

文書ID: VGS-SPEC-001  
版: 0.1-draft  
基準日: 2026-07-13  
想定初期リリース: 0.1.0、仕様安定後 1.0.0

---

## 1. 目的

`VulkanGpuSupport` は、Vulkanを使用するアプリケーションまたは上位ライブラリに対し、次の反復実装を再利用可能なC++17 APIとして提供します。

- 外部所有Vulkan Device/Queueへの安全な接続
- per-instance / per-device function dispatch
- Queue submitとtimeline semaphoreによる完了点
- Command Pool / Command BufferのRAIIと状態管理
- VMAを用いたBuffer/Image作成
- 明示的なResource Stateとsynchronization2 barrier
- CPU memoryからGPU Buffer/Imageへの非同期Upload
- Upload staging ringの再利用
- GPU完了を考慮したImage Poolとmove-only Lease
- GPU完了後の遅延破棄
- Compute Shaderによるformat conversion、resize、fused conversion + resize
- Validation、object naming、能力検証の補助

本ライブラリは「Vulkan API全体を覆うC++ラッパー」ではありません。また、特定のカメラSDK、動画デコーダ、UI、rendererに依存しません。

## 2. 背景と位置付け

既存のVulkan関連ライブラリは、以下の責務を十分に実装しています。

- Vulkan function loading: volk
- Vulkan device memory allocation: VMA
- HLSL to SPIR-V: DXC
- SPIR-V validation/optimization: SPIRV-Tools
- Instance/Device選択の簡易化: vk-bootstrap

したがって、`VulkanGpuSupport` はこれらを再実装しません。一方、Uploadのrecord/submit、画像planeの取り扱い、GPU完了付きPool、Lease lifetime、明示state、汎用Compute Processingは既存部品を組み合わせただけでは完成しないため、本ライブラリが実装します。

## 3. 設計原則

### 3.1 汎用性

公開APIの型名、引数、コメントに `Camera`、`Capture`、`FrameRate`、`DropFrame` などのカメラ固有概念を入れません。入力は `CpuImageView`、出力は `VulkanImageBundle` / `VulkanImageLease` として表現します。

### 3.2 明示性

Vulkanが要求する次の情報を隠しません。

- Queue family
- Pipeline stage/access mask
- Image layout
- GPU completion point
- Resource ownership
- borrowed/owned distinction
- thread-safety class

利便APIが内部でbarrierを記録する場合でも、呼び出し側がbefore/after stateを指定します。

### 3.3 既存実装の尊重

Loader、device memory allocator、shader compiler、SPIR-V validatorを自作しません。薄い統合adapterのみ実装します。

### 3.4 安全側の失敗

GPUが使用中か不明なResourceをPoolへ即時返却しません。Leaseの返却契約が不足した場合、Resourceを再利用せずquarantineし、診断を出します。silent corruptionよりcapacity lossを選びます。

### 3.5 Compose-first

Processing APIは、独自Command Bufferへ勝手にsubmitするだけのAPIにしません。主APIは呼び出し側Command Bufferへ記録する `Record*` 形式とし、複数処理を1 submitへまとめられるようにします。簡易 `Execute*` は上位convenienceとして実装します。

## 4. v1.0の範囲

### 4.1 対象プラットフォーム

- Windows 10/11 x64
- Linux x86_64
- Vulkan 1.3を提供するloaderとphysical device
- MSVC、Clang、GCCのC++17実装
- CMake 3.24以上

32-bit、Android、macOS/MoltenVKはv1.0の受入対象外です。移植を妨げる不要なOS API依存は入れません。

### 4.2 必須Vulkan機能

- Vulkan API version 1.3以上
- `timelineSemaphore` featureがDevice作成時に有効
- `synchronization2` featureがDevice作成時に有効
- primary QueueがComputeとTransferを実行可能
- 必須formatでSampled Image / Storage Image / Transfer Src/Dstの必要featureが利用可能

Vulkan 1.3でcore化されていても、Device作成時にfeatureを有効化する必要がある点をAPI契約に含めます。

### 4.3 初期Queueモデル

- v1.0の標準経路は単一primary Queue family。
- primary QueueはCompute + Transferを担当。
- 同一Queueへのlibrary内部host accessは `VulkanQueue` が直列化。
- 呼び出し側も同じraw `VkQueue`へ直接アクセスする場合、contextへ共有lock/unlock callbackを渡し、library外のQueue呼出しにも同じlockを使用する。callbackを渡さない場合、Queueへの全host accessを`VulkanQueue`経由へ限定する。
- 複数Queue間ownership transfer用のstate型は保持するが、高水準Transfer/Processing APIはv1.0では同一Queue familyを要求。
- dedicated transfer queue最適化はv1.x以降。

### 4.4 対応Pixel Format

v1.0で保証する論理format:

| PixelFormat | Plane構成 | VkFormat |
|---|---:|---|
| `R8Unorm` | 1 | `VK_FORMAT_R8_UNORM` |
| `RG8Unorm` | 1 | `VK_FORMAT_R8G8_UNORM` |
| `RGBA8Unorm` | 1 | `VK_FORMAT_R8G8B8A8_UNORM` |
| `BGRA8Unorm` | 1 | `VK_FORMAT_B8G8R8A8_UNORM` |
| `R16Unorm` | 1 | `VK_FORMAT_R16_UNORM` |
| `RG16Unorm` | 1 | `VK_FORMAT_R16G16_UNORM` |
| `RGBA16Float` | 1 | `VK_FORMAT_R16G16B16A16_SFLOAT` |
| `NV12` | 2 | Y=`R8_UNORM`, UV=`R8G8_UNORM` |
| `P010` | 2 | Y=`R16_UNORM`, UV=`R16G16_UNORM` |

NV12/P010は論理2-plane imageです。`VK_FORMAT_G8_B8R8_2PLANE_420_UNORM`等のnative multiplanar imageはv1.0では使用しません。NV12/P010のwidth/heightは偶数を要求します。表のformatは論理APIとして認識する集合であり、各Transfer/Processing用途はphysical deviceのformat featureを満たす場合だけ利用可能です。特にBGRA storage outputはcapability-dependentです。

### 4.5 Processing機能

v1.0:

- packed RGBA-like copy/format conversion
- NV12 → RGBA8/BGRA8/RGBA16F
- P010 → RGBA8/BGRA8/RGBA16F
- point resize
- linear resize
- convert + resize fused dispatch
- BT.601 / BT.709 / BT.2020
- Full / Limited range
- crop rectangle

v1.0対象外:

- blur、mask、remap、pyramid、3D LUT等の高度処理
- RGB→NV12/P010書き戻し
- native multiplanar sampler YCbCr conversion
- image compression format

高度処理はv1.x backlogとし、Core APIを破壊せず追加できる構成にします。

## 5. 非目標

次は本ライブラリに実装しません。

1. カメラ列挙、撮影開始停止、timestamp、frame sequence、drop policy
2. Media Foundation、V4L2、GStreamer、IC4等のcapture backend
3. Window、Surface、SwapChain、Present
4. Graphics rendering、mesh、scene graph、UI
5. D3D11/D3D12/CUDA/OpenCL interop
6. DMA-BUF、opaque FD/Win32 handle import/export
7. Vulkan Video decode/encode
8. Vulkan API全体のC++ wrapper
9. 独自Vulkan loader
10. 独自device memory allocator
11. 実行時HLSL/GLSL compiler
12. JSON設定、ログframework、thread pool
13. PNG/JPEG/MP4等のfile I/O
14. shader reflection framework

## 6. 依存仕様

### 6.1 必須依存

| 依存 | 用途 | link範囲 |
|---|---|---|
| Vulkan-Headers | Vulkan型・定数・構造体 | PUBLIC headers |
| volk | loader、instance/device dispatch table | PRIVATE implementation |
| VMA | Buffer/Image memory allocation | PRIVATE実装。外部allocator接続型のみ公開可 |

### 6.2 ビルド・開発依存

| 依存 | 用途 | 本体runtime依存 |
|---|---|---:|
| DXC | HLSL→SPIR-V | なし |
| SPIRV-Tools | `spirv-val`、任意optimizer | なし |
| Vulkan Validation Layers | test/sample | なし |
| vk-bootstrap | standalone test/sampleのInstance/Device作成 | なし |

### 6.3 明示的な非採用

- Boost
- ThreadToolkit / ThreadKit
- nlohmann/json
- Vulkan-Hpp
- shaderc/glslangを本プロジェクトのHLSL pipelineに混在
- GLFW/SDL/ImGui
- OpenCV/FFmpeg

標準C++ライブラリの `std::mutex`、`std::condition_variable`、`std::atomic`、`std::thread`、`std::filesystem`、`std::vector`、`std::optional` は使用可能です。

### 6.4 Version pinning

初期基準版は次で固定します。

| 依存 | tag | commit | 備考 |
|---|---|---|---|
| Vulkan-Headers | `vulkan-sdk-1.4.350.1` | `8864cdc` | runtime baselineはVulkan 1.3 |
| volk | `1.4.350` | `3ca312a` | Headersとrevisionを整合 |
| VMA | `v3.4.0` | `3aa9212` | internally synchronized allocatorを使用 |
| DXC | `v1.9.2602.24` | `d355aa8` | stable、build時のみ |
| SPIRV-Tools | `v2026.2` | `0539c81` | build/test時のみ |
| vk-bootstrap | `v1.4.350` | `aee321c` | sample/test時のみ |

- floating branch (`main`, `master`)をRelease buildで参照しない。
- FetchContentは上記commit SHAまたはtagへ固定する。
- SHA、license、更新日を `THIRD_PARTY.md` に記録する。
- システム/Vulkan SDK提供packageを使う経路と、明示opt-inのFetchContent経路を両方提供する。
- Vulkan-Headersとvolkは同じheader revisionへ揃える。
- dependency更新は独立PRとし、全GPU/validation/package testを実行する。

## 7. 機能要求

要求IDはテスト名とPR説明に使用します。

### 7.1 Foundation

- **VGS-FND-001**: C++17でbuildできること。
- **VGS-FND-002**: `VkResult`を保持する `VulkanException` と `CheckVkResult` を提供すること。
- **VGS-FND-003**: overflow-safeなsize/alignment計算を提供すること。
- **VGS-FND-004**: `ArrayView<T>`、`Extent2D`、`Rect2D`、`PixelFormat`等の依存なし値型を提供すること。
- **VGS-FND-005**: public headers単独include testに合格すること。

### 7.2 Loader / Dispatch / Context

- **VGS-CORE-001**: system loaderまたは利用者指定 `PFN_vkGetInstanceProcAddr` でvolkをprocess-wide一度だけ初期化すること。
- **VGS-CORE-002**: global `volkLoadDevice` に依存せず、contextごとの `VolkInstanceTable` / `VolkDeviceTable` を保持すること。
- **VGS-CORE-003**: 外部所有 `VkInstance` / `VkPhysicalDevice` / `VkDevice` / `VkQueue` を借用すること。
- **VGS-CORE-004**: 借用handleをcontext destructorで破棄しないこと。
- **VGS-CORE-005**: external `VmaAllocator`を借用でき、未指定ならcontextがVMA allocatorを作成・所有できること。external allocatorはVMA内部同期が有効であることを必須契約とすること。VMAは既存allocatorのcreation flagsを照会するAPIを提供しないため、呼び出し側がdescriptorでこの契約を宣言し、非対応宣言は生成時にrejectすること。
- **VGS-CORE-006**: API version、enabled feature宣言、Queue capability、format featureを検証すること。
- **VGS-CORE-007**: context/resource wrapperがDeviceより先に破棄される契約を文書化・debug検出すること。
- **VGS-CORE-008**: 複数context/複数Deviceが同一processに存在してもdispatchが交差しないこと。

### 7.3 Queue / Synchronization

- **VGS-SYNC-001**: `VulkanQueue` は同一 `VkQueue` へのlibrary内submit/wait host accessを直列化すること。外部コードも同じQueueを使用する場合は、contextで注入した共有host synchronization callbackをlibraryと外部コードが共用できること。
- **VGS-SYNC-002**: Queueごとにlibrary-owned timeline semaphoreを持ち、submitごとに単調増加値をsignalすること。
- **VGS-SYNC-003**: `VulkanSyncPoint` はtimeline semaphore/valueと所有またはborrowed stateを保持し、`IsComplete` / `Wait`を提供すること。外部所有timeline semaphore/valueをborrowするfactoryを提供すること。
- **VGS-SYNC-004**: `vkQueueSubmit2`を使用し、wait/signal stage maskを明示すること。
- **VGS-SYNC-005**: binary/timelineの外部wait/signal記述を受け入れられること。
- **VGS-SYNC-006**: value overflowを検出し、未定義なwrapを行わないこと。
- **VGS-SYNC-007**: CPU mutexによる直列化とGPU memory dependencyを混同しないこと。

### 7.4 Command

- **VGS-CMD-001**: `VulkanCommandContext` はCommand Poolとprimary Command Bufferを所有するmove-only型であること。
- **VGS-CMD-002**: Command Contextはthread-confinedであり、debug buildでowner thread違反を検出すること。
- **VGS-CMD-003**: Initial/Recording/Executable/Pending状態遷移を検証すること。
- **VGS-CMD-004**: Pending中のreset/re-recordを拒否し、完了後にだけ再利用すること。
- **VGS-CMD-005**: one-time submitを標準とし、明示optionで再利用可能記録を許可すること。
- **VGS-CMD-006**: 同一Command Poolを複数threadで暗黙共有しないこと。

### 7.5 Resource / State / Barrier

- **VGS-GPU-001**: VMAを用いるmove-only `VulkanBuffer` / `VulkanImage` を提供すること。
- **VGS-GPU-002**: memory classとしてDeviceLocal、Upload、Readbackを提供すること。
- **VGS-GPU-003**: mapped memoryのflush/invalidateを非coherent atom sizeに合わせて実行すること。
- **VGS-GPU-004**: `VulkanImageView` とnon-owning `VulkanImageViewRef`を区別すること。論理複数plane画像にはformat/extent/plane viewをまとめたnon-owning `VulkanImageBundleView`を提供すること。
- **VGS-GPU-005**: `VulkanImage`は暗黙current layout/stateを持たないこと。
- **VGS-GPU-006**: `VulkanImageState` / `VulkanBufferState` にstage/access/layout/queue familyを明示すること。
- **VGS-GPU-007**: `VkDependencyInfo` / `VkImageMemoryBarrier2` / `VkBufferMemoryBarrier2`を生成するbarrier helperを提供すること。
- **VGS-GPU-008**: image subresource rangeを常に明示し、mip/layer/aspectを検証すること。
- **VGS-GPU-009**: logical multi-planeを表す `VulkanImageBundle` / Viewを提供すること。
- **VGS-GPU-010**: Resource作成時にformat feature、usage、extent、allocation結果を検証すること。

### 7.6 Descriptor / Shader / Pipeline

- **VGS-DESC-001**: thread-confinedな `VulkanDescriptorArena` を提供すること。
- **VGS-DESC-002**: Descriptor Pool resetは関連GPU処理完了後だけ許可すること。
- **VGS-DESC-003**: Processor内部に書換え可能な共有Descriptor Setを保持しないこと。
- **VGS-SHADER-001**: SPIR-Vをbuild時生成し、runtime compilerを要求しないこと。
- **VGS-SHADER-002**: binding ABIを固定し、HLSL/C++定数layoutをstatic_assertとshader testで検証すること。
- **VGS-SHADER-003**: shader cache/pipeline cacheは初期化後thread-safeであること。
- **VGS-SHADER-004**: shader binary hashと生成コマンドを追跡可能にすること。

### 7.7 Upload

- **VGS-XFER-001**: packedおよびNV12/P010の `CpuImageView` を検証すること。
- **VGS-XFER-002**: CPU source pointerはstaging copy完了までのみ必要とし、GPU完了までは要求しないこと。
- **VGS-XFER-003**: persistently mapped staging ringを提供すること。
- **VGS-XFER-004**: ring sliceをGPU完了点と結び付け、完了前に上書きしないこと。
- **VGS-XFER-005**: wrap-around、alignment、noncoherent flushを正しく処理すること。
- **VGS-XFER-006**: v1.0はsource rowをtight packingへrepackしてからcopyすること。
- **VGS-XFER-007**: `vkCmdCopyBufferToImage2`を使用すること。
- **VGS-XFER-008**: copy前後のstate transitionを呼び出し側指定stateに基づき記録すること。
- **VGS-XFER-009**: low-level `VulkanUploadBatch` と thread-safe convenience `VulkanUploadExecutor` を分離すること。Executorはin-flight Command Context slotをpoolし、同一Command Pool/Command Bufferを複数threadが同時利用しないこと。
- **VGS-XFER-010**: submission失敗時にsliceを安全にrollbackまたはquarantineすること。
- **VGS-XFER-011**: CPU byte rangeからGPU BufferへのUploadを `vkCmdCopyBuffer2` で記録できること。
- **VGS-XFER-012**: Buffer uploadもbefore/after `VulkanBufferState`を受け取り、Image uploadと同じring/submit/retire契約を使用すること。

### 7.8 Pool / Lease / Deferred Release

- **VGS-POOL-001**: Image Bundle単位のthread-safe Poolを提供すること。
- **VGS-POOL-002**: `TryAcquire`、blocking `Acquire`、timeout付きAcquireを提供すること。
- **VGS-POOL-003**: capacity、initial count、max count、exhaustion behaviorをdescriptorで指定できること。
- **VGS-POOL-004**: Leaseはmove-onlyであること。
- **VGS-POOL-005**: `ReleaseAfter(syncPoint, finalState)` によりGPU完了後だけFreeへ戻すこと。
- **VGS-POOL-006**: GPUへ投入しなかった場合の `ReleaseNow(state)` を明示提供すること。
- **VGS-POOL-007**: retire情報なしでLeaseが破棄された場合、即時再利用せずquarantineすること。
- **VGS-POOL-008**: Acquire時に前回final stateを明示的に返すこと。
- **VGS-POOL-009**: Pool破棄とoutstanding LeaseのUAFを防ぐshared stateを持つこと。
- **VGS-POOL-010**: Deferred Release Queueは任意ResourceをSyncPoint完了後に破棄できること。
- **VGS-POOL-011**: 内部生成Imageの最初のAcquire stateは`UNDEFINED`に固定し、任意の初期layoutを申告だけで設定してはならないこと。

### 7.9 Processing

- **VGS-PROC-001**: Processorは初期化後immutableで、複数threadからRecord可能であること。
- **VGS-PROC-002**: Record APIはcaller-owned Command ContextとDescriptor Arenaを使用すること。
- **VGS-PROC-003**: source/destinationのbefore/after stateを必須引数とすること。
- **VGS-PROC-004**: format/extent/rect/usage/featureをdispatch前に検証すること。
- **VGS-PROC-005**: NV12/P010の2-plane samplingを実装すること。
- **VGS-PROC-006**: BT.601/709/2020、Full/Limitedを実装すること。
- **VGS-PROC-007**: Point/Linear resizeを実装すること。
- **VGS-PROC-008**: convert+resize fused dispatchを実装すること。
- **VGS-PROC-009**: out-of-placeをv1.0の原則とし、未対応aliasを拒否すること。
- **VGS-PROC-010**: dispatch後のmemory visibilityをafter stateへbarrierすること。
- **VGS-PROC-011**: resizeのpixel-center座標規約、edge clamp、point/linear roundingを仕様化し、CPU goldenと一致させること。
- **VGS-PROC-012**: NV12/P010のcode range、P010上位10bit解釈、BT.601/709/2020係数、chroma sitingを仕様化し、shaderとCPU referenceで共通契約にすること。

### 7.10 Diagnostics / Packaging

- **VGS-DIAG-001**: Debug Utilsが利用可能ならobject nameを設定できること。
- **VGS-DIAG-002**: ログ依存を持たず、optional diagnostic callbackを提供すること。
- **VGS-DIAG-003**: Validation Layer messageがErrorにならないGPU testを提供すること。
- **VGS-PKG-001**: `VulkanGpuSupport::VulkanGpuSupport` CMake targetをexportすること。
- **VGS-PKG-002**: build treeとinstall treeの `find_package` smoke testに合格すること。
- **VGS-PKG-003**: public headers、license、shader source/generated binary、CMake configをinstallすること。

## 8. 非機能要求

### 8.1 Thread safety

公開クラスを次の3分類で文書化します。

- **Thread-safe**: 同一instanceへの同時呼出し可
- **Thread-compatible**: 異なるinstanceは並列可。同一instanceは外部同期必要
- **Thread-confined**: 生成/利用/破棄を1 threadへ拘束

最低保証:

| 型 | 保証 |
|---|---|
| `VulkanExecutionContext` | 初期化後Thread-safe/read-only |
| `VulkanQueue` | Thread-safe。library内呼出しを直列化。raw Queueを外部共有する場合は同じhost-sync callbackが必要 |
| `VulkanSyncPoint` | copyable immutable、Thread-safe |
| `VulkanCommandContext` | Thread-confined |
| `VulkanDescriptorArena` | Thread-confined |
| `VulkanBuffer` / `VulkanImage` | Thread-compatible |
| `VulkanUploadRing` | Thread-safe allocation/retire |
| `VulkanUploadBatch` | Thread-confined |
| `VulkanUploadExecutor` | Thread-safe。in-flight command slotを排他的に貸し出し、同一Command Poolを並列利用しない |
| `VulkanImagePool` | Thread-safe |
| `VulkanImageLease` | move-only、Thread-compatible |
| Processor | 初期化後Thread-safe |
| Deferred Release Queue | Thread-safe |

### 8.2 Performance

- steady-state frame/image処理でDevice allocationを発生させない構成を可能にする。
- Upload RingとImage Poolは事前確保可能。
- Queue submitは必要な範囲だけmutexを保持する。
- shader/pipelineはcacheし、dispatchごとに再生成しない。
- v1.0は正しさ優先でrow repackを行う。zero-copy row stride最適化は測定後。
- CPU waitを通常経路で使用しない。Pool blocking APIのみ明示的に待機可。

### 8.3 例外安全性

- Create/Initializeは失敗時にVulkan object/VMA allocationを漏らさない。
- move代入は既存Resourceを正しく解放する。
- destructorは `noexcept`。
- destructor内で無期限waitしない。
- submit成功後の所有権移動はatomicにcommitする。

### 8.4 Lifetime

- `VkInstance` / `VkPhysicalDevice` / `VkDevice` / borrowed Queue / external allocatorは利用者所有。
- 利用者は、それらを全library objectより長く保持する。
- library-owned timeline semaphore、Command Pool、Descriptor Pool、Buffer/Image/View、internal VMA allocatorはlibraryが破棄。
- GPU pending objectは完了前に破棄しない。

### 8.5 Compatibility

- Semantic Versioningを採用。
- v1.xで既存public signatureを破壊しない。
- 新しいProcessor/formatは追加で提供。
- Source compatibilityを壊す変更はmajor versionのみ。
- Vulkan extension/promoted core差を内部capabilityで吸収するが、v1.0 runtime baselineは1.3。

### 8.6 診断可能性

- すべてのVulkan objectに任意debug nameを付与可能。
- 例外messageは操作名、VkResult名、主要descriptor情報を含む。
- PoolはFree/Leased/Pending/Quarantined countを取得可能。
- Upload Ringはcapacity/in-flight/high-water markを取得可能。

## 9. Public APIの基本契約

### 9.1 Handle公開

Wrapperは必要に応じてraw handle getterを提供します。

```cpp
VkDevice GetDevice() const noexcept;
VkQueue GetQueue() const noexcept;
VkBuffer GetBuffer() const noexcept;
VkImage GetImage() const noexcept;
VkImageView GetImageView() const noexcept;
```

raw handleを取得した利用者は、libraryが宣言するthread/lifetime/state契約を維持します。

### 9.2 Borrowed view

Non-owning viewはResourceを破棄せず、参照先を延命しません。GPU処理完了まで外部ownerがResourceを保持します。

### 9.3 State

`VulkanImage`のgetterから「現在layout」は取得できません。状態はcommand streamの性質であり、Resource objectの単一フィールドでは正しく表現できないためです。操作はbefore/after stateを明示し、次の操作へafterを引き継ぎます。

### 9.4 Pool state

Poolだけは、Resource返却時に利用者が申告したfinal stateをslot metadataとして保持し、次回Acquire結果に明示して返します。これは暗黙trackingではなく、Lease境界での明示的なstate受渡しです。

### 9.5 Timeout

CPU wait APIは `std::chrono::nanoseconds` を使用し、無限待機は明示constantで指定します。timeout時は例外ではなくbool/optionalを返すAPIを基本とします。

## 10. v1.0受入条件

以下をすべて満たした時点で1.0候補とします。

1. Windows/MSVCとLinux/GCCまたはClangでbuild。
2. public header standalone compile testが全件成功。
3. Vulkan Validation Layer有効時、全GPU testにError messageなし。
4. 同一processで2 context/2 device dispatch isolation test成功。2 deviceがない環境ではmockable table testを実施。
5. 8 CPU threadから同一Queueへ各1,000 submitして破損・data raceなし。
6. Upload Ring wrap-around/timeout/retire stress test成功。
7. Image PoolがGPU完了前にslotを再利用しないことをテストで証明。
8. Lease未retire破棄時にslotがquarantineされること。
9. RGBA copy/point resizeはgoldenと完全一致。
10. Linear resizeとYUV変換は規定tolerance内。
11. NV12/P010 stride付きCPU入力を正しくupload。
12. `add_subdirectory`、install、`find_package` smoke test成功。
13. DXCをruntimeに要求しない実行確認。
14. CPU byte range→GPU Buffer upload/readback test成功。
15. ASan/UBSan対象CPU test成功。対応環境ではTSanによるPool/Queue test成功。
16. `README.md`、API docs、third-party license、sampleを同梱。

## 11. v1.x以降のbacklog

優先候補:

- dedicated transfer queueとqueue family ownership transfer executor
- native multiplanar image / sampler YCbCr conversion
- DMA-BUF / Win32 external memory import
- readback ring
- RGB→NV12/P010
- remap、blur、mask、threshold、pyramid、3D LUT
- descriptor indexing / descriptor buffer backend
- Vulkan 1.4 baselineの再評価
- Android/MoltenVK

これらをv1.0のAPIへ先回りして実装しません。ただし、state型、bundle型、provider型は追加を妨げないようにします。

## 12. 要求トレーサビリティ

各test suiteは要求IDをmetadataまたはtest nameに含めます。例:

```text
Core.DispatchIsolation.VGS_CORE_008
Queue.ConcurrentSubmit.VGS_SYNC_001
Upload.RingWrapAndRetire.VGS_XFER_004
Pool.NoReuseBeforeGpuCompletion.VGS_POOL_005
Processing.Nv12Bt709Limited.VGS_PROC_005
```

PR本文には、実装した要求ID、追加test、未対応事項を記載します。

## 13. 仕様変更手順

1. Issueで要求IDと変更理由を記録。
2. 仕様書を先に更新。
3. 設計書と実装手順の影響節を更新。
4. compatibilityとmigrationを記載。
5. 実装・testを同じPRまたは直後PRで更新。
6. 公開API変更はAPI review checklistを通す。

---

以上を `VulkanGpuSupport` v1.0の規範仕様とします。
