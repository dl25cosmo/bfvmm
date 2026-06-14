# Bareflank Hypervisor 
> Ubuntu 20.04 환경에서 Bareflank Hypervisor rc 2.0.4를 소스코드부터 직접 빌드한 연구 프로젝트입니다.  
<img width="1852" height="1016" alt="Image" src="https://github.com/user-attachments/assets/0422c4c5-a0a5-45cd-95c0-e911270271e5" />
---

## 목차
1. [개요](#개요)
2. [빌드 환경](#빌드-환경)
3. [빌드 설정](#빌드-설정)
4. [에러 해결 과정](#에러-해결-과정)
5. [빌드 결과](#빌드-결과)
6. [미해결 이슈](#미해결-이슈)
7. [학습 내용](#학습-내용)
8. [수정된 파일 요약](#수정된-파일-요약)

---

## 개요

Bareflank는 x86 하드웨어 가상화 기술(Intel VT-x)을 기반으로 하는 오픈소스 하이퍼바이저 연구 프레임워크입니다. 이 프로젝트는 RC 2.0.4 릴리즈를 x86_64 아키텍처 + UEFI/EFI 환경을 타겟으로 소스에서 직접 빌드하는 것을 목표로 했습니다.

빌드 과정에서 glibc vs newlib 호환성 문제, 로케일 함수 의존성, 매크로 네임스페이스 충돌, 링커 undefined symbol 등 5가지 핵심 컴파일 에러를 만났고, 인공지능과의 질답을 통해 각각을 분석하여 빌드에 성공하였습니다.

---

## 빌드 환경

| 항목 | 내용 |
|------|------|
| **OS** | Ubuntu 20.04 LTS |
| **컴파일러** | Clang 11.0.0-2~ubuntu20.04.1 |
| **타겟 아키텍처** | x86_64-pc-linux-gnu |
| **빌드 시스템** | CMake + Ninja |
| **LLVM 툴체인** | llvm-project 소스에서 직접 빌드 |

---

## 빌드 설정

```bash
cmake -B build -S . \
  -G Ninja \
  -DCMAKE_BUILD_TYPE=Release \
  -DBUILD_STATIC_LIBS=ON \
  -DENABLE_BUILD_EFI=ON \
  -DBF_VMM_C_COMPILER=/home/llvm-project/llvm-project-build/bin/clang \
  -DBF_VMM_CXX_COMPILER=/home/llvm-project/llvm-project-build/bin/clang++ \
  -DBF_VMM_LD=/home/llvm-project/llvm-project-build/bin/ld.lld \
  -DCMAKE_CXX_LINK_EXECUTABLE="/home/llvm-project/llvm-project-build/bin/ld.lld" \
  -DLIBCXX_ENABLE_THREADS=OFF \
  -DLIBCXXABI_ENABLE_THREADS=OFF \
  -DNEWLIB_DISABLE_LOCALE=ON \
  -DBF_VERBOSE=ON

ninja -C build
```

---

## 에러 해결 과정

### 에러 #1: vasprintf 함수 미정의

**에러 메시지:**
```
hypervisor-rc2.0.4/build/cache/libcxx/include/__bsd_locale_fallbacks.h:122:17:
error: use of undeclared identifier 'vasprintf'
int __res = vasprintf(__s, __format, __va);
```

**원인 분석:**  
`vasprintf()`는 glibc의 GNU 확장 함수로, Bareflank VMM 툴체인이 사용하는 newlib에는 존재하지 않습니다. libc++가 POSIX 환경을 가정하고 로케일 관련 포맷 함수에 `vasprintf()`를 사용하려 했기 때문에 발생했습니다.

**해결 방법** (`__bsd_locale_fallbacks.h` 수정):

```cpp
// 수정 전
inline int __libcpp_asprintf_l(char **__s, locale_t __l,
                               const char *__format, ...) {
    va_list __va;
    va_start(__va, __format);
    __libcpp_locale_guard __current(__l);
    int __res = vasprintf(__s, __format, __va);
    va_end(__va);
    return __res;
}

// 수정 후
inline int __libcpp_asprintf_l(char **__s, locale_t /*__l*/,
                               const char *__format, ...) {
    va_list __va;
    va_start(__va, __format);
    // 프리스탠딩 환경에서 로케일 기능 비활성화
    va_end(__va);
    return -1; // VMM 컨텍스트에서 사용되지 않음
}
```

**근거:** VMM/UEFI 환경에서 로케일 특화 포맷팅은 불필요합니다. -1을 반환해도 Bareflank VMM은 로케일 문자열 포맷팅에 의존하지 않기 때문에 동작에 영향이 없습니다.

---

### 에러 #2: 로케일 전용 변환 함수 미정의

**에러 메시지:**
```
hypervisor-rc2.0.4/build/cache/libcxx/include/locale:739:26:
error: use of undeclared identifier 'strtoll_l'
long long __ll = strtoll_l(__a, &__p2, __base, _LIBCPP_GET_C_LOCALE);
```

**원인 분석:**  
`strtoll_l()`, `strtoull_l()`, `strtof_l()`, `strtod_l()`, `strtold_l()` 등은 POSIX 확장 함수로, newlib 같은 최소 C 라이브러리 구현에는 존재하지 않습니다.

**해결 방법** (`locale` 헤더 상단에 추가):

```cpp
#ifndef strtoll_l
#define strtoll_l(a, b, c, d) strtoll((a), (b), (c))
#endif
#ifndef strtoull_l
#define strtoull_l(a, b, c, d) strtoull((a), (b), (c))
#endif
#ifndef strtof_l
#define strtof_l(a, b, c) strtof((a), (b))
#endif
#ifndef strtod_l
#define strtod_l(a, b, c) strtod((a), (b))
#endif
#ifndef strtold_l
#define strtold_l(a, b, c) strtold((a), (b))
#endif
```

**근거:** 로케일 파라미터를 무시하고 표준 변환 함수로 리다이렉트합니다. 프리스탠딩 환경에서는 기본적으로 C 로케일을 사용하므로 동작에 영향이 없습니다.

---

### 에러 #3: remove_reference 모호한 참조

**에러 메시지:**
```
hypervisor-rc2.0.4/bfsdk/include/bfstd.h:55:26:
error: reference to 'remove_reference' is ambiguous
template<class T> struct remove_reference<T &> {
```

**원인 분석:**  
Bareflank의 커스텀 표준 라이브러리(`bfstd.h`)가 자체적으로 `remove_reference` 템플릿을 정의했는데, `<type_traits>`의 `std::remove_reference`와 충돌했습니다. 명시적인 네임스페이스 한정 없이는 컴파일러가 어느 템플릿을 쓸지 결정할 수 없습니다.

**해결 방법** (`bfstd.h` 수정):

```cpp
// 수정 전
template<class T>
constexpr T&& forward(typename remove_reference<T>::type &t) noexcept {
    return static_cast<T&&>(t);
}

// 수정 후
template<class T>
constexpr T&& forward(typename std::remove_reference<T>::type &t) noexcept {
    return static_cast<T&&>(t);
}
```

**근거:** `std::` 네임스페이스를 명시적으로 지정하여 모호성을 제거합니다. `remove_reference`가 참조되는 모든 인스턴스에 동일하게 적용했습니다.

---

### 에러 #4: log 매크로와 수학 함수 충돌

**에러 메시지:**
```
hypervisor-rc2.0.4/build/prefixes/x86_64-vmm-elf/include/math.h:109:15:
error: functions that differ only in their return type cannot be overloaded
extern double log (double);
       ~~~~~~ ^
note: expanded from macro 'log'
#define log(...) printf(__VA_ARGS__);
```

**원인 분석:**  
디버깅 매크로 `log(...)`가 `printf(...)`의 별칭으로 정의되어 있었습니다. C 전처리기가 컴파일 전에 텍스트 치환을 수행하기 때문에, `math.h`와 libc++ 헤더의 수학 함수 `log()` 선언까지 모두 치환해버렸습니다.

**해결 방법** (`log.h` 수정):

```cpp
// 수정 전
#define log(...) printf(__VA_ARGS__)

// 수정 후
#define bf_log(...) printf(__VA_ARGS__)
```

그리고 Bareflank 소스코드 전체에서 `log(...)` 호출을 `bf_log(...)`로 일괄 교체했습니다.

**근거:** 프로젝트 전용 접두사(`bf_`)를 붙여 표준 라이브러리 함수와의 네임스페이스 충돌을 방지합니다. 시스템 프로그래밍에서 매크로 오염을 피하는 표준 관행입니다.

---

### 에러 #5: 최종 링킹 단계 undefined symbol

**에러 메시지:**
```
hypervisor-rc2.0.4/build/prefixes/x86_64-vmm-elf/lib/libc.so:
undefined reference to `_jp2uc_l'
hypervisor-rc2.0.4/build/prefixes/x86_64-vmm-elf/lib/libc.so:
undefined reference to `_uc2jp_l'
hypervisor-rc2.0.4/build/prefixes/x86_64-vmm-elf/lib/libc++abi.so:
undefined reference to `__tls_get_addr'
```

**원인 분석:**
- `_jp2uc_l` / `_uc2jp_l`: newlib의 iconv 구현에서 일본어 인코딩 변환을 위한 함수로, 최소 빌드에는 포함되지 않습니다.
- `__tls_get_addr`: Thread-Local Storage(TLS) 접근 함수로, glibc의 동적 링커가 제공하지만 프리스탠딩 환경에는 없습니다.

**해결 방법** (`vcpu_factory.cpp`에 weak symbol 주입):

```cpp
extern "C" {

__attribute__((weak))
int _jp2uc_l() {
    return 0;
}

__attribute__((weak))
int _uc2jp_l() {
    return 0;
}

__attribute__((weak))
void *__tls_get_addr(void *arg) {
    return nullptr;
}

}
```

**근거:** `__attribute__((weak))`는 링커의 요구사항을 충족하는 weak symbol을 생성합니다. VMM 환경에서 일본어 로케일 변환과 TLS는 사용되지 않으므로 스텁 구현이 안전합니다.

---

## 빌드 결과

5가지 에러를 모두 해결한 후 빌드가 성공적으로 완료되었습니다:

```
[81/83] Performing build step for 'bfvmm_main_x86_64-vmm-elf'
[1/2] Building CXX object CMakeFiles/bfvmm_shared.dir/arch/intel_x64/vcpu_factory.cpp.o
[2/2] Linking CXX executable bfvmm_shared
[82/83] Performing install step for 'bfvmm_main_x86_64-vmm-elf'
[83/83] Completed 'bfvmm_main_x86_64-vmm-elf'
```

**생성된 결과물:**
- `bfvmm_shared` - 메인 하이퍼바이저 실행 파일
- `bareflank.efi` - UEFI 부팅 가능한 EFI 바이너리
- VMM 컴포넌트 정적 라이브러리

**실행 로그 (일부):**
```
now in VM
ERROR: unhandled exit reason
exit_reason: xsetbv
```
"now in VM" 출력으로 하이퍼바이저가 실제로 VM 진입에 성공했음을 확인했습니다.

---

## 미해결 이슈

빌드는 성공했으나 런타임 이슈가 남아있습니다:

**EFI 런타임 실행 문제:**  
`bareflank.efi` 바이너리가 UEFI 펌웨어 환경에서 `xsetbv` unhandled exit으로 인해 중단됩니다.

**원인 추정:**
- xsetbv VMEXIT 핸들러 미구현
- weak symbol 스텁의 런타임 동작 문제
- EFI 바이너리 생성 시 메모리 정렬 문제
- UEFI 프로토콜 초기화 누락

**향후 계획:** QEMU + OVMF 환경에서 verbose 로깅으로 디버깅 예정

---

## 학습 내용

### 크로스 컴파일 툴체인
호스트 환경과 타겟 환경의 차이 이해, LLVM/Clang 크로스 컴파일 설정, glibc vs newlib 등 복수의 C 라이브러리 구현 관리

### 임베디드 시스템 프로그래밍
표준 라이브러리 가정이 성립하지 않는 프리스탠딩 환경, 최소 런타임 환경의 제약 이해, 누락된 기능에 대한 스텁 함수 구현

### 링커 동작과 심볼 해석
weak symbol vs strong symbol, 컴파일 단위 간 심볼 가시성 관리, 복잡한 멀티 라이브러리 프로젝트의 링킹 에러 해결

### 체계적 문제 해결 방법론
에러 메시지 분석 → 근본 원인 파악 → 해결책 구현 → 증분 빌드로 검증하는 디버깅 접근법

---

## 수정된 파일 요약

| 파일 | 수정 유형 | 목적 |
|------|-----------|------|
| `__bsd_locale_fallbacks.h` | 함수 스텁 | vasprintf 의존성 제거 |
| `locale` | 매크로 정의 | locale 함수 리다이렉트 |
| `bfstd.h` | 네임스페이스 한정 | 템플릿 모호성 해결 |
| `log.h` | 매크로 이름 변경 | log() 함수 충돌 방지 |
| `vcpu_factory.cpp` | weak symbol 주입 | 링커 요구사항 충족 |

---

## 참고 자료

- [Bareflank Hypervisor Framework](https://github.com/Bareflank/hypervisor)
- [LLVM Project Documentation](https://llvm.org/docs/)
- [Newlib C Library Documentation](https://sourceware.org/newlib/)
- UEFI Specification, Version 2.10
- Intel® 64 and IA-32 Architectures Software Developer's Manual

---

*이 프로젝트는 포트폴리오 제출을 위해 작성되었으며, 기술적 문제 해결 능력과 저수준 시스템 프로그래밍 경험을 보여줍니다.*
