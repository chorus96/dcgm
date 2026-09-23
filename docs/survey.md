# sm_61(Pascal) GPU 지원 조사

이 저장소의 DCGM을 sm_61(Pascal 세대, 예: GTX 1080, Tesla P40·P4) GPU에서 빌드하고 실행할 수 있는지 코드를 기준으로 조사한 결과입니다. GPU가 있는 환경에서 실제로 실행해 보지는 않았습니다.

## 요약

- **빌드:** GPU 종류와 상관없이 됩니다.
- **실행:** 모니터링, 헬스, 정책, 설정 기능은 됩니다.
- **진단(NVVS):** 드라이버가 CUDA 13.0 이하를 보고하면 됩니다. L1 캐시 태그 테스트만 건너뜁니다. CUDA 13.1 이상이면 실패할 가능성이 큽니다.
- **프로파일링 지표:** Pascal에서는 쓸 수 없을 가능성이 높습니다(미확인).

## 빌드

빌드는 GPU 종류와 관계없습니다. 빌드는 Docker 이미지 안에서 호스트 코드로만 이루어지고, CMake에 특정 GPU 세대를 지정하는 설정(`CMAKE_CUDA_ARCHITECTURES`, `-gencode` 등)이 없습니다. 진단용 CUDA 커널은 `sm_30` 기준 PTX로 들어가서 실행할 때 드라이버가 컴파일합니다(`nvvs/plugin_src/diagnostic/build_ptx_string.sh`, `nvvs/plugin_src/memory/build_ptx_string.sh`). 빌드 명령은 다른 환경과 같습니다(`./build.sh -r`).

## 실행: 기능별 상황

| 구성 요소 | sm_61 지원 여부 | 근거 |
|---|---|---|
| `nv-hostengine`, `dcgmi`, 필드 모니터링, 헬스, 정책 | 지원. NVML로 동작하며 Pascal을 따로 처리하는 코드가 있음 | `dcgmlib/src/DcgmCacheManager.cpp:1716`, `modules/config/DcgmConfigManager.cpp:251` |
| 진단(NVVS) 대상 GPU 판정 | 지원. Maxwell(5.x) 이상이면 모든 브랜드를 허용 | `nvvs/src/NvidiaValidationSuite.cpp:696-706` |
| 진단의 L1 캐시 태그 테스트 | 미지원. Volta 전용이라 건너뜀 | `nvvs/plugin_src/memory/L1TagCuda.cpp:93-122` |
| `dcgmproftester` | `dcgmproftester12` 사용 필요 | `dcgmproftester/DcgmProfTester.cpp:1213-1225` |
| 프로파일링 지표(`DCGM_FI_PROF_*`) | 코드로 확인할 수 없음. 해당 모듈은 비공개 소스임 | NVIDIA 문서상 Volta 이상만 지원하는 것으로 알고 있음(미확인) |

## 주의: CUDA 13과 Pascal

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

## 확인되지 않은 사항

- 프로파일링 모듈(`libdcgmmoduleprofiling`)의 Pascal 지원 여부: 비공개 소스라 코드로 확인할 수 없습니다.
- Pascal을 지원하는 마지막 드라이버 계열과 그 드라이버가 보고하는 CUDA 버전: NVIDIA 공식 자료로 확인해야 합니다.
- 실제 sm_61 GPU에서의 동작: 위 내용은 모두 코드를 읽고 판단한 것입니다.
