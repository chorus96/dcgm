# DCGM 코드 조사

이 저장소의 코드를 읽고 조사한 내용을 주제별로 정리한 문서입니다. GPU가 있는 환경에서 실제로 실행해 확인하지는 않았습니다.

- [1. sm_61(Pascal) GPU 지원](#1-sm_61pascal-gpu-지원)
- [2. memtest 플러그인의 빌드, 로드, 실행 과정](#2-memtest-플러그인의-빌드-로드-실행-과정)

## 1. sm_61(Pascal) GPU 지원

이 저장소의 DCGM을 sm_61(Pascal 세대, 예: GTX 1080, Tesla P40·P4) GPU에서 빌드하고 실행할 수 있는지 코드를 기준으로 조사한 결과입니다.

### 요약

- **빌드:** GPU 종류와 상관없이 됩니다.
- **실행:** 모니터링, 헬스, 정책, 설정 기능은 됩니다.
- **진단(NVVS):** 드라이버가 CUDA 13.0 이하를 보고하면 됩니다. L1 캐시 태그 테스트만 건너뜁니다. CUDA 13.1 이상이면 실패할 가능성이 큽니다.
- **프로파일링 지표:** Pascal에서는 쓸 수 없을 가능성이 높습니다(미확인).

### 빌드

빌드는 GPU 종류와 관계없습니다. 빌드는 Docker 이미지 안에서 호스트 코드로만 이루어지고, CMake에 특정 GPU 세대를 지정하는 설정(`CMAKE_CUDA_ARCHITECTURES`, `-gencode` 등)이 없습니다. 진단용 CUDA 커널은 `sm_30` 기준 PTX로 들어가서 실행할 때 드라이버가 컴파일합니다(`nvvs/plugin_src/diagnostic/build_ptx_string.sh`, `nvvs/plugin_src/memory/build_ptx_string.sh`). 빌드 명령은 다른 환경과 같습니다(`./build.sh -r`).

### 실행: 기능별 상황

| 구성 요소 | sm_61 지원 여부 | 근거 |
|---|---|---|
| `nv-hostengine`, `dcgmi`, 필드 모니터링, 헬스, 정책 | 지원. NVML로 동작하며 Pascal을 따로 처리하는 코드가 있음 | `dcgmlib/src/DcgmCacheManager.cpp:1716`, `modules/config/DcgmConfigManager.cpp:251` |
| 진단(NVVS) 대상 GPU 판정 | 지원. Maxwell(5.x) 이상이면 모든 브랜드를 허용 | `nvvs/src/NvidiaValidationSuite.cpp:696-706` |
| 진단의 L1 캐시 태그 테스트 | 미지원. Volta 전용이라 건너뜀 | `nvvs/plugin_src/memory/L1TagCuda.cpp:93-122` |
| `dcgmproftester` | `dcgmproftester12` 사용 필요 | `dcgmproftester/DcgmProfTester.cpp:1213-1225` |
| 프로파일링 지표(`DCGM_FI_PROF_*`) | 코드로 확인할 수 없음. 해당 모듈은 비공개 소스임 | NVIDIA 문서상 Volta 이상만 지원하는 것으로 알고 있음(미확인) |

### 주의: CUDA 13과 Pascal

CUDA 13.0부터 compute capability 7.5 미만 GPU(Maxwell, Pascal, Volta) 지원이 빠졌습니다. 그래서 진단 플러그인은 CUDA 11, 12, 13 버전별로 따로 빌드됩니다(`nvvs/plugin_src/CMakeLists.txt:77`, `SUPPORTED_CUDA_VERSIONS 11 12 13`). 실행할 때 사용할 플러그인 버전은 `nvvs/include/TestFramework.inl:376-407`의 `GetCompatibleCudaMajorVersion()`에서 고릅니다.

```cpp
if ((arch == MAXWELL || arch == PASCAL || arch == VOLTA)
    && cudaDriverMajorVersion == 13 && cudaDriverMinorVersion == 0)
    return 12;   // cuda12 플러그인으로 대체
```

| 드라이버가 보고하는 CUDA 버전 | 선택되는 플러그인 | sm_61에서 결과 |
|---|---|---|
| 11.x | `cuda11` | 정상 |
| 12.x | `cuda12` | 정상 |
| 13.0 | `cuda12`(대체) | 정상 |
| 13.1 이상 | `cuda13` | 진단의 CUDA 테스트가 실패할 가능성이 큼 |

- 대체 조건은 CUDA 버전이 정확히 **13.0**일 때만 적용됩니다.
- `dcgmproftester`도 같은 방식입니다. CUDA 13.0에서는 `dcgmproftester12`를 쓰라는 안내가 나옵니다. 13.1 이상에서는 이런 안내가 없고, 버전이 맞지 않는다는 오류가 납니다.
- Pascal을 지원하는 마지막 드라이버 계열이 R580(CUDA 13.0)인 것으로 알고 있습니다. 그렇다면 Pascal 환경에서 CUDA 13.1 이상이 보고되는 경우는 실제로 없을 수 있습니다. 이 부분은 저장소에서 확인한 내용이 아니므로 NVIDIA 드라이버 지원 정책을 확인해야 합니다.

### 확인되지 않은 사항

- 프로파일링 모듈(`libdcgmmoduleprofiling`)의 Pascal 지원 여부: 비공개 소스라 코드로 확인할 수 없습니다.
- Pascal을 지원하는 마지막 드라이버 계열과 그 드라이버가 보고하는 CUDA 버전: NVIDIA 공식 자료로 확인해야 합니다.
- 실제 sm_61 GPU에서의 동작: 위 내용은 모두 코드를 읽고 판단한 것입니다.

## 2. memtest 플러그인의 빌드, 로드, 실행 과정

memtest는 CUDA 커널을 미리 PTX 텍스트로 만들어 플러그인 `.so` 안에 바이트 배열로 넣어 둡니다. 진단을 실행하면 `nvvs`가 이 `.so`를 `dlopen`으로 열고, 플러그인이 그 PTX를 CUDA Driver API로 GPU에 올려 커널을 실행합니다.

```
[미리 해 둔 작업] tests.cu ──nvcc -ptx──▶ tests.ptx ──bin2c──▶ inc/tests.h (memtest_ptx_string[])
[빌드]            Memtest.cpp + inc/tests.h ──CUDA 11/12/13별로──▶ plugins/cudaXX/libMemtest.so.*
[실행]            dcgmi diag ─▶ nv-hostengine(diag 모듈) ─▶ nvvs 자식 프로세스 ─dlopen─▶ libMemtest
                  ─▶ cuModuleLoadData(PTX) ─(드라이버가 JIT 컴파일)─▶ cuLaunchKernel
```

### 빌드 단계

**① 커널 소스 → PTX → C 헤더 (저장소에 이미 들어 있음)**
- GPU 커널은 `nvvs/plugin_src/memtest/tests.cu`에 있습니다. `move_inv_write`, `test0_*`부터 `test10_*`까지 모두 `extern "C" __global__`로 선언되어 있습니다.
- 이 파일을 컴파일한 결과인 `tests.ptx`가 저장소에 들어 있습니다. PTX 헤더를 보면 CUDA 10.2 컴파일러로 `.target sm_30`, `.version 6.5`로 생성되었습니다.
- PTX는 `bin2c`로 변환되어 `inc/tests.h`의 `unsigned char memtest_ptx_string[]` 배열이 됩니다.
- 이 과정은 CMake 빌드에 포함되어 있지 않습니다. 커널을 바꾸면 PTX와 헤더를 직접 다시 만들어야 합니다. 옆에 있는 memory 플러그인은 이 작업용 스크립트(`nvvs/plugin_src/memory/build_ptx_string.sh`: nvcc `-ptx -arch=sm_30` 후 `bin2c`)가 있지만, memtest 폴더에는 이런 스크립트가 없습니다.

**② 플러그인 공유 라이브러리 빌드**
- `memtest/CMakeLists.txt`는 `declare_nvvs_plugin(memtest .)`으로 소스를 등록합니다. 그다음 `Cuda11/`, `Cuda12/`, `Cuda13/` 하위 디렉터리마다 `define_plugin(Memtest <ver>)`를 호출합니다.
- `define_plugin` 매크로(`nvvs/plugin_src/CMakeLists.txt:44-75`)가 같은 소스를 CUDA 버전마다 한 번씩 빌드해 `Memtest_11`, `Memtest_12`, `Memtest_13` 타깃을 만듭니다.
  - 해당 버전의 CUDA 라이브러리와 `pluginCudaCommon_<ver>`, `pluginCommon`을 링크합니다.
  - 출력 이름은 `Memtest`입니다.
  - 외부에 공개할 심볼은 `nvvs_plugin.linux_def` 버전 스크립트로 제한합니다.
- `Memtest.cpp`는 `#include <inc/tests.h>`로 PTX 배열을 가져오므로, PTX가 `.so` 안에 데이터로 들어갑니다.
- 설치 경로는 `libexec/datacenter-gpu-manager-4/plugins/cuda{11,12,13}/`입니다(`CMakeLists.txt:703-720`).

### 실행 경로: 진단 요청부터 플러그인 로드까지

1. **요청:** memtest는 `dcgmi diag -r memtest`처럼 이름으로 지정하거나, 가장 긴 진단 단계(`NVVS_SUITE_XLONG`, 레벨 4)에 포함되어 실행됩니다(`nvvs/src/NvidiaValidationSuite.cpp:1170-1174`).
2. **nvvs 실행:** `nv-hostengine`의 diag 모듈(`modules/diag/DcgmDiagManager.cpp`)이 `nvvs` 바이너리를 자식 프로세스로 실행합니다.
3. **플러그인 디렉터리 선택:** nvvs는 드라이버가 보고하는 CUDA 버전을 보고 `/cuda11/`, `/cuda12/`, `/cuda13/` 중 하나를 고릅니다(`GetPluginCudaDirExtension`, `nvvs/include/TestFramework.inl:325`). [1장](#주의-cuda-13과-pascal)의 Pascal/CUDA 13.0 대체 규칙이 여기서 적용됩니다.
4. **dlopen:** nvvs는 선택한 디렉터리에서 `*.so.<숫자>` 파일을 모두 찾아 `dlopen`합니다(`LoadPluginWithDir`, `nvvs/src/PluginLib.cpp:180`). 그다음 `dlsym`으로 필수 진입점을 찾습니다.

   | 진입점(`MemtestWrapper.cpp`) | 역할 |
   |---|---|
   | `GetPluginInterfaceVersion` | 플러그인 인터페이스 버전 확인 |
   | `GetPluginInfo` | 테스트 이름 `memtest`와 파라미터 목록(`test_duration`, `test0`~`test10`, `num_chunks` 등) 등록 |
   | `InitializePlugin` | `MemtestPlugin` 객체 생성, 로깅과 멈춤 감지(hang detection) 연결 |
   | `RunTest` | `MemtestPlugin::Go()` 호출 |
   | `RetrieveResults` / `RetrieveCustomStats` | 결과와 통계를 nvvs에 반환 |

5. **Go():** `memtest_wrapper.cpp:55`에서 파라미터를 적용합니다(기본 `test_duration` 600초). `is_allowed`가 false이면 테스트를 건너뜁니다. 그 외에는 `Memtest` 객체를 만들어 `Run()`을 호출합니다.

### GPU에 로드하기 (`Memtest.cpp`)

`Memtest::Run()`(`Memtest.cpp:535`)은 다음 순서로 진행합니다.

1. **Init():** `cuInit(0)`을 호출합니다. 그다음 DCGM이 넘겨준 GPU마다 PCI 버스 ID로 `cuDeviceGetByPCIBusId`를 호출해 CUDA 장치를 찾습니다. PCI ID로 찾기 때문에 `CUDA_VISIBLE_DEVICES`로 순서가 바뀌어도 맞는 GPU를 찾습니다.
2. **CudaInit():** GPU마다 `cudaSetDevice`, `cudaDeviceReset`으로 상태를 정리한 뒤 `cuCtxCreate_v2`로 전용 컨텍스트를 만듭니다.
3. **LoadCudaModule():** 핵심 단계입니다(`Memtest.cpp:260`).
   - `cuModuleLoadData(&gpu->cuModule, memtest_ptx_string)`로 `.so`에 들어 있는 PTX 텍스트를 드라이버에 넘깁니다.
   - 이때 **드라이버가 PTX를 해당 GPU의 기계어(SASS)로 JIT 컴파일**합니다. PTX가 `sm_30` 기준이라 sm_61을 포함한 이후 세대 GPU에서도 드라이버가 컴파일할 수 있습니다.
   - 그다음 `cuModuleGetFunction`으로 커널 함수 핸들(`cuFuncTest0Write`, `cuFuncMoveInvRead` 등 약 20개)을 하나씩 가져옵니다.

### 커널 실행

1. **워커 스레드:** GPU마다 `MemtestWorker` 스레드를 하나씩 띄우고 멈춤 감지에 등록합니다(`Memtest.cpp:560-575`). 모든 워커가 끝날 때까지 기다립니다.
2. **메모리 확보:** `MemtestWorker::run()`(`Memtest.cpp:802`)이 다음을 수행합니다.
   - `cuCtxSetCurrent`로 자기 GPU의 컨텍스트를 현재 스레드에 연결합니다.
   - `cudaMemGetInfo`로 빈 메모리를 확인합니다. 빈 메모리가 `minimum_allocation_percentage`보다 적으면 테스트를 건너뜁니다.
   - `MemoryChunkManager`가 1MB(`BLOCKSIZE`) 단위로 계산한 크기를 `num_chunks`개 청크로 나누어 할당합니다. 기본은 `cudaMalloc`이고, `use_mapped_mem`이 켜져 있으면 `cudaHostAlloc(Mapped)`를 씁니다.
   - 할당에 실패하면 크기를 조금씩 줄이며 다시 시도합니다.
3. **테스트 반복:** `RunTests()`(`Memtest.cpp:727`)는 `cuda_memtests[]` 표에서 켜진 테스트만 차례로 실행합니다. 기본으로 켜진 것은 Test7(랜덤 숫자 시퀀스)과 Test10(메모리 스트레스)입니다. 이 순서를 `test_duration`이 지날 때까지 반복합니다.
4. **커널 호출:** 각 `testN()` 함수는 청크별로 `cuLaunchKernel(gpu->cuFuncTestNWrite, grid…)`를 실행해 패턴을 씁니다. 그다음 Read 커널로 다시 읽어 비교합니다. 오류 개수는 `error_checking()`으로 모읍니다.
5. **판정과 정리:** `CheckPassFail()`이 GPU별 오류 개수(`gpu_errors[]`)로 통과/실패를 정합니다. `Cleanup()`은 `cuModuleUnload`, `cuCtxDestroy`, `cuDevicePrimaryCtxReset`으로 자원을 해제합니다. 결과는 `RetrieveResults`를 통해 nvvs로, 다시 diag 모듈과 `dcgmi`로 전달됩니다.

### 참고

- 커널은 CUDA Runtime의 `<<<>>>` 문법이 아니라 **Driver API(`cuModuleLoadData` + `cuLaunchKernel`)** 로 실행됩니다. 그래서 `.so`에는 fatbin이 없고 PTX 텍스트만 들어 있습니다. 커널 기계어는 실행할 때마다 설치된 드라이버가 만듭니다.
- 테스트할 때 `__DCGM_DIAG_MEMTEST_FAIL_GPU` 환경 변수를 설정하면 가짜 실패를 만들 수 있습니다(`Memtest.cpp:719`). 가짜 GPU(NVML 인젝션) 환경에서는 커널을 실행하지 않고 통과로 처리합니다(`memtest_wrapper.cpp`의 `UsingFakeGpus`).
