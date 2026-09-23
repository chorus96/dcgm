# CLAUDE.md

이 파일은 Claude Code(claude.ai/code)가 이 저장소에서 작업할 때 참고할 가이드를 제공합니다.

## 프로젝트 개요

NVIDIA Data Center GPU Manager(DCGM): 데이터센터 GPU를 모니터링하고 관리하기 위한 C++ 데몬(`nv-hostengine`), 클라이언트 라이브러리(`libdcgm.so`), CLI(`dcgmi`), 진단 도구 모음(NVVS), Python/C SDK로 구성됩니다. 이 GitHub 저장소는 NVIDIA 내부 코드베이스를 릴리스 단위로 공개한 스냅샷입니다(대략 릴리스당 커밋 하나, 예: "DCGM 4.6.1"). 따라서 대규모 리팩터링은 업스트림에 병합하기 어려우므로 변경은 범위를 좁게 유지하세요.

## 빌드

모든 빌드는 Docker 빌드 이미지 안에서 실행됩니다. `build.sh`, `format-dcgm`, `validate_format.sh`는 `DCGM_BUILD_INSIDE_DOCKER=1`이 아니면 `intodocker.sh`를 통해 컨테이너 안에서 자기 자신을 다시 실행합니다. 빌드 이미지 자체를 만드는 작업(`cd dcgmbuild && ./build.sh`)은 몇 시간이 걸리며, 사용할 이미지는 `DCGM_DOCKER_IMAGE`로 바꿀 수 있습니다.

```bash
./build.sh -r                 # amd64용 릴리스(RelWithDebInfo) 빌드, 단위 테스트 실행
./build.sh -d                 # 디버그 빌드
./build.sh -r --deb           # .deb도 생성 (--rpm, tar.gz는 -p/--packages)
./build.sh -d -n              # 빌드 시 테스트 생략
./build.sh -c ...             # 클린 리빌드
./build.sh -a aarch64 ...     # 다른 아키텍처용 빌드
./build.sh --address-san      # --thread-san, --ub-san, --leak-san, --coverage도 가능
./build.sh -d -- -DCMAKE_VERBOSE_MAKEFILE=ON   # -- 뒤에 추가 cmake 인자
```

- 빌드 트리: `_out/build/<Linux-arch-buildtype>/`, 설치 트리: `_out/<Linux-arch-buildtype>/`.
- 병렬 빌드 수는 `NPROC`로 조절합니다. `DCGM_SKIP_PYTHON_LINTING=1`을 설정하면 빌드 후마다 실행되는 pylint 검사를 건너뜁니다.
- `CMakePresets.json`(Ninja, CMake 4.0 이상)에는 컨테이너 안에서 쓸 수 있는 `Debug`/`RelWithDebInfo`/`Release`/`DEB`/`RPM`/`TGZ` 프리셋이 있습니다(예: `./intodocker.sh -- bash`로 진입).

## 테스트

**단위 테스트(C++, Catch2)**는 각 `*/tests/CMakeLists.txt`에서 `catch_discover_tests`로 CTest에 등록되며, `build.sh`가 자동으로 실행합니다(`ctest --output-on-failure --test-dir <빌드 디렉터리>`). 컨테이너 안에서 일부만 실행하려면:

```bash
ctest --test-dir _out/build/<suffix> -R <정규식> --output-on-failure
# 또는 테스트 바이너리를 Catch2 필터와 함께 직접 실행, 예:
_out/build/<suffix>/dcgmlib/src/tests/dcgmlibtests "<테스트 케이스 이름>"
```

테스트 바이너리에는 `dcgmlibtests`(dcgmlib/src/tests), `coreModuleTests`, `healthtests`, `policyModuleTests`, `diagtests`, `dcgmitests`, `nvvscoretests` 등이 있습니다. 예전 방식의 단위 테스트는 `testing/Test*.cpp`에도 있습니다.

**통합 테스트(Python)**는 `testing/python3/tests/`에 있으며 실제 GPU(또는 NVML 인젝션)가 필요합니다. 이 테스트는 `datacenter-gpu-manager-tests` 패키지로 배포됩니다. 패키지를 풀고 `cd share/dcgm_tests` 후 `sudo ./run_tests.sh`를 실행하세요. 일부만 실행하려면 `main.py -f/--filter-tests <정규식>`(`module.test_fn`과 매칭)을 쓰고, `-d <nvml id>`로 GPU 하나로 제한할 수 있습니다. 전체 테스트는 30분 이상 걸립니다.

**NVML 인젝션**(`nvml-injection/`): `NVML_INJECTION_MODE`로 선택되는 가짜 `libnvml_injection.so`로, YAML 캡처 파일(`NVML_YAML_FILE`)의 값을 사용합니다. 특정 하드웨어 없이 테스트할 때 씁니다.

## 포맷 / 린트

- C/C++: `clang-format`(`.clang-format`, `sdk/`와 `_out/`은 제외), `.clang-tidy`. Python: `autopep8`(`.pep8`), pylint(`testing/python3/pylintrc`).
- `./format-dcgm`은 전체 트리를 포맷합니다. `./validate_format.sh`는 스테이징된 파일을 검사하며, `./install_git_hooks.sh`가 설치하는 pre-commit 훅입니다.
- 업스트림 기여에는 DCO 서명(`git commit -s`)이 필요합니다.

## 아키텍처

**프로세스 구조.** `nv-hostengine`(`hostengine/`)이 데몬이며, `dcgmlib/src/`의 코어 엔진을 링크합니다. 클라이언트(`dcgmi`, Python 바인딩, 사용자 프로그램)는 `dcgmlib/dcgm_agent.h`의 공개 C API(구현은 `dcgmlib/src/DcgmApi.cpp`)를 사용하며, TCP(기본 포트 5555)나 Unix 소켓(`common/transport/`)으로 호스트 엔진과 통신하거나 엔진을 프로세스 안에 임베드해 실행합니다. 각 공유 라이브러리에서 익스포트되는 심볼은 `*.linux_def` 버전 스크립트로 관리되므로, 새 공개 API 함수는 반드시 여기에 추가해야 합니다.

**코어 엔진(`dcgmlib/src/`).** `DcgmHostEngineHandler`가 중앙 디스패처입니다. `DcgmCacheManager`는 NVML을 폴링해 감시(watch) 대상 필드 값을 캐시하고, `DcgmGroupManager`/`DcgmFieldGroup`은 엔티티 그룹과 필드 그룹을 관리합니다. MIG, GPM, IMEX, 토폴로지 등은 각각 별도의 매니저가 있습니다. 필드 ID는 `dcgmlib/dcgm_fields.h`에 정의되어 있습니다.

**모듈(`modules/`).** 기능은 `DcgmModule`(`modules/DcgmModule.h`)을 상속하고 `ProcessMessage()`를 구현하는 로드 가능한 모듈로 제공됩니다. 각 모듈은 `libdcgmmodule<name>.so.4`로 빌드되며, `DcgmHostEngineHandler`가 필요할 때 `dlopen`으로 로드합니다(ID는 `dcgm_structs.h`의 `dcgmModuleId_t`). 모듈 목록: core, nvswitch, vgpu, introspect, health, policy, config, diag, profiling(비공개 소스), sysmon, mndiag. 클라이언트 API 호출은 `dcgm_module_command_header_t` 메시지(모듈별 구조체는 `modules/<name>/dcgm_<name>_structs.h`)를 만들어 `dcgmModuleSendBlockingFixedRequest`로 보내고, 모듈은 `DcgmCoreProxy`(`modules/common`)를 통해 엔진을 다시 호출합니다. 기능을 추가할 때는 보통 공개 API + 버전 있는 구조체 → 모듈 메시지 구조체 → 모듈 핸들러 → dcgmi 명령 → Python 바인딩 → 테스트 순으로 작업합니다.

**버전이 붙은 구조체.** 모든 공개/모듈 구조체에는 `MAKE_DCGM_VERSION(type, n)`(sizeof | version<<24)으로 만든 `version` 필드가 있습니다. 구조체 레이아웃을 바꾸려면 새 버전(`_vN` typedef + `dcgm..._versionN` define)을 만들고 기존 버전도 계속 동작하도록 유지해야 합니다. 들어오는 메시지는 `DcgmModule::CheckVersion`으로 검증합니다. `testing/TestVersioning.cpp`와 `dcgmlib/src/tests/DcgmVersionTests.cpp`가 이를 검사합니다.

**진단.** diag 모듈은 별도의 `nvvs` 바이너리(`nvvs/src`, 진입점 `NvvsMain.cpp`)를 실행하며, `nvvs`는 `nvvs/plugin_src/`의 테스트 플러그인(diagnostic/gpuburn, memory, memtest, pcie, targetedpower, targetedstress, contextcreate, nvbandwidth, nccl_tests, software)을 로드합니다. 플러그인은 CUDA 메이저 버전(11/12/13)별로 빌드됩니다. 멀티노드 진단은 `modules/mndiag` + `testing/mndiag`에 있습니다.

**기타 구성 요소.** `dcgmi/`: 하위 명령마다 클래스 하나로 구성된 CLI(`Diag`, `Health`, `Policy` 등). `common/`: 공용 유틸리티(로깅, 스레드, 자식 프로세스, 전송, 직렬화). `cublas_proxy/`, `dcgmproftester/`: CUDA 부하 생성기. `sdk_samples/`: C 및 Python API 예제. `testing/python3/`: Python 바인딩(`dcgm_agent.py`, `dcgm_structs.py`, `dcgm_fields.py`, `pydcgm.py`)과 테스트 프레임워크.

**Python 바인딩은 C API와 항상 일치해야 합니다.** C 구조체, 필드, API 함수를 바꾸면 `testing/python3/dcgm_structs.py` / `dcgm_fields.py` / `dcgm_agent.py`도 함께 수정해야 합니다. 업스트림은 C만 바꾼 변경을 미완성으로 간주합니다.

## 코딩 규칙 (docs/coding_best_practices.md 기준)

- 예외 사용 금지: `Initialize()` 메서드와 `dcgmReturn_t` 반환 코드, 로깅을 사용하세요. 서드파티 예외는 발생 지점에서 잡아 처리합니다.
- C++ 스타일 캐스트만 사용하고, `const_cast`는 피하며, `reinterpret_cast`보다 `static_cast`를 우선합니다.
- 다른 디렉터리의 헤더는 `#include <...>`, 같은 디렉터리의 헤더는 `"..."`를 사용합니다.
- `.cpp` 파일에서 `using namespace` 금지, 헤더에서는 어떤 형태의 `using`도 금지합니다.
- 인자가 하나인 생성자는 `explicit`으로 선언합니다.
- 이름 규칙: 지역 변수는 camelCase, 멤버는 `m_`(기본적으로 private), 상수는 `c_`, 전역 변수는 `g_`.
- 변경 하나에는 주제 하나만 담고, 함수는 작고 단위 테스트가 가능하게 유지합니다.
