# VulkanGpuSupport アーキテクチャ・詳細設計書 — Part A: 全体構造・Context・Queue・Command

文書ID: VGS-DESIGN-001  
版: 0.1-draft  
基準日: 2026-07-13

> 設計書index: [02_Architecture_Design.md](02_Architecture_Design.md)

---

## 1. アーキテクチャ概要

`VulkanGpuSupport` は、Vulkan APIそのものを隠蔽する巨大frameworkではなく、外部Device上で汎用GPU data pathを構成するための薄いmodule群です。

```text
Application / Camera / Decoder / Inference / Simulation
                         |
                         v
+--------------------------------------------------------------+
| VulkanGpuSupport                                             |
|                                                              |
| Foundation                                                   |
|    |                                                         |
|    v                                                         |
| Core: Loader / Dispatch / Context / Queue / Sync / Command   |
|    |                                                         |
|    v                                                         |
| Gpu: Buffer / Image / View / State / Barrier / Descriptor    |
|    |--------------------|--------------------|               |
|    v                    v                    v               |
| Transfer             Pool               Processing           |
| Upload Ring/Batch    Lease/Retire       Compute Pipelines     |
|    \____________________|____________________/               |
|                         v                                    |
| Diagnostics: names / counters / optional callback            |
+--------------------------------------------------------------+
                         |
                         v
                 External VkDevice / VkQueue
```

### 1.1 依存方向

```text
Foundation
  -> Core
      -> Gpu
          -> Transfer
          -> Pool
          -> Processing
      -> Diagnostics
```

- 循環依存を禁止します。
- ProcessingはPoolに依存しません。
- PoolはProcessingの型を知りません。
- Camera libraryとのadapterは別repository/targetに置きます。
## 2. Repository構成

```text
VulkanGpuSupport/
├── CMakeLists.txt
├── LICENSE
├── README.md
├── THIRD_PARTY.md
├── cmake/
│   ├── Dependencies.cmake
│   ├── VulkanGpuSupportConfig.cmake.in
│   ├── CompileVulkanShader.cmake
│   ├── EmbedSpirv.cmake
│   └── PackageSmokeTest.cmake
├── include/VulkanGpuSupport/
│   ├── VulkanGpuSupport.hpp
│   ├── Foundation/
│   │   ├── VulkanFoundation.hpp
│   │   ├── Version.hpp
│   │   ├── ArrayView.hpp
│   │   ├── Error.hpp
│   │   ├── CheckedMath.hpp
│   │   ├── PixelFormat.hpp
│   │   └── Geometry.hpp
│   ├── Core/
│   │   ├── VulkanCore.hpp
│   │   ├── VulkanLoader.hpp
│   │   ├── VulkanCapabilities.hpp
│   │   ├── VulkanExecutionContext.hpp
│   │   ├── VulkanQueue.hpp
│   │   ├── VulkanSyncPoint.hpp
│   │   └── VulkanCommandContext.hpp
│   ├── Gpu/
│   │   ├── VulkanGpu.hpp
│   │   ├── VulkanResourceState.hpp
│   │   ├── VulkanBarrier.hpp
│   │   ├── VulkanBuffer.hpp
│   │   ├── VulkanImage.hpp
│   │   ├── VulkanImageView.hpp
│   │   ├── VulkanImageBundle.hpp
│   │   ├── VulkanDescriptorArena.hpp
│   │   ├── VulkanShaderModule.hpp
│   │   ├── VulkanComputePipeline.hpp
│   │   └── VulkanDeferredReleaseQueue.hpp
│   ├── Transfer/
│   │   ├── VulkanTransfer.hpp
│   │   ├── CpuImageView.hpp
│   │   ├── VulkanUploadRing.hpp
│   │   ├── VulkanUploadBatch.hpp
│   │   └── VulkanUploadExecutor.hpp
│   ├── Pool/
│   │   ├── VulkanPool.hpp
│   │   ├── VulkanImagePool.hpp
│   │   ├── VulkanImageLease.hpp
│   │   └── VulkanBufferPool.hpp
│   ├── Processing/
│   │   ├── VulkanProcessing.hpp
│   │   ├── ProcessingTypes.hpp
│   │   ├── VulkanProcessingContext.hpp
│   │   ├── VulkanProcessingShaderRegistry.hpp
│   │   ├── VulkanFormatConverter.hpp
│   │   ├── VulkanResizer.hpp
│   │   └── VulkanFusedProcessor.hpp
│   └── Diagnostics/
│       ├── VulkanDiagnostics.hpp
│       ├── VulkanObjectName.hpp
│       └── VulkanDiagnosticCallback.hpp
├── src/
│   ├── Foundation/
│   ├── Core/
│   ├── Gpu/
│   ├── Transfer/
│   ├── Pool/
│   ├── Processing/
│   ├── Diagnostics/
│   └── VmaImplementation.cpp
├── shaders/
│   ├── include/
│   │   ├── ProcessingCommon.hlsli
│   │   ├── ColorConversion.hlsli
│   │   ├── YuvPrimitives.hlsli
│   │   └── ResizePrimitives.hlsli
│   ├── src/
│   │   ├── ConvertPacked.hlsl
│   │   ├── ConvertYuv420.hlsl
│   │   ├── Resize.hlsl
│   │   └── FusedConvertResize.hlsl
│   ├── generated/
│   │   └── *.spv
│   └── ShaderRegistry.in.cpp
├── sample/
│   ├── 01_BorrowedContext/
│   ├── 02_UploadRgba/
│   ├── 03_UploadNv12/
│   ├── 04_ConvertResize/
│   └── 05_ImagePool/
├── test/
│   ├── TestHarness.hpp
│   ├── Cpu/
│   ├── Gpu/
│   ├── Stress/
│   ├── Package/
│   └── CMakeLists.txt
└── doc/
    └── specification-design-implementation/
```
## 3. NamingとCoding規約

- C++ namespace: `VulkanGpuSupport`
- CMake target: `VulkanGpuSupport::VulkanGpuSupport`
- public include root: `<VulkanGpuSupport/...>`
- class: PascalCase
- function/member: PascalCase（既存D3DHelperとの認知整合を優先）
- private member: `m_` prefix
- enum classを使用
- owning classは原則move-only
- non-owning型は末尾 `View` または `Ref`
- descriptorは末尾 `Desc`
- GPU完了値は末尾 `Point`
- public APIで裸の整数をformat/stage/accessの代用にしない
- `VK_NULL_HANDLE` / `nullptr` を明示初期値にする
- public headerから実装専用volk型を出さない
- C++17のみを使用し、designated initializer、`std::span`等のC++20機能を使用しない
## 4. CMakeと依存統合

### 4.0 固定dependency baseline

| 依存 | tag | commit | target内の位置付け |
|---|---|---|---|
| Vulkan-Headers | `vulkan-sdk-1.4.350.1` | `8864cdc` | PUBLIC header |
| volk | `1.4.350` | `3ca312a` | PRIVATE loader/dispatch |
| VMA | `v3.4.0` | `3aa9212` | PRIVATE allocator |
| DXC | `v1.9.2602.24` | `d355aa8` | build tool |
| SPIRV-Tools | `v2026.2` | `0539c81` | build/test tool |
| vk-bootstrap | `v1.4.350` | `aee321c` | sample/test only |

Vulkan-Headers/volk/vk-bootstrapは1.4.350 revisionへ揃えます。runtime feature baselineはVulkan 1.3のままであり、Vulkan 1.4 featureを公開APIの必須条件にはしません。依存更新は独立PRで実施し、floating branchは使用しません。

### 4.1 Root options

```cmake
cmake_minimum_required(VERSION 3.24)
project(VulkanGpuSupport VERSION 0.1.0 LANGUAGES CXX)

option(VGS_BUILD_TESTS              "Build VulkanGpuSupport tests" ON)
option(VGS_BUILD_SAMPLES            "Build VulkanGpuSupport samples" ON)
option(VGS_ENABLE_WARNINGS          "Enable strict warnings" ON)
option(VGS_ENABLE_PROCESSING        "Build compute processing module" ON)
option(VGS_REBUILD_SHADERS          "Rebuild SPIR-V with DXC" OFF)
option(VGS_VALIDATE_SHADERS         "Run spirv-val on generated shaders" ON)
option(VGS_FETCH_DEPENDENCIES       "Fetch pinned Vulkan dependencies" OFF)
option(VGS_ENABLE_VK_BOOTSTRAP      "Use vk-bootstrap in tests/samples" OFF)
option(VGS_INSTALL                  "Enable install/export rules" ON)
option(VGS_ENABLE_PACKAGE_SMOKE     "Enable install/find_package test" ON)
```

- C++17 required、extensions off。
- Library targetは初期版ではSTATIC。
- `BUILD_SHARED_LIBS`対応はABI export macroを整備した後のv1.x。
- Core targetはVulkan loader libraryへ直接linkしません。
- unit test frameworkは外部依存を追加せず、repository内の軽量`TestHarness.hpp`を使用します。

### 4.2 Target dependency

概念上:

```cmake
target_link_libraries(VulkanGpuSupport
    PUBLIC
        Vulkan::Headers
    PRIVATE
        volk::volk
        GPUOpen::VulkanMemoryAllocator
)
```

package target名が環境で異なる場合、`Dependencies.cmake` 内でcanonical aliasを作り、本体CMakeには差を漏らしません。

### 4.3 `VK_NO_PROTOTYPES`

- public headerはVulkan型だけを使用します。
- library実装targetには `VK_NO_PROTOTYPES` をPRIVATE定義。
- consumer全体へ `VK_NO_PROTOTYPES` をPUBLIC強制しません。
- implementation translation unitはVulkan functionを直接global symbolで呼ばず、context dispatch table経由で呼びます。

### 4.4 volk

`volk::volk` dependency targetがvolk implementationを提供します。本repositoryで `VOLK_IMPLEMENTATION` を重複定義しません。dependency targetを使わずsource統合へ切り替える場合も、implementationはprocess内でちょうど1 translation unitに限定します。

process-wide初期化:

```cpp
struct VulkanLoaderDesc {
    PFN_vkGetInstanceProcAddr getInstanceProcAddr = nullptr;
};

class VulkanLoader final {
public:
    static void Initialize(const VulkanLoaderDesc& desc = {});
    static bool IsInitialized() noexcept;
};
```

規約:

1. `getInstanceProcAddr == nullptr` はsystem loaderを使う。
2. 非nullなら `volkInitializeCustom` 相当を使用。
3. `std::once_flag` で一度だけ初期化。
4. 2回目以降に異なるcustom functionが渡されたら例外。
5. instance/deviceごとにtableをロード。
6. library内でglobal `volkLoadDevice` を呼ばない。
7. すべてのDevice functionは当該contextのtableから呼ぶ。

### 4.5 VMA

`VmaImplementation.cpp` だけで `VMA_IMPLEMENTATION` を定義します。

```cpp
#define VMA_IMPLEMENTATION
#define VMA_STATIC_VULKAN_FUNCTIONS 0
#define VMA_DYNAMIC_VULKAN_FUNCTIONS 0
#include <vk_mem_alloc.h>
```

- VMAの関数pointerは `VmaVulkanFunctions` へ明示設定。
- `vkGetInstanceProcAddr` / `vkGetDeviceProcAddr` と必要core functionをcontext dispatchから設定。
- VMAがglobal Vulkan prototypeへfallbackしない構成にする。
- external allocatorが渡された場合、libraryはdestroyしない。external allocatorはVMA内部同期が有効であることを必須契約とします。VMAから既存allocatorのcreation flagsは照会できないため、`externalAllocatorInternallySynchronized`を呼び出し側宣言として検証し、falseはrejectします。呼び出し側の虚偽宣言は契約違反です。
- internal allocatorの場合、`vulkanApiVersion`、instance、physicalDevice、device、callbacksをcontext descから設定。
- allocator破棄前に全VMA allocationが解放済みであることをdebug検査。

### 4.6 DXC / SPIR-V

標準shader build commandの概念:

```text
dxc -T cs_6_0 -E Main -spirv -fspv-target-env=vulkan1.3 -HV 2021 \
    -O3 -I shaders/include input.hlsl -Fo output.spv
spirv-val --target-env vulkan1.3 output.spv
```

Debug shaderは `-Od -Zi` を使用可能ですが、Releaseにdebug情報を必須化しません。

- runtimeでDXC DLL/shared libraryをloadしない。
- bindingはHLSL `[[vk::binding(binding, set)]]` で明示。
- push constantは `[[vk::push_constant]]` を使用。
- reflection libraryは使用しない。
- binding/layout ABIはC++側定数とshader testで固定。
- precompiled SPIR-Vをrepositoryまたはrelease packageへ含める。
- `VGS_REBUILD_SHADERS=ON` の場合だけDXCが必須。
- generated binaryにはSHA-256と生成commandを `ShaderRegistry` へ埋め込む。
## 5. Error model

### 5.1 Exception型

```cpp
class VulkanException : public std::runtime_error {
public:
    VulkanException(VkResult result,
                    std::string operation,
                    std::string detail = {});

    VkResult Result() const noexcept;
    const std::string& Operation() const noexcept;
};
```

- Vulkan API失敗: `VulkanException`
- invalid descriptor/state: `std::invalid_argument`
- state machine違反: `std::logic_error`
- timeout: bool/optional、通常は例外にしない
- Pool exhaustion: `TryAcquire`はempty、blocking APIはtimeout結果

### 5.2 Destructor

- 全destructorは `noexcept`。
- Vulkan destroy functionは失敗値を返さないため通常解放。
- GPU pendingが疑われる場合、destructor内で無期限waitせずdiagnostic + quarantine/deferred lifetimeを使用。
- callbackから例外を伝播しない。
## 6. 所有権モデル

### 6.1 Borrowed root handles

`VulkanExecutionContext` は次を借用します。

- `VkInstance`
- `VkPhysicalDevice`
- `VkDevice`
- primary `VkQueue`
- optional external `VmaAllocator`
- optional `VkAllocationCallbacks`

これらをdestroyしません。

### 6.2 Context-owned objects

- per-instance dispatch table
- per-device dispatch table
- internally-created `VmaAllocator`
- `VulkanQueue` wrapper state
- Queue completion timeline semaphore
- capability snapshot
- diagnostics state

### 6.3 Context state sharing

Resource wrapperがdispatch/allocatorより長生きしないよう、内部で `std::shared_ptr<Detail::ContextState>` を保持します。ただし、このshared stateは外部 `VkDevice` 自体を所有しません。

```text
External VkDevice lifetime
    > VulkanExecutionContext public wrapper
    >= ContextState shared by all wrappers
    > VkBuffer/VkImage/VkImageView/Pool/Lease
```

利用者がDeviceを先に破棄するのは契約違反です。

### 6.4 Resource ownership表

| 型 | handle所有 | copy | move |
|---|---:|---:|---:|
| `VulkanBuffer` | yes | no | yes |
| `VulkanImage` | yes | no | yes |
| `VulkanImageView` | yes | no | yes |
| `VulkanImageViewRef` | no | yes | yes |
| `VulkanImageBundle` | yes | no | yes |
| `VulkanImageBundleView` | no | yes | yes |
| `VulkanSyncPoint` | timeline state共有 | yes | yes |
| `VulkanCommandContext` | pool/cmd yes | no | yes |
| `VulkanImageLease` | pool slot lease | no | yes |
## 7. Loader / Dispatch / Context詳細

### 7.1 Public descriptor

`VmaAllocator`を受け取るため、public headerではVMAのopaque handle型だけをforward declarationします。VMAの全headerをconsumerへ強制includeしません。`VmaAllocator_T`のforward declaration/aliasは`namespace VulkanGpuSupport`の外側（global namespace）に置き、VMA本来のopaque型と同一型にします。

```cpp
struct VmaAllocator_T;
using VmaAllocator = VmaAllocator_T*;

struct VulkanEnabledFeatureSet {
    bool timelineSemaphore = false;
    bool synchronization2 = false;
};

using VulkanQueueHostLockCallback = void(*)(void* userData) noexcept;

struct VulkanQueueHostSynchronizationDesc {
    VulkanQueueHostLockCallback lock = nullptr;
    VulkanQueueHostLockCallback unlock = nullptr;
    void* userData = nullptr;
};

struct VulkanQueueBindingDesc {
    VkQueue queue = VK_NULL_HANDLE;
    uint32_t familyIndex = VK_QUEUE_FAMILY_IGNORED;
    uint32_t queueIndex = 0;
    VkQueueFlags capabilities = 0;
    VulkanQueueHostSynchronizationDesc hostSynchronization = {};
};

struct VulkanExecutionContextDesc {
    VkInstance instance = VK_NULL_HANDLE;
    VkPhysicalDevice physicalDevice = VK_NULL_HANDLE;
    VkDevice device = VK_NULL_HANDLE;
    uint32_t apiVersion = VK_API_VERSION_1_3;

    VulkanQueueBindingDesc primaryQueue = {};
    VulkanEnabledFeatureSet enabledFeatures = {};

    VmaAllocator externalAllocator = nullptr;
    bool externalAllocatorInternallySynchronized = true;
    const VkAllocationCallbacks* allocationCallbacks = nullptr;

    PFN_vkGetInstanceProcAddr getInstanceProcAddr = nullptr;
    const char* debugName = nullptr;
};
```

`capabilities`は利用者申告だけに依存せず、physical device queue family propertyと照合します。`hostSynchronization.lock`/`unlock`は両方nullまたは両方non-nullでなければなりません。nullならwrapper内部mutexを使用します。同じraw Queueを外部コードでも使う場合、外部コードは必ず同じcallbackを使ってhost accessを直列化します。同一Queueを別々の内部mutexを持つ複数wrapperで包むことは禁止します。callbackは例外を投げず、callback中に同じQueue wrapperへ再入してはいけません。callbackの`userData`、`VkAllocationCallbacks`、external allocator、root Vulkan handlesはContextStateと全派生Resourceより長く生存させます。

### 7.2 Public context

```cpp
class VulkanExecutionContext final {
public:
    static std::shared_ptr<VulkanExecutionContext>
    Create(const VulkanExecutionContextDesc& desc);

    VkInstance GetInstance() const noexcept;
    VkPhysicalDevice GetPhysicalDevice() const noexcept;
    VkDevice GetDevice() const noexcept;
    VmaAllocator GetAllocator() const noexcept;

    VulkanQueue& PrimaryQueue() noexcept;
    const VulkanCapabilities& Capabilities() const noexcept;

    std::unique_ptr<VulkanCommandContext> CreateCommandContext() const;
    void CollectDeferredReleases();
    void WaitIdle();
};
```

`WaitIdle` は明示的shutdown/debug用途で、通常frame pathでは使用しません。

### 7.3 Capability snapshot

保存項目:

- `VkPhysicalDeviceProperties2`
- `VkPhysicalDeviceVulkan11Properties`
- `VkPhysicalDeviceVulkan12Properties`
- `VkPhysicalDeviceVulkan13Properties`
- memory properties
- queue family properties
- limits: alignment、push constants、descriptor counts、workgroup size/count
- required format properties (`VkFormatProperties3`)
- optional debug utils availability

Deviceで実際にenabledされたfeatureはVulkanから照会できないため、`enabledFeatures`は利用者契約です。support query結果とenabled宣言を両方検証します。
## 8. QueueとTimeline設計

### 8.1 SyncPoint

```cpp
class VulkanSyncPoint final {
public:
    VulkanSyncPoint() noexcept = default;

    explicit operator bool() const noexcept;
    VkSemaphore Semaphore() const noexcept;
    uint64_t Value() const noexcept;

    bool IsComplete() const;
    bool Wait(std::chrono::nanoseconds timeout) const;
    void Wait() const;

    static VulkanSyncPoint FromBorrowedTimeline(
        std::shared_ptr<VulkanExecutionContext> context,
        VkSemaphore timelineSemaphore,
        uint64_t value);

private:
    std::shared_ptr<const Detail::TimelineState> m_state;
    uint64_t m_value = 0;
};
```

`TimelineState`はDevice dispatch、semaphore、Device handle、ownership flagを保持します。Queueが生成したpointではsemaphoreを所有して延命し、`FromBorrowedTimeline`ではdestroyしません。borrowed semaphoreは同じ`VkDevice`由来であることを呼び出し側契約とし、pointを参照するPool/DeferredRelease itemが完了・破棄されるまで外部ownerが保持します。binary semaphoreはcounter queryできないため直接SyncPoint化せず、command bufferなしの`VulkanQueue::Submit`へbinary waitを渡し、library timeline signalへbridgeします。

### 8.2 Submit descriptor

```cpp
struct VulkanSemaphoreWaitDesc {
    VkSemaphore semaphore = VK_NULL_HANDLE;
    uint64_t value = 0; // binaryの場合0
    VkPipelineStageFlags2 stageMask = VK_PIPELINE_STAGE_2_ALL_COMMANDS_BIT;
    bool timeline = true;
};

struct VulkanSemaphoreSignalDesc {
    VkSemaphore semaphore = VK_NULL_HANDLE;
    uint64_t value = 0;
    VkPipelineStageFlags2 stageMask = VK_PIPELINE_STAGE_2_ALL_COMMANDS_BIT;
    bool timeline = true;
};

struct VulkanQueueSubmitDesc {
    ArrayView<const VkCommandBuffer> commandBuffers;
    ArrayView<const VulkanSemaphoreWaitDesc> waits;
    ArrayView<const VulkanSemaphoreSignalDesc> signals;
};
```

### 8.3 Queue

```cpp
class VulkanQueue final {
public:
    VulkanSyncPoint Submit(const VulkanQueueSubmitDesc& desc);
    void WaitFor(const VulkanSyncPoint& point,
                 VkPipelineStageFlags2 stageMask);
    void WaitIdle();

    VkQueue Get() const noexcept;
    uint32_t FamilyIndex() const noexcept;
    VkQueueFlags Capabilities() const noexcept;
};
```

Submit algorithm:

1. descriptor検証。
2. internal mutexまたは注入host lock callbackでQueue host accessをlock。
3. next timeline valueをoverflow checkして採番。
4. caller waits/signalsを `VkSemaphoreSubmitInfo` へ変換。
5. library timeline signalを必ず末尾に追加。
6. `vkQueueSubmit2`。
7. 成功した場合だけnext value commit。
8. Queue host accessをunlock。
9. SyncPoint返却。

submit失敗時、Command/Upload sliceの所有状態をcommitしません。queue lost/device lostは例外に含めます。

### 8.4 CPU/GPU同期の区別

- Queue mutex: host-side external synchronizationだけ。
- Barrier: 同一Queue内のexecution/memory dependency。
- Semaphore: submit間またはQueue間dependency。
- Timeline wait: CPU↔GPU completion。

mutex取得をresource visibilityの代わりにしてはいけません。
## 9. Command Context設計

### 9.1 状態機械

```text
Initial --Begin--> Recording --End--> Executable --Submit--> Pending
   ^                    |                 |                    |
   |                    +--Abort----------+                    |
   +---------------- Reset/Complete ---------------------------+
```

- Pending中のBegin/Resetは禁止。
- `TryRecycle()` はlast submit SyncPoint完了時のみ成功。
- `WaitAndRecycle()` は明示CPU wait。
- destructorでPendingならResourceをdeferred destroyへ渡し、無期限waitしない。

### 9.2 API

```cpp
enum class VulkanCommandContextState {
    Initial,
    Recording,
    Executable,
    Pending,
};

struct VulkanCommandBeginDesc {
    bool oneTimeSubmit = true;
};

class VulkanCommandContext final {
public:
    void Begin(const VulkanCommandBeginDesc& desc = {});
    void End();
    void Abort() noexcept;

    VkCommandBuffer Get() const noexcept;
    VulkanCommandContextState State() const noexcept;

    VulkanSyncPoint Submit(VulkanQueue& queue,
                           ArrayView<const VulkanSemaphoreWaitDesc> waits = {});

    bool TryRecycle();
    void WaitAndRecycle();
};
```

owner thread IDはdebug buildで保存し、BeginからSubmitまでのrecording epochを1 threadへ拘束します。`AdoptCurrentThread()`はInitial state時だけ許可し、Recording/Executable/Pending中の所有thread変更は禁止します。Executorのslot poolは、completion確認とRecycleでInitialへ戻した後だけ別threadへadoptします。
