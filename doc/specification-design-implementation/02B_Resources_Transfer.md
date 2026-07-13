# VulkanGpuSupport アーキテクチャ・詳細設計書 — Part B: Resource・Barrier・Descriptor・Upload

文書ID: VGS-DESIGN-001  
版: 0.1-draft  
基準日: 2026-07-13

> 設計書index: [02_Architecture_Design.md](02_Architecture_Design.md)

---

## 10. ResourceとMemory設計

### 10.1 Memory class

```cpp
enum class VulkanMemoryClass {
    DeviceLocal,
    Upload,
    Readback,
};
```

VMA mapping:

- DeviceLocal: `VMA_MEMORY_USAGE_AUTO_PREFER_DEVICE`
- Upload: `VMA_MEMORY_USAGE_AUTO_PREFER_HOST` + sequential write + mapped
- Readback: `VMA_MEMORY_USAGE_AUTO_PREFER_HOST` + random access + mapped

exact flagはVMA版の推奨へ合わせ、public APIにVMA flagを必須化しません。高度利用者向けにoptional native allocation overrideを別descへ追加できます。

### 10.2 Buffer

```cpp
struct VulkanBufferDesc {
    VkDeviceSize size = 0;
    VkBufferUsageFlags usage = 0;
    VulkanMemoryClass memoryClass = VulkanMemoryClass::DeviceLocal;
    bool persistentlyMapped = false;
    VkSharingMode sharingMode = VK_SHARING_MODE_EXCLUSIVE;
    ArrayView<const uint32_t> queueFamilyIndices;
    const char* debugName = nullptr;
};

class VulkanBuffer final {
public:
    static VulkanBuffer Create(
        const std::shared_ptr<VulkanExecutionContext>& context,
        const VulkanBufferDesc& desc);

    VkBuffer Get() const noexcept;
    VkDeviceSize Size() const noexcept;
    void* MappedData() const noexcept;

    void Flush(VkDeviceSize offset, VkDeviceSize size);
    void Invalidate(VkDeviceSize offset, VkDeviceSize size);
};
```

- Flush/Invalidate rangeは内部でnonCoherentAtomSizeへexpand。
- HOST_COHERENTでも合法なno-opとして扱う。
- map/unmapの同時操作を許さない。

### 10.3 Image

```cpp
struct VulkanImageDesc {
    VkImageType type = VK_IMAGE_TYPE_2D;
    VkFormat format = VK_FORMAT_UNDEFINED;
    VkExtent3D extent = {0, 0, 1};
    uint32_t mipLevels = 1;
    uint32_t arrayLayers = 1;
    VkSampleCountFlagBits samples = VK_SAMPLE_COUNT_1_BIT;
    VkImageTiling tiling = VK_IMAGE_TILING_OPTIMAL;
    VkImageUsageFlags usage = 0;
    VkImageCreateFlags flags = 0;
    VkSharingMode sharingMode = VK_SHARING_MODE_EXCLUSIVE;
    ArrayView<const uint32_t> queueFamilyIndices;
    const char* debugName = nullptr;
};
```

- creation initial layoutは`UNDEFINED`。
- DeviceLocal allocation。
- format featureとusageの整合を検証。
- `VulkanImage`にmutable state fieldを持たせない。

### 10.4 Image View

```cpp
struct VulkanImageViewDesc {
    VkImageViewType viewType = VK_IMAGE_VIEW_TYPE_2D;
    VkFormat format = VK_FORMAT_UNDEFINED; // undefinedならimage format
    VkComponentMapping components = {};
    VkImageSubresourceRange range = {};
    const char* debugName = nullptr;
};

class VulkanImageView final { /* owns VkImageView */ };

struct VulkanImageViewRef {
    VkImage image = VK_NULL_HANDLE;
    VkImageView view = VK_NULL_HANDLE;
    VkFormat format = VK_FORMAT_UNDEFINED;
    VkExtent3D extent = {};
    VkImageSubresourceRange range = {};
    uint32_t queueFamilyIndex = VK_QUEUE_FAMILY_IGNORED;
};
```

`VulkanImageView`は `VkImageView` のみを所有し、元の `VkImage` は借用します。元ImageはViewより長く生存させます。`VulkanImageViewRef`はImageもViewも延命しません。
## 11. Logical Image Bundle

### 11.1 Pixel Format

```cpp
enum class PixelFormat {
    Undefined,
    R8Unorm,
    RG8Unorm,
    RGBA8Unorm,
    BGRA8Unorm,
    R16Unorm,
    RG16Unorm,
    RGBA16Float,
    NV12,
    P010,
};
```

helper:

```cpp
uint32_t GetPlaneCount(PixelFormat format);
VkFormat GetPlaneVkFormat(PixelFormat format, uint32_t plane);
VkExtent2D GetPlaneExtent(PixelFormat format, VkExtent2D full, uint32_t plane);
uint32_t GetPlaneBytesPerTexel(PixelFormat format, uint32_t plane);
```

### 11.2 Bundle

```cpp
struct VulkanImageBundleDesc {
    PixelFormat format = PixelFormat::Undefined;
    VkExtent2D extent = {};
    VkImageUsageFlags usage = 0;
    const char* debugName = nullptr;
};

struct VulkanImagePlane {
    VulkanImage image;
    VulkanImageView defaultView;
};

struct VulkanImageBundleView {
    PixelFormat format = PixelFormat::Undefined;
    VkExtent2D extent = {};
    std::array<VulkanImageViewRef, 3> planes = {};
    uint32_t planeCount = 0;

    explicit operator bool() const noexcept;
    const VulkanImageViewRef& Plane(uint32_t index) const;
};

class VulkanImageBundle final {
public:
    static VulkanImageBundle Create(
        const std::shared_ptr<VulkanExecutionContext>& context,
        const VulkanImageBundleDesc& desc);

    PixelFormat Format() const noexcept;
    VkExtent2D Extent() const noexcept;
    uint32_t PlaneCount() const noexcept;
    const VulkanImagePlane& Plane(uint32_t index) const;
    VulkanImageBundleView View() const noexcept;
};
```

NV12/P010:

```text
full extent = W x H
plane 0 = W x H
plane 1 = (W/2) x (H/2)
```

UV planeの1 texelがU/V pairです。`VulkanImageBundleView`は一切のhandleを所有せず、元のImage/View/Deviceを延命しません。Record/Submit完了までのlifetimeは呼び出し側が保証します。
## 12. Resource StateとBarrier

### 12.1 State型

```cpp
struct VulkanImageState {
    VkPipelineStageFlags2 stageMask = VK_PIPELINE_STAGE_2_NONE;
    VkAccessFlags2 accessMask = VK_ACCESS_2_NONE;
    VkImageLayout layout = VK_IMAGE_LAYOUT_UNDEFINED;
    uint32_t queueFamilyIndex = VK_QUEUE_FAMILY_IGNORED;
};

struct VulkanBufferState {
    VkPipelineStageFlags2 stageMask = VK_PIPELINE_STAGE_2_NONE;
    VkAccessFlags2 accessMask = VK_ACCESS_2_NONE;
    uint32_t queueFamilyIndex = VK_QUEUE_FAMILY_IGNORED;
};

struct VulkanImageBundleState {
    std::array<VulkanImageState, 3> planes = {};
    uint32_t planeCount = 0;
};
```

定数helper:

- `ImageStateUndefined()`
- `ImageStateTransferDst(family)`
- `ImageStateComputeSampledRead(family)`
- `ImageStateComputeStorageWrite(family)`
- `ImageStateGeneralReadWrite(family)`
- `BufferStateHostWrite()`
- `BufferStateTransferRead(family)`

### 12.2 Barrier API

```cpp
void RecordImageBarrier(
    VkCommandBuffer commandBuffer,
    const VulkanImageViewRef& image,
    const VulkanImageState& before,
    const VulkanImageState& after);

void RecordBufferBarrier(
    VkCommandBuffer commandBuffer,
    VkBuffer buffer,
    VkDeviceSize offset,
    VkDeviceSize size,
    const VulkanBufferState& before,
    const VulkanBufferState& after);

class VulkanBarrierBatch final {
public:
    void AddImage(...);
    void AddBuffer(...);
    void AddMemory(...);
    void Record(VkCommandBuffer commandBuffer) const;
};
```

規約:

- `vkCmdPipelineBarrier2`だけを使用。
- no-op barrierは省略可能。
- queue familyが異なる場合、ownership transfer pairが必要。v1.0 high-level APIは異なるfamilyを拒否。
- `UNDEFINED` beforeは内容破棄を意味するため、既存内容を読む経路で使用禁止。
- aspect maskはformatから推論しても、最終的に明示rangeを保存。
## 13. Descriptor Arena

### 13.1 目的

Processing dispatchごとのDescriptor Setを安全に割り当て、GPU完了後にPool resetで一括再利用します。

### 13.2 API

```cpp
struct VulkanDescriptorArenaDesc {
    uint32_t maxSets = 256;
    std::vector<VkDescriptorPoolSize> poolSizes;
    VkDescriptorPoolCreateFlags flags = 0;
    const char* debugName = nullptr;
};

class VulkanDescriptorArena final {
public:
    VkDescriptorSet Allocate(VkDescriptorSetLayout layout);

    void RetireAfter(const VulkanSyncPoint& point);
    bool TryReset();
    void WaitAndReset();

    bool IsRetired() const noexcept;
};
```

- thread-confined。
- Recording中にallocate。
- submit後 `RetireAfter`。
- Point完了前のreset禁止。
- Descriptor Set個別freeは標準経路で使わない。
## 14. Shader / Compute Pipeline

### 14.1 固定binding ABI

v1.0 set 0:

| binding | type | 用途 |
|---:|---|---|
| 0 | sampled image | source plane 0 / packed source |
| 1 | sampled image | source plane 1、packed時dummy |
| 2 | storage image | destination |
| 3 | sampler | pointまたはlinear |

push constantsは処理別構造体を使用し、最低保証128 bytes以内に収めます。

### 14.2 Processorのmutable state排除

Processorが保持可能:

- shared ProcessingContext
- immutable pipeline key
- immutable validation rules

Processorが保持してはいけない:

- dispatchごとに書き換えるDescriptor Set
- caller Command Buffer
- caller Resource state
- temporary upload memory
- pending SyncPoint

### 14.3 Pipeline cache

`VulkanProcessingContext` 内部のthread-safe mapでpipelineをlazy生成します。double-checked accessではなくmutexで正しく構築します。生成後のpipeline objectはimmutable。
## 15. CPU Image View

```cpp
struct CpuImagePlaneView {
    const std::byte* data = nullptr;
    size_t sizeBytes = 0;
    uint32_t rowStrideBytes = 0;
    uint32_t width = 0;
    uint32_t height = 0;
};

struct CpuImageView {
    PixelFormat format = PixelFormat::Undefined;
    uint32_t width = 0;
    uint32_t height = 0;
    std::array<CpuImagePlaneView, 3> planes = {};
    uint32_t planeCount = 0;
};

void ValidateCpuImageView(const CpuImageView& view);
```

検証:

- null、zero size、overflow
- formatのplane count一致
- plane dimensions一致
- row stride >= minimum bytes
- `sizeBytes >= rowStride * (height - 1) + activeRowBytes`
- NV12/P010偶数extent
- P010は16-bit word格納。上位10bit利用という色解釈はshader側で処理
## 16. Upload Ring設計

### 16.1 Backing buffer

- VMA Upload memory
- `VK_BUFFER_USAGE_TRANSFER_SRC_BIT`
- persistently mapped
- capacityはdesc指定
- default 64 MiB（sample値。library固定値ではなくdesc default）
- alignmentは少なくとも4 bytes、`optimalBufferCopyOffsetAlignment`、format要件を満たす
- flush rangeは`nonCoherentAtomSize`へalign

### 16.2 API

```cpp
struct VulkanUploadRingDesc {
    VkDeviceSize capacity = 64ull * 1024ull * 1024ull;
    bool allowBlockingAcquire = true;
    const char* debugName = nullptr;
};

class VulkanUploadSlice final {
public:
    std::byte* Data() const noexcept;
    VkBuffer Buffer() const noexcept;
    VkDeviceSize Offset() const noexcept;
    VkDeviceSize Size() const noexcept;
};

class VulkanUploadRing final {
public:
    std::optional<VulkanUploadSlice>
    TryAcquire(VkDeviceSize size, VkDeviceSize alignment);

    std::optional<VulkanUploadSlice>
    Acquire(VkDeviceSize size,
            VkDeviceSize alignment,
            std::chrono::nanoseconds timeout);

    void CollectCompleted();
    VulkanUploadRingStats Stats() const;
};
```

`VulkanUploadSlice`は直接Ringへcommitしません。`VulkanUploadBatch`がsliceを所有し、submit成功後にRingへretireします。

### 16.3 Ring algorithm

管理値:

- monotonic virtual head/tail（64-bit）
- physical offset = virtual % capacity
- active allocation queue（取得順）
- allocation state: Reserved / Recorded / Retired(point) / Complete / Cancelled

手順:

1. mutex取得。
2. pending pointをpollしてtailを前進。
3. checked alignmentでhead候補を計算。
4. physical endを越える場合、padding reservationを入れてwrap。
5. `head + requested - tail <= capacity` を確認。
6. slice reservation作成。
7. mutex解放。
8. callerがCPU copy。
9. Batchがflushしcopy command記録。
10. submit成功後、reservationへSyncPoint設定。
11. point完了後tail前進。

reservationがBatch submit前に破棄された場合Cancelledとして回収します。取得順に穴があっても先頭が完了するまでtailを越えません。

### 16.4 Row repack

v1.0は入力strideをそのまま `bufferRowLength` に変換せず、active pixelだけをtight stagingへ行単位copyします。

利点:

- 任意camera/decoder strideを吸収
- texel単位換算のformat制約を単純化
- plane offset alignmentを明示管理

欠点はCPU copy量ですが、正しさを優先します。測定後にdirect stride pathを追加できます。
## 17. Upload Batch / Executor

### 17.1 Batch

```cpp
struct VulkanUploadBufferDesc {
    ArrayView<const std::byte> source;
    VkBuffer destination = VK_NULL_HANDLE;
    VkDeviceSize destinationOffset = 0;
    VkDeviceSize destinationSize = 0;
    VulkanBufferState before;
    VulkanBufferState after;
};

struct VulkanUploadImageDesc {
    CpuImageView source;
    VulkanImageBundleView destination;
    VulkanImageBundleState before;
    VulkanImageBundleState after;
};

class VulkanUploadBatch final {
public:
    VulkanUploadBatch(VulkanCommandContext& command,
                      VulkanUploadRing& ring);

    void RecordBuffer(const VulkanUploadBufferDesc& desc);
    void RecordImage(const VulkanUploadImageDesc& desc);
    VulkanSyncPoint Submit(VulkanQueue& queue,
                           ArrayView<const VulkanSemaphoreWaitDesc> waits = {});
    void Cancel() noexcept;
};
```

RecordBuffer:

1. source/destination range、zero size、offset+size overflowを検証。
2. Ring slice取得、CPU byte copy、non-coherent range flush。
3. before→TransferWrite用buffer barrier。
4. `vkCmdCopyBuffer2`を記録。
5. TransferWrite→after barrier。
6. sliceをBatch pending listへ保持。

RecordImage:

1. source/destination validation。
2. 各plane tight byte size計算。
3. Ring slices取得。
4. row copy。
5. flush。
6. before→TransferDst barrier。
7. `vkCmdCopyBufferToImage2` planeごとに記録。
8. TransferDst→after barrier。
9. sliceをBatch pending listへ保持。

Submit成功後に全sliceを同じcompletion pointへretireします。失敗時はGPU参照が成立していないためcancel可能ですが、Device lostなど不明状態はquarantineします。

### 17.2 Executor

```cpp
struct VulkanUploadExecutorDesc {
    VulkanUploadRingDesc ring = {};
    uint32_t initialCommandSlots = 2;
    uint32_t maxCommandSlots = 8;
    bool blockWhenAllCommandSlotsPending = true;
    const char* debugName = nullptr;
};

class VulkanUploadExecutor final {
public:
    static std::unique_ptr<VulkanUploadExecutor> Create(
        std::shared_ptr<VulkanExecutionContext> context,
        const VulkanUploadExecutorDesc& desc = {});

    VulkanSyncPoint Upload(const VulkanUploadBufferDesc& desc);
    VulkanSyncPoint Upload(const VulkanUploadImageDesc& desc);
};
```

- Thread-safe convenience。
- Executorは`CommandSlot { VulkanCommandContext, lastPoint, state }`をpoolします。slotは1 callerへ排他的に貸し出し、Recording/Submit中に共有しません。
- 呼出し開始時にcompleted slotをcollectし、`TryRecycle`後にInitial状態へ戻してcurrent threadへadoptします。free slotがなければ`maxCommandSlots`まで増加し、上限時はdescriptor方針に従って最古pointを待つかexhaustionを返します。
- Queue submitは`VulkanQueue`が直列化し、各slotのCommand Poolは独立しているため、複数callerのcommand recordingは並列化できます。
- high-throughput/明示制御用途は各threadでBatch/Command Contextを直接持てます。
- Executorはworker threadやカメラthread poolを内蔵しません。CPU source bytesは`Upload`呼出し中にstagingへcopyされます。
