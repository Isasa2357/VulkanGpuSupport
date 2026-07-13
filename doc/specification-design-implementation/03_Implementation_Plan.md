# VulkanGpuSupport 実装手順書

文書ID: VGS-PLAN-001  
版: 0.1-draft  
基準日: 2026-07-13

---

## 1. この手順書の使い方

本手順は、Codexまたは人間の実装者が、前提知識をチャットから補わずに実装するための作業分解です。Phaseは依存順です。各Phaseを原則1 PRにし、完了条件を満たしてから次へ進みます。

### 1.1 共通ルール

- 仕様変更をコードだけで行わない。
- public API追加時はheader standalone testを追加。
- Vulkan object追加時はownership、thread safety、GPU lifetimeをheader commentへ記載。
- Vulkan call追加時は失敗処理、debug name、validation testを追加。
- GPU testはValidation Layerを有効化し、Errorを0件にする。
- `vkDeviceWaitIdle`で同期不備を隠さない。
- non-Vulkan依存を追加しない。
- dependency versionをfloating branchへ向けない。基準pinはVulkan-Headers `vulkan-sdk-1.4.350.1`/`8864cdc`、volk `1.4.350`/`3ca312a`、VMA `v3.4.0`/`3aa9212`、DXC `v1.9.2602.24`/`d355aa8`、SPIRV-Tools `v2026.2`/`0539c81`、vk-bootstrap `v1.4.350`/`aee321c`。
- generated shaderだけ変更せず、HLSL sourceと生成情報を同時更新。

### 1.2 PR本文template

```markdown
## Purpose

## Requirements
- VGS-XXX-001

## Public API changes

## Ownership / synchronization notes

## Tests added

## Validation result

## Known limitations

## Documentation changes
```

### 1.3 Definition of Done共通項目

- Debug/Release build成功。
- CPU test成功。
- 対象PhaseにGPU testがあれば成功。
- clang-formatまたはproject format適用。
- warning level最大構成でlibrary warning 0。
- ASan/UBSan対象test成功（対応platform）。
- public commentと文書更新。
- 新規TODOにIssue番号または後続Phaseを記載。

## 2. 開発環境

### 2.1 Windows

推奨:

- Windows 11 x64
- Visual Studio 2022
- CMake 3.24+
- NinjaまたはVisual Studio generator
- Vulkan SDK（Vulkan 1.3 headers/loader/validation/spirv tools）
- DXC SPIR-V backend

構成例:

```bat
cmake -S . -B out/build/windows-debug -G Ninja ^
  -DCMAKE_BUILD_TYPE=Debug ^
  -DVGS_BUILD_TESTS=ON ^
  -DVGS_BUILD_SAMPLES=ON ^
  -DVGS_REBUILD_SHADERS=ON ^
  -DVGS_VALIDATE_SHADERS=ON

cmake --build out/build/windows-debug --parallel
ctest --test-dir out/build/windows-debug --output-on-failure
```

### 2.2 Linux

推奨:

- Ubuntu系または同等x86_64
- GCC 12+またはClang 16+
- CMake 3.24+
- Ninja
- Vulkan loader、validation layers、Vulkan SDK tools
- 実GPUdriverまたはlavapipe等のsoftware Vulkan 1.3 implementation

```bash
cmake -S . -B out/build/linux-debug -G Ninja \
  -DCMAKE_BUILD_TYPE=Debug \
  -DVGS_BUILD_TESTS=ON \
  -DVGS_BUILD_SAMPLES=ON \
  -DVGS_REBUILD_SHADERS=ON \
  -DVGS_VALIDATE_SHADERS=ON

cmake --build out/build/linux-debug --parallel
ctest --test-dir out/build/linux-debug --output-on-failure
```

### 2.3 Sanitizer構成

```bash
cmake -S . -B out/build/asan -G Ninja \
  -DCMAKE_BUILD_TYPE=Debug \
  -DVGS_BUILD_TESTS=ON \
  -DCMAKE_CXX_FLAGS="-fsanitize=address,undefined -fno-omit-frame-pointer" \
  -DCMAKE_EXE_LINKER_FLAGS="-fsanitize=address,undefined"
```

TSanはVulkan driver内部やloader由来の報告を分離し、CPU-only Pool/Queue mock testを主対象にします。

## 3. Phase 0 — Repository bootstrap

### 3.1 目的

Build、dependency、install、testの骨格を先に固定し、後続実装が場当たり的なCMakeにならないようにします。

### 3.2 作成ファイル

```text
CMakeLists.txt
cmake/Dependencies.cmake
cmake/VulkanGpuSupportConfig.cmake.in
include/VulkanGpuSupport/VulkanGpuSupport.hpp
include/VulkanGpuSupport/Foundation/Version.hpp
src/Placeholder.cpp
test/TestHarness.hpp
test/CMakeLists.txt
sample/CMakeLists.txt
README.md
LICENSE
THIRD_PARTY.md
```

### 3.3 作業

1. C++17、CMake 3.24、STATIC library targetを定義。
2. `VulkanGpuSupport::VulkanGpuSupport` aliasを定義。
3. build/install interface include pathを設定。
4. warnings:
   - MSVC: `/W4 /permissive- /utf-8 /EHsc`
   - GCC/Clang: `-Wall -Wextra -Wpedantic -Wconversion`を段階導入
5. optionsを仕様どおり追加。
6. Vulkan-Headers、volk、VMAのpackage discoveryを実装。
7. `VGS_FETCH_DEPENDENCIES=ON`時だけ上記固定tag/commitのFetchContentを使用。
8. install/export/config/version fileを実装。
9. `add_subdirectory` smoke executableを追加。
10. install tree `find_package` smoke testを追加。
11. public header compile testを追加。
12. CI skeletonを追加してもよいが、認証情報やrelease publishはまだ行わない。

### 3.4 Dependency pin作業

- official repository/releaseを使用。
- README/仕様書の固定tag/commitを初期値として使用。
- commit SHAを記録。
- licenseを `THIRD_PARTY.md` に記録。
- volk/VMAのtarget aliasをcanonical名へ統一。
- Vulkan-Hppを導入しない。

### 3.5 Tests

- external frameworkを使わない軽量`TestHarness.hpp`のself-test。
- `Version.hpp` include。
- root umbrella include。
- add_subdirectory consumer。
- install/find_package consumer。

### 3.6 完了条件

- 実装がplaceholderでもWindows/Linuxでlibrary生成。
- networkなしのsystem dependency pathが成立。
- FetchContent pathは明示opt-in。
- install packageからtargetをlink可能。

## 4. Phase 1 — Foundation

### 4.1 対象要求

VGS-FND-001〜005。

### 4.2 作成

- `ArrayView.hpp`
- `Error.hpp/.cpp`
- `CheckedMath.hpp`
- `PixelFormat.hpp/.cpp`
- `Geometry.hpp`
- `VulkanFoundation.hpp`

### 4.3 実装詳細

#### ArrayView

C++17で `std::span` の代替となるread-only viewを提供します。

- pointer + size
- vector/array/C array constructor
- bounds checked `At`
- unchecked `operator[]`
- empty/null整合
- lifetimeを所有しないことをcomment

#### CheckedMath

- `CheckedAdd`
- `CheckedMultiply`
- `AlignUp`
- `CeilDivide`
- `NarrowCastChecked`

overflow時は `std::overflow_error`。

#### Error

- `VulkanException`
- `VkResultToString`
- `CheckVkResult`
- operation/detail message

#### PixelFormat

- plane count
- plane format
- plane extent
- bytes per texel
- packed/planar判定
- format name
- validation

### 4.4 Tests

- all VkResult name cases used by project
- checked math boundary/max tests
- NV12/P010 plane geometry
- odd extent rejection
- ArrayView constructors/lifetime-neutral behavior
- all headers standalone compile

### 4.5 完了条件

FoundationがVMA/volkの実装detailをincludeしない。

## 5. Phase 2 — Loader、Dispatch、Context、Capability

### 5.1 対象要求

VGS-CORE-001〜008。

### 5.2 作成

- `VulkanLoader.hpp/.cpp`
- internal `VulkanDispatch.hpp/.cpp`
- `VulkanCapabilities.hpp/.cpp`
- `VulkanExecutionContext.hpp/.cpp`
- `VmaImplementation.cpp`

### 5.3 手順

1. volk process initialization wrapper。
2. custom proc addressのfirst-call contract。
3. `ContextState` PImpl作成。external VMA allocatorは内部同期有効をdescriptor契約として要求し、false宣言をreject。
4. instance table load。
5. device table load。
6. physical device properties/features/queue/format query。
7. desc validation。
8. external allocator path。
9. internal VMA allocator path。
10. explicit `VmaVulkanFunctions` population。
11. context destructor order。
12. context fault stateの枠だけ用意。

### 5.4 Capability validation

最低限check:

- non-null handles
- apiVersion >=1.3
- physical API support
- queue family index範囲
- queue flags compute+transfer
- enabledFeatures宣言true
- support features true
- pushConstant size >= project maximum
- descriptor counts >= minimum
- required plane/output formatsのfeature
- allocation callback/queue host-sync userData/external allocator/root handlesのborrowed lifetime契約をAPI docsとdebug testで確認

Format validationはProcessingをdisableしたbuildではCore最低formatだけにします。Processing Context作成時に追加formatを検証します。

### 5.5 Multiple Device test

実Deviceが2つ存在する場合:

- 2 context作成。
- 各device tableでDevice固有objectを作成。
- handle/dispatch交差なし。

1 Device環境:

- dispatch table loaderをinject可能なinternal test seamで2 fake tableを検証。
- production public APIへmock型を露出しない。

### 5.6 完了条件

- context destructorがexternal instance/device/queue/allocatorを破棄しない。
- internal allocatorのみdestroy。
- global `volkLoadDevice`をgrepで禁止。
- `vulkan-1` / `libvulkan`への直接linkをcoreで行わない。

## 6. Phase 3 — Queue、Timeline、SyncPoint

### 6.1 対象要求

VGS-SYNC-001〜007。

### 6.2 実装

1. `TimelineState`作成。
2. timeline semaphore create。
3. atomic/locked counter設計。
4. `VulkanSyncPoint` copyable immutable実装。library-ownedとborrowed external timelineの両方を扱う。
5. `IsComplete` / timeout wait。
6. submit descriptor validation。
7. internal mutexまたは注入host lock callbackで `vkQueueSubmit2` を直列化。
8. own timeline signal append。
9. external binary/timeline wait/signal。
10. device lost fault propagation。
11. overflow test seam。

### 6.3 重要なcommit順

`nextValue`はsubmit成功前に外部公開しません。

推奨:

- mutex内でcandidate算出
- submit情報構築
- Vulkan call
- successならcounter更新
- SyncPoint構築

失敗時にvalue gapがあっても機能上は問題ありませんが、単調性とtestの単純化のためsuccess commitを採用します。ただし同じvalueを別submitで再利用してはならないため、Vulkan callがdevice側で部分受理され得るエラーの扱いを仕様確認し、device lost時はQueueをfaultedにして再submit禁止とします。

### 6.4 Tests

- empty submitでtimeline signal
- 1,000 sequential monotonic values
- timeout and completion
- 8 threads x 1,000 submits
- external timeline wait
- binary wait sample
- faulted queue mock
- Validation error 0

### 6.5 完了条件

Thread sanitizer対象のhost stateにdata raceなし。

## 7. Phase 4 — Command ContextとBarrier

### 7.1 対象要求

VGS-CMD-001〜006、VGS-GPU-005〜008。

### 7.2 実装

- Command Pool create/destroy
- primary Command Buffer allocate
- state machine
- owner thread debug validation
- Begin/End/Abort
- Submit integration
- last point保存
- TryRecycle / WaitAndRecycle
- `VulkanImageState` / `VulkanBufferState`
- state helper
- Barrier Batch
- `vkCmdPipelineBarrier2`

### 7.3 Abort semantics

Recording中の例外時:

- `vkEndCommandBuffer`を無理に呼ばない。
- command buffer reset可能なpool flagを使用。
- AbortはstateをInitialへ戻すresetを試みる。
- reset失敗はdiagnosticし、contextをfaultedにする。

### 7.4 Tests

- all legal transitions
- all illegal transitions
- owner thread violation debug test
- Pending before completion reset rejection
- completion after recycle
- image barrier structure unit test via builder
- actual transfer barrier validation test

## 8. Phase 5 — VMA Buffer/Image/View/Bundle

### 8.1 対象要求

VGS-GPU-001〜004、009〜010。

### 8.2 実装順

1. Buffer descriptor validation。
2. VMA Buffer creation。
3. map/flush/invalidate。
4. Image descriptor validation。
5. VMA Image creation。
6. Image View creation。
7. non-owning refs。
8. PixelFormat logical Bundle creation。
9. debug names。
10. move/destruction order。

### 8.3 VMA lifetime test

- custom allocator callback counters。
- move construction/assignment。
- exception during view creation cleanup。
- external allocator not destroyed。
- internal allocator destruction after resources zero。

### 8.4 Format feature test

- unsupported usageは作成前にreject。
- BGRA storage supportがないDeviceではProcessing capabilityから除外またはtest skip。
- mandatory pathの不足はProcessing Context作成失敗として明確message。

### 8.5 Bundle tests

- packed 1 plane。
- NV12/P010 2 plane extent/format。
- odd dimensions rejected。
- required usageが全planeへ適用。
- view ref lifetime comment/static tests。

## 9. Phase 6 — Descriptor、Shader、Pipeline

### 9.1 対象要求

VGS-DESC-001〜003、VGS-SHADER-001〜004。

### 9.2 Shader toolchain

1. HLSL include/source skeleton。
2. CMake DXC command。
3. depfileまたはinclude dependency追跡。
4. spirv-val command。
5. generated SPIR-V registry。
6. hash/command metadata。
7. precompiled path。
8. reproducibility test。

### 9.3 Descriptor Arena

- Pool sizesはProcessing Contextから推奨値を生成可能。
- allocate failure時、capacity不足message。
- retire point必須。
- completion前reset拒否。
- thread-confined検査。

### 9.4 Compute pipeline wrapper

- Shader Module RAII。
- Descriptor Set Layout RAII。
- Pipeline Layout RAII。
- Compute Pipeline RAII。
- external Pipeline Cache borrowed。
- internal cache owned。
- pipeline key hash。

### 9.5 Tests

- SPIR-V validation。
- invalid magic/size rejection。
- descriptor allocate/reset lifecycle。
- pipeline creation。
- concurrent lazy pipeline lookup。
- binding ABI constants。

## 10. Phase 7 — Upload Ring

### 10.1 対象要求

VGS-XFER-001〜006、010の一部。

### 10.2 実装順

1. `CpuImageView` validation。
2. Upload Buffer create/mapped pointer。
3. virtual head/tail ring。
4. reservation state。
5. alignment/padding/wrap。
6. TryAcquire。
7. timeout Acquire。
8. Cancel path。
9. Retire SyncPoint。
10. CollectCompleted。
11. flush range。
12. stats/high-water mark。

### 10.3 CPU-only test seam

Ring core algorithmをVulkan Bufferから分離したinternal `RingAllocator` としてtest可能にします。完成APIへその型を露出しません。

### 10.4 Stress cases

- exact capacity
- capacity-1 + alignment
- wrap padding
- cancelled reservationが先頭
- out-of-order GPU completion point
- multiple producers
- timeout
- size > capacity
- integer overflow
- noncoherent range expansion

### 10.5 完了条件

10 million程度のrandomized CPU allocation/retire simulationでinvariant違反なし。実行時間がCI過大ならseed固定の縮小版を通常CI、長時間版をnightlyに分離します。

## 11. Phase 8 — Upload Batch / Executor

### 11.1 対象要求

VGS-XFER-007〜012。

### 11.2 実装

1. CPU byte range→GPU Buffer copy (`vkCmdCopyBuffer2`)。
2. packed row repack。
3. NV12 row repack。
4. P010 row repack。
5. plane offset alignment。
6. before→TransferDst barrier。
7. `VkCopyBufferToImageInfo2`構築。
8. after barrier。
9. Batch slices ownership。
10. submit成功commit。
11. cancel/exception rollback。
12. thread-safe ExecutorのCommandSlot pool、completion collect、slot上限/block policy。

### 11.3 Test readback utility

Public v1 APIにReadbackを入れなくても、GPU golden testのため `test/Gpu/TestReadback` を実装します。これはtest targetだけに置き、製品APIと混同しません。将来public化は別仕様変更です。

### 11.4 Tests

- Buffer upload offset/size/state/readback exact。
- R8/RG8/RGBA8/BGRA8/R16/RG16/RGBA16F。
- padded source stride。
- minimum/large extent。
- NV12/P010 plane content exact readback。
- source pointerをUpload call直後に破棄しても成功。
- Ring slotがcompletion前に再利用されない。
- concurrent Executor calls。各in-flight callが異なるCommand Poolを使用し、slot再利用はcompletion後だけ。
- validation errors 0。

## 12. Phase 9 — Image Pool / Lease / Deferred Release

### 12.1 対象要求

VGS-POOL-001〜011。

### 12.2 実装順

1. Pool shared state/slot/generation。
2. eager initial allocation。初回stateは必ずUNDEFINEDとして検証。
3. TryAcquire free path。
4. capacity reservation + lazy growth。
5. Lease move semantics。
6. ReleaseNow。
7. ReleaseAfter。
8. Pending completion collection。
9. Blocking/timeout Acquire。
10. Quarantine。
11. Stats。
12. Deferred Release Queue。
13. Pool public object破棄とoutstanding Lease。

### 12.3 Invariants

常時成立:

```text
Free + Leased + Pending + Quarantined + Creating == totalSlots
Free slot has no retire point
Pending slot has valid retire point
Leased slot generation matches lease token
A generation can be released at most once
Quarantined slot is never acquired
```

assertはdebug、releaseでは安全なerror処理。

### 12.4 Tests

- initial/max capacity。
- TryAcquire exhaustion。
- blocking wake。
- timeout。
- GPU completion gating。
- stale/double release拒否。
- Lease move chain。
- destructor quarantine。
- Pool destruction with outstanding Lease。
- 16 threads randomized acquire/release。
- final state handoff。
- deferred destruction before/after point。

### 12.5 完了条件

PoolがGPU完了前に同一VkImage handleを再Acquireしないことをtestでassert。

## 13. Phase 10 — Processing基盤

### 13.1 対象要求

VGS-PROC-001〜004、009〜010、VGS-DIAG-001〜002。

### 13.2 実装

1. Processing value types。
2. Context/descriptor layouts/samplers。
3. fixed binding ABI。
4. pipeline registry。
5. packed copy shader。
6. Processor immutable design。
7. input validation。
8. before/working/after barrier。
9. dispatch calculation。
10. object names/diagnostic callback。

### 13.3 First GPU milestone

RGBA8 input→RGBA8 output identity copyを実装し、Upload→Processing→Readbackを1 testで通します。

### 13.4 Tests

- identity exact。
- crop exact。
- invalid rect。
- unsupported usage。
- source/destination alias rejection。
- correct final state used by subsequent transfer readback。
- two threads/independent command+descriptor arena, shared Processor。

## 14. Phase 11 — Resize、NV12/P010、Fused

### 14.1 対象要求

VGS-PROC-005〜008、011〜012。

### 14.2 Shader順

1. packed point resize。
2. packed linear resize。
3. YUV primitive CPU reference。
4. NV12 point conversion。
5. NV12 linear conversion。
6. P010 conversion。
7. BT.601/709/2020。
8. Full/Limited。
9. fused convert+resize。
10. RGBA16F output。

### 14.3 Golden生成

GPU出力の正解をGPU実装から生成しません。独立CPU referenceをtest内に実装します。

- double precision matrix calculation
- nearest/linear sampling reference
- P010 code extraction
- clamp/rounding規約
- edge pixel/chroma position

### 14.4 Tolerance

初期基準:

- RGBA8 point/identity: exact
- RGBA8 linear: channel absolute error <=1、edge条件は<=2を許容する場合理由を文書化
- YUV→RGBA8: channel absolute error <=2
- RGBA16F: absolute/relative epsilonをformatごとに規定

driver差を理由に広いtoleranceへ逃げません。

### 14.5 Test vectors

- black/white/primary colors
- limited range endpoints
- chroma extremes
- color bars
- odd crop origin（image extent自体はeven）
- 2x2、4x4 minimum
- 1920x1080 padded stride
- P010 values 0/64/512/940/960/1023等
- each matrix/range combination

## 15. Phase 12 — Integration、Samples、Packaging、Release hardening

### 15.1 Samples

#### 01_BorrowedContext

- vk-bootstrapはsample内だけ。
- external handlesをContextへ渡す。
- capability表示。

#### 02_UploadRgba

- synthetic CPU image。
- Pool Acquire。
- Upload。
- test/sample readback hash。

#### 03_UploadNv12

- two-plane source。
- GPU bundleへupload。

#### 04_ConvertResize

- NV12→RGBA fused。
- output hash/optional raw dump。

#### 05_ImagePool

- multiple in-flight images。
- timeline completionとreuse。
- Pool stats。

sampleはWindow/SwapChainを使いません。

### 15.2 Documentation

- Root README quick start。
- module index。
- ownership/threading rules。
- external Device creation requirements。
- shader rebuild guide。
- package integration。
- troubleshooting Validation errors。
- Camera adapter exampleは非規範別節。

### 15.3 Package

- headers
- static library
- CMake config/targets/version
- shader source
- precompiled SPIR-Vまたはembedded registry
- licenses
- docs optional

### 15.4 Release matrix

| Platform | Compiler | Config | Validation | Sanitizer |
|---|---|---|---|---|
| Windows x64 | MSVC | Debug/Release | yes | optional ASan |
| Linux x64 | GCC | Debug/Release | yes | ASan/UBSan |
| Linux x64 | Clang | Debug/Release | yes | ASan/UBSan/CPU TSan |

### 15.5 1.0 release checklist

- 全要求IDにtestまたは根拠。
- Validation Error 0。
- package smoke成功。
- ABI/public header review。
- third-party license確認。
- generated shader hash一致。
- C++17のみを使用し、designated initializer等のC++20構文なし。
- no forbidden dependency。
- no global Device dispatch。
- no implicit current layout。
- no ordinary-path device idle。
- docs/code一致。
- tag前にversion更新。

## 16. CI設計

### 16.1 常時CPU job

- configure/build
- header compile
- CPU unit
- package smoke
- formatting
- forbidden dependency grep
- shader source/generated hash check

### 16.2 GPU job

self-hostedまたはVulkan software driver環境:

- validation layer
- upload/golden
- processing
- queue stress
- pool completion

GPU unavailable時に全GPU testをsilent passしません。CI job自体を明示optionalにし、Release gateでは最低1 Windows GPU + 1 Linux Vulkan implementationを要求します。

### 16.3 Static checks

禁止pattern例:

```text
volkLoadDevice(
vkDeviceWaitIdle(  # allowlistされたshutdown/test以外
#include <boost/
#include <nlohmann/
ThreadKit
ThreadToolkit
#include <vulkan/vulkan.hpp>
```

Vulkan global function callの機械検査は完全ではないためreviewも必要です。

## 17. Test suite構成

```text
Foundation
Loader
DispatchIsolation
Capabilities
Queue
SyncPoint
CommandContext
Barrier
Buffer
Image
ImageBundle
DescriptorArena
ShaderRegistry
ComputePipeline
CpuImageView
UploadRing
UploadBatch
UploadExecutor
ImagePool
DeferredRelease
ProcessingPacked
ProcessingResize
ProcessingNv12
ProcessingP010
ProcessingFused
ThreadingStress
Validation
PackageSmoke
```

## 18. Code review checklist

### Ownership

- 何を所有し、何をborrowしているか。
- destroy順序。
- move後状態。
- pending GPU lifetime。

### Threading

- class classification。
- Vulkan externally synchronized object。
- lock order。
- callback under lockの有無。

### Synchronization

- source/destination stage/access。
- layout transition。
- queue family。
- semaphore wait stage。
- host flush/invalidate。

### Resource

- format feature。
- usage flag。
- subresource range。
- overflow/alignment。

### Error

- partial construction cleanup。
- Device lost。
- timeout。
- destructor noexcept。

### Test

- negative case。
- Validation。
- GPU completion before reuse。
- thread stress。

## 19. Codex作業時の必須報告

各Phase終了時、Codexは次を回答に含めます。

1. 変更ファイル一覧。
2. 実装した要求ID。
3. 所有権・同期上の重要判断。
4. 実行したbuild/test command。
5. test結果。
6. Validation結果。
7. 未実施項目と理由。
8. 次Phaseへ進めるか。

「buildしていない」「GPU test環境がない」場合は明記し、成功したように書きません。

## 20. 実装中に判断が必要な場合

優先順位:

1. Vulkan specificationとValid Usage
2. `01_Specification.md`
3. `02_Architecture_Design.md`
4. 正しさ・安全なlifetime
5. APIの明示性
6. 性能
7. 短いコード

不明点が、公開API・ownership・sync・dependencyを変えない局所実装なら合理的な選択を行い、コメント/testへ残します。公開契約を変える場合は文書を先に更新します。

---

このPhase順を初期実装の標準工程とします。
