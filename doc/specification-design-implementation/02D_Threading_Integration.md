# VulkanGpuSupport アーキテクチャ・詳細設計書 — Part D: Diagnostics・Threading・統合契約

文書ID: VGS-DESIGN-001  
版: 0.1-draft  
基準日: 2026-07-13

> 設計書index: [02_Architecture_Design.md](02_Architecture_Design.md)

---

## 22. Diagnostics

### 22.1 Callback

```cpp
enum class VulkanDiagnosticLevel {
    Info,
    Warning,
    Error,
};

using VulkanDiagnosticCallback = void(*)(
    VulkanDiagnosticLevel level,
    const char* category,
    const char* message,
    void* userData) noexcept;
```

- callbackはoptional。
- libraryはstdout/stderrへ勝手に出力しない。
- callback中にlibraryへ再入可能とは保証しない。
- callback exceptionは禁止。

### 22.2 Object names

Debug Utils functionがdispatch tableに存在する場合だけ `vkSetDebugUtilsObjectNameEXT`。存在しなければno-op。
## 23. Thread safety詳細

### 23.1 Lock hierarchy

deadlock防止の順序:

```text
1. Global Loader initialization lock
2. Context registry/cache lock
3. Queue mutex
4. Pool/UploadRing mutex
5. Diagnostic callback（lockを解放してから）
```

原則として2つのsubsystem mutexを同時保持しません。Queue submit中にPool mutexを保持しない、Pool completion query中にQueue mutexを取得しない構成にします。

### 23.2 Thread-localにするもの

- Command Context
- Descriptor Arena
- Upload Batch
- temporary vectors/builders

共有しないことでVulkan external synchronizationを構造的に満たします。

### 23.3 Shared immutable

- Context capability snapshot
- Shader registry
- Pipeline/Shader key
- Processor object
- completed SyncPoint
## 24. Device lost

- `VK_ERROR_DEVICE_LOST`を専用判定できるようExceptionに保持。
- Device lost後、追加submit/allocateを試みないcontext fault stateを設定。
- Pool/Upload Ringはpending完了を待てないためquarantine。
- destructorは通常destroyを試みるが、無期限waitしない。
- recovery/Device再生成は上位の責務。
## 25. Camera libraryとの境界（非規範利用例）

```text
Camera SDK CPU frame
   -> Camera側でCpuImageView作成
   -> VulkanImagePool::TryAcquire
   -> VulkanUploadBatch::RecordImage
   -> VulkanFusedProcessor::RecordConvertResize
   -> submit / ReadyPoint
   -> Camera側VulkanFrame { lease, timestamp, sequence }
```

Camera側に残すもの:

- timestamp/sequence/device ID
- capture thread
- reconnect
- pool枯渇時drop/wait policy
- camera pixel negotiation

VulkanGpuSupportへ入れるもの:

- CpuImageView validation
- staging upload
- GPU image/pool/lease
- processing
- synchronization
## 26. 禁止設計

Codex/実装者は次を行ってはいけません。

1. `VkDevice`をVulkanGpuSupport本体が勝手に生成する。
2. global `volkLoadDevice` dispatchへ依存する。
3. VMAとは別のsuballocatorを実装する。
4. `VulkanImage::CurrentLayout`のような単一暗黙stateを追加する。
5. Command Poolを全threadで共有してmutexだけで隠す。
6. Processor内に再利用Descriptor Setを1つだけ保持する。
7. Pool Lease destructorでGPU完了不明ResourceをFreeへ戻す。
8. `vkDeviceWaitIdle`を通常Upload/Processing pathで呼ぶ。
9. runtime shader compileを必須化する。
10. カメラ型/SDK headerをpublic moduleへincludeする。
11. JSON設定/Boost/ThreadToolkitを導入する。
12. Validation errorをtest側でfilterして成功扱いする。
## 27. API umbrella headers

```cpp
// VulkanGpuSupport.hpp
#include <VulkanGpuSupport/Foundation/VulkanFoundation.hpp>
#include <VulkanGpuSupport/Core/VulkanCore.hpp>
#include <VulkanGpuSupport/Gpu/VulkanGpu.hpp>
#include <VulkanGpuSupport/Transfer/VulkanTransfer.hpp>
#include <VulkanGpuSupport/Pool/VulkanPool.hpp>
#include <VulkanGpuSupport/Processing/VulkanProcessing.hpp>
#include <VulkanGpuSupport/Diagnostics/VulkanDiagnostics.hpp>
```

各module umbrellaはそのmoduleのpublic headersだけをincludeし、逆方向依存を作りません。
## 28. 最小利用例

```cpp
using namespace VulkanGpuSupport;

VulkanExecutionContextDesc contextDesc{};
contextDesc.instance = instance;
contextDesc.physicalDevice = physicalDevice;
contextDesc.device = device;
contextDesc.primaryQueue = {
    queue,
    queueFamilyIndex,
    0,
    VK_QUEUE_COMPUTE_BIT | VK_QUEUE_TRANSFER_BIT,
};
contextDesc.enabledFeatures.timelineSemaphore = true;
contextDesc.enabledFeatures.synchronization2 = true;

auto context = VulkanExecutionContext::Create(contextDesc);

VulkanImagePoolDesc poolDesc{};
poolDesc.image.format = PixelFormat::RGBA8Unorm;
poolDesc.image.extent = {1920, 1080};
poolDesc.image.usage = VK_IMAGE_USAGE_TRANSFER_DST_BIT |
                       VK_IMAGE_USAGE_SAMPLED_BIT |
                       VK_IMAGE_USAGE_STORAGE_BIT;
poolDesc.initialState = MakeUndefinedBundleState(1);
poolDesc.initialCapacity = 4;
poolDesc.maxCapacity = 8;

auto pool = VulkanImagePool::Create(context, poolDesc);
auto uploader = VulkanUploadExecutor::Create(context);

auto lease = pool->TryAcquire();
if (!lease) {
    // 上位ライブラリがdrop/waitを決定する。
    return;
}

VulkanUploadImageDesc upload{};
upload.source = cpuImage;
upload.destination = lease->Image();
upload.before = lease->AcquiredState();
upload.after = MakeComputeSampledBundleState(queueFamilyIndex, 1);

VulkanSyncPoint ready = uploader->Upload(upload);
lease->ReleaseAfter(ready, upload.after);
```

実際のconsumerへLeaseを渡す場合は、consumerの最終submit pointまでLeaseを保持し、そのpointで `ReleaseAfter` します。Upload完了だけでPoolへ返すと、consumerが使用中に再利用されるため誤りです。

---

本設計書のAPI signatureは実装開始時の規範候補です。Vulkan仕様上の制約で修正が必要な場合、仕様書要求IDを維持し、設計書とtestを同時更新してください。
