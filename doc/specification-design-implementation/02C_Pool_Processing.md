# VulkanGpuSupport アーキテクチャ・詳細設計書 — Part C: Pool・Lifetime・Processing

文書ID: VGS-DESIGN-001  
版: 0.1-draft  
基準日: 2026-07-13

> 設計書index: [02_Architecture_Design.md](02_Architecture_Design.md)

---

## 18. Image Pool / Lease設計

### 18.1 Pool descriptor

```cpp
enum class VulkanPoolExhaustionPolicy {
    ReturnEmpty,
    Block,
};

struct VulkanImagePoolDesc {
    VulkanImageBundleDesc image;
    VulkanImageBundleState initialState;
    size_t initialCapacity = 0;
    size_t maxCapacity = 0;
    VulkanPoolExhaustionPolicy exhaustionPolicy =
        VulkanPoolExhaustionPolicy::ReturnEmpty;
    const char* debugName = nullptr;
};
```

`maxCapacity == 0` はinitialCapacityと同値ではなく、明示的に「無制限」を許可しません。0はinvalidとします。OOM暴走を防ぐためmax必須。内部で新規作成したImageの最初の実状態は常に`UNDEFINED`です。そのためv1.0では`initialState`の全planeが`layout=UNDEFINED`、stage/access=`NONE`であることを要求し、それ以外はrejectします。任意layoutへの初期transitionをPoolが暗黙実行してはなりません。

### 18.2 Slot state

```text
Free
  | Acquire
  v
Leased
  | ReleaseAfter(point,state)
  v
Pending ---------------------+
  | point complete           |
  v                          |
Free                         |
                             |
Leased -- ReleaseNow ------> Free
  |
  +-- destructor without retire --> Quarantined
```

Slot metadata:

- owned `VulkanImageBundle`
- explicit last completed/declared `VulkanImageBundleState`
- state enum
- retire SyncPoint
- generation counter
- optional diagnostic string/thread ID

### 18.3 Lease API

```cpp
class VulkanImageLease final {
public:
    VulkanImageLease(VulkanImageLease&&) noexcept;
    VulkanImageLease& operator=(VulkanImageLease&&) noexcept;
    ~VulkanImageLease() noexcept;

    explicit operator bool() const noexcept;
    VulkanImageBundleView Image() const noexcept;
    const VulkanImageBundleState& AcquiredState() const noexcept;
    uint64_t Generation() const noexcept;

    void ReleaseNow(const VulkanImageBundleState& finalState);
    void ReleaseAfter(const VulkanSyncPoint& point,
                      const VulkanImageBundleState& finalState);
};
```

- copy禁止。
- Release後invalid。
- `ReleaseNow` はGPUへ投入していないか、完了が既に保証される場合のみ。
- `ReleaseAfter` はsubmit成功後に呼ぶ。
- destructorでactiveならquarantineし、diagnostic callback。
- move代入先にactive leaseがあれば先に安全なquarantine処理。

### 18.4 Pool API

```cpp
class VulkanImagePool final {
public:
    static std::shared_ptr<VulkanImagePool> Create(...);

    std::optional<VulkanImageLease> TryAcquire();
    std::optional<VulkanImageLease>
    Acquire(std::chrono::nanoseconds timeout);

    void CollectCompleted();
    VulkanImagePoolStats Stats() const;
};
```

Pool internal stateは `shared_ptr` でLeaseと共有します。public Pool objectが破棄されても、outstanding Leaseがstateを延命します。新規Acquireは停止します。

### 18.5 完了poll

1. mutex内でPending slotのpoint snapshotを取る。
2. mutex外で `IsComplete` を呼ぶ。
3. mutexへ戻りgeneration/pointが同じことを確認。
4. completed slotをFreeへ移動。
5. condition variable notify。

Vulkan function call中にPool mutexを長時間保持しません。

### 18.6 Blocking Acquire

- Free slotがあれば即返す。
- capacity未満なら新規slot作成。ただし作成はlock外で行い、reservation countで上限競合を防ぐ。
- 上限時は最古Pending pointを待機候補にする。
- timeoutまでcondition variableまたはtimeline wait。
- カメラ固有のdrop/oldest置換は実装しない。利用者が `TryAcquire` 結果で判断。
## 19. Deferred Release Queue

```cpp
class VulkanDeferredReleaseQueue final {
public:
    template<class T>
    void Enqueue(T&& object, const VulkanSyncPoint& point);

    void CollectCompleted();
    void Drain();
    size_t PendingCount() const;
};
```

- Tはmove constructible、destructor noexceptを要求。
- type erasureはinternal nodeで実装。
- callbackはmutex外で破棄。
- `Drain()`は明示CPU wait。
- context destructor前にDrainを推奨。
## 20. Processing型

### 20.1 共通値型

```cpp
struct ProcessingRect {
    int32_t x = 0;
    int32_t y = 0;
    uint32_t width = 0;
    uint32_t height = 0;
};

enum class ProcessingFilter { Point, Linear };
enum class ProcessingColorMatrix { Identity, BT601, BT709, BT2020 };
enum class ProcessingColorRange { Full, Limited };

struct ProcessingColorDesc {
    ProcessingColorMatrix sourceMatrix = ProcessingColorMatrix::BT709;
    ProcessingColorRange sourceRange = ProcessingColorRange::Full;
};
```

### 20.2 Resource binding

```cpp
struct VulkanProcessingImage {
    VulkanImageBundleView image;
    VulkanImageBundleState before;
    VulkanImageBundleState after;
};
```

### 20.3 Descriptors

```cpp
struct FormatConvertDesc {
    ProcessingColorDesc color = {};
    ProcessingRect sourceRect = {};
    ProcessingRect destinationRect = {};
};

struct ResizeDesc {
    ProcessingFilter filter = ProcessingFilter::Linear;
    ProcessingRect sourceRect = {};
    ProcessingRect destinationRect = {};
};

struct FusedConvertResizeDesc {
    ProcessingColorDesc color = {};
    ProcessingFilter filter = ProcessingFilter::Linear;
    ProcessingRect sourceRect = {};
    ProcessingRect destinationRect = {};
};
```

zero rectは「full extent」を意味するconvenienceにできますが、Normalize関数で実値へ変換し、validation/constantにはzeroを残しません。
## 21. Processing Context / Processor

### 21.1 Context

```cpp
struct VulkanProcessingContextDesc {
    VkPipelineCache externalPipelineCache = VK_NULL_HANDLE;
    const char* debugName = nullptr;
};

class VulkanProcessingContext final {
public:
    static std::shared_ptr<VulkanProcessingContext>
    Create(std::shared_ptr<VulkanExecutionContext> context,
           const VulkanProcessingContextDesc& desc = {});

    VkDescriptorSetLayout DescriptorSetLayout() const noexcept;
    VkPipelineLayout PipelineLayout() const noexcept;
};
```

所有:

- descriptor set layout
- pipeline layout
- point/linear sampler
- internal pipeline cache（externalがなければ）
- thread-safe pipeline registry
- shader registry reference

### 21.2 Processor API

```cpp
class VulkanFormatConverter final {
public:
    explicit VulkanFormatConverter(
        std::shared_ptr<VulkanProcessingContext> context);

    void RecordConvert(
        VulkanCommandContext& command,
        VulkanDescriptorArena& descriptors,
        const VulkanProcessingImage& source,
        const VulkanProcessingImage& destination,
        const FormatConvertDesc& desc) const;
};

class VulkanResizer final {
public:
    void RecordResize(..., const ResizeDesc& desc) const;
};

class VulkanFusedProcessor final {
public:
    void RecordConvertResize(...,
        const FusedConvertResizeDesc& desc) const;
};
```

### 21.3 Record sequence

1. commandがRecordingか検証。
2. source/destination bundle/format/extent/rectを検証。
3. format featureとusageを検証。
4. source before→ComputeSampledRead barrier。
5. destination before→ComputeStorageWrite/GENERAL barrier。
6. Descriptor Set allocate/update。
7. pipeline bind。
8. descriptor bind。
9. push constants。
10. dispatch count checked calculation。
11. source working→source after barrier。
12. destination working→destination after barrier。

source afterがworking stateと同じならbarrier省略可。destination visibilityが後続処理に必要ならaccess/stageを正しく含めます。

### 21.4 Dispatch dimensions

- workgroup sizeはshaderごとに固定（例 16x16x1）。
- `ceil(dstWidth / groupX)` をchecked arithmeticで計算。
- physical device `maxComputeWorkGroupCount`を超えたらreject。
- destination rect外へ書き込まない。

### 21.5 Resize座標規約

source/destination rectangleは左上原点のhalf-open pixel rectangleです。destination local pixel centerからsource continuous coordinateを次で求めます。

```text
srcCoord = srcOrigin + ((dstLocal + 0.5) * srcSize / dstSize) - 0.5
```

- Point: `floor(srcCoord + 0.5)` をsource rectangle内へclamp。
- Linear: `floor(srcCoord)` とfractionで4 tap bilinearし、各tapをsource rectangle内へclamp。
- 1:1、同一originではpixel centerが完全一致。
- shaderとCPU goldenは同じ式を独立実装し、test vectorで一致させます。

### 21.6 YUV変換

- NV12: Yは8-bit、UVはinterleaved 8-bit。normalized codeはY=`code/255`、full-range C=`(code-128)/255`。
- P010: 16-bit wordの上位10bitがcodeで、概念上 `code = word >> 6`。`R16_UNORM` sampling値`sample`からは `codeNorm = clamp(sample * 65535 / (64 * 1023), 0, 1)` と等価に復元します。full-range Cは`(code-512)/1023`。
- Limited 8-bit: Y=`(code-16)/219`、C=`(code-128)/224`。
- Limited 10-bit: Y=`(code-64)/876`、C=`(code-512)/896`。入力範囲外は変換後にclampします。
- matrixはnon-constant-luminanceの`Kr/Kb`を使用します。BT.601=`0.299/0.114`、BT.709=`0.2126/0.0722`、BT.2020=`0.2627/0.0593`。`Kg=1-Kr-Kb`とし、R/G/Bを標準Y'CbCr式から算出します。
- chroma sampling positionはv1.0でcentered 4:2:0。luma center coordinate `L` に対するchroma plane coordinateは `C=(L-0.5)/2` とし、edge clampします。異なるchroma sitingはfuture descです。
- output alphaは1.0、最終RGBはoutput formatへ量子化する前に`[0,1]`へclampします。
- shaderとCPU goldenで同じ係数・range・座標契約を使用し、magic coefficientの別管理を禁止します。
