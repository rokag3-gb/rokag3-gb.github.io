---
title: '.NET Native AOT 빌드 및 WSL2 테스트 가이드 - 크로스 플랫폼 배포 최적화'
date: 2025-07-18
description: '.NET 8 Native AOT 컴파일로 빌드한 애플리케이션을 WSL2 Ubuntu에서 테스트하는 완전 가이드. 크로스 플랫폼 호환성, 성능 최적화, RHEL 배포 환경까지 실무 적용 방법을 상세히 다룹니다.'
categories: [Dev]
tags: [.NET, Native AOT, WSL2, Ubuntu, Linux, 크로스플랫폼, 성능최적화, 배포, RHEL, Docker, 컨테이너, 서버리스, JIT, 컴파일, 빌드, 개발도구, Microsoft]
index_img: img/202507-1650/dotnet-aot-wsl2-rhel.png
keywords: [.NET Native AOT, WSL2 테스트, 크로스 플랫폼 배포, .NET 8 성능 최적화, Linux .NET 배포, RHEL 호환성, 컨테이너 배포, 서버리스 .NET]
---

# .NET Native AOT로 빌드한 애플리케이션을 WSL2에서 테스트하기

## 개요

이 글에서는 .NET 8로 개발한 콘솔 애플리케이션을 **Native AOT (Ahead-of-Time)** 컴파일로 빌드하고, WSL2 Ubuntu 환경에서 테스트하는 과정을 다룹니다. 실제 배포 환경인 RHEL도 고려하였습니다.

## 배경

실제 프로덕션 환경에서는 애플리케이션을 배포할 때 다음과 같은 고려사항이 있습니다:

- **크로스 플랫폼 호환성**: Windows에서 개발한 애플리케이션이 Linux에서 정상 작동하는지 확인
- **런타임 의존성**: 배포 환경에 .NET 런타임 설치 여부
- **성능 최적화**: 빠른 시작 시간과 메모리 효율성

## 프로젝트 구조

```
AirBridge/
├── AirBridge.csproj
├── Program.cs
└── bin/
    └── Debug/net8.0/
        ├── AirBridge (실행 파일)
        ├── AirBridge.dll
        ├── AirBridge.deps.json
        └── AirBridge.runtimeconfig.json
```

## 1단계: 기본 .NET 애플리케이션 빌드 및 테스트

### 1.1 WSL2 Ubuntu 환경 설정

먼저 WSL2에 Ubuntu를 설치하고 .NET 8.0 SDK를 설치합니다:

```bash
# WSL2 Ubuntu 접속
wsl -d Ubuntu

# .NET 8.0 SDK 설치
sudo apt update
sudo apt install -y dotnet-sdk-8.0

# 설치 확인
dotnet --version
# 출력: 8.0.118
```

### 1.2 기본 빌드 및 실행

```bash
# 프로젝트 디렉토리로 이동
cd /mnt/c/source/AirBridge/AirBridge

# 빌드
dotnet build

# 실행
dotnet run
# 출력: Hello, World!
```

### 1.3 Framework-dependent vs Self-contained

기본 빌드로 생성된 파일들을 분석해보면:

```bash
# 빌드된 파일 확인
ls -la bin/Debug/net8.0/

# 실행 파일 타입 확인
file bin/Debug/net8.0/AirBridge
# 출력: ELF 64-bit LSB pie executable, x86-64, dynamically linked

# 의존성 확인
ldd bin/Debug/net8.0/AirBridge
```

이는 **Framework-dependent executable**로, 시스템에 .NET 런타임이 설치되어 있어야 합니다.

## 2단계: Native AOT 빌드 설정

### 2.1 프로젝트 파일 수정

`AirBridge.csproj` 파일에 Native AOT 설정을 추가합니다:

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net8.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
    <PublishAot>true</PublishAot>
    <RuntimeIdentifier>linux-x64</RuntimeIdentifier>
    <OptimizationPreference>Size</OptimizationPreference>
    <IlcOptimizationPreference>Size</IlcOptimizationPreference>
  </PropertyGroup>

</Project>
```

### 2.2 Native AOT 빌드 전제 조건 설치

Native AOT 빌드에는 추가적인 도구들이 필요합니다. Ubuntu와 RHEL 환경에서 각각 다른 패키지 관리자를 사용합니다:

#### Ubuntu 환경 (apt 사용)

```bash
# C/C++ 컴파일러 및 빌드 도구 설치
sudo apt update
sudo apt install -y clang build-essential

# zlib 라이브러리 개발 패키지 설치
sudo apt install -y zlib1g-dev

# NuGet 캐시 정리 (WSL2 환경에서 필요)
dotnet nuget locals all --clear
dotnet restore
```

#### RHEL/CentOS 환경 (dnf/yum 사용)

```bash
# EPEL 저장소 활성화 (RHEL의 경우)
sudo subscription-manager repos --enable codeready-builder-for-rhel-8-x86_64-rpms
# 또는 CentOS의 경우
sudo dnf install -y epel-release

# C/C++ 컴파일러 및 빌드 도구 설치
sudo dnf groupinstall -y "Development Tools"
sudo dnf install -y clang

# zlib 라이브러리 개발 패키지 설치
sudo dnf install -y zlib-devel

# NuGet 캐시 정리
dotnet nuget locals all --clear
dotnet restore
```

#### 주요 차이점

| 항목 | Ubuntu (apt) | RHEL/CentOS (dnf/yum) |
|------|-------------|----------------------|
| 패키지 관리자 | `apt` | `dnf` (RHEL 8+) / `yum` (이전 버전) |
| 개발 도구 | `build-essential` | `Development Tools` 그룹 |
| C/C++ 컴파일러 | `clang` | `clang` |
| zlib 개발 패키지 | `zlib1g-dev` | `zlib-devel` |
| 저장소 활성화 | 기본 활성화 | EPEL 저장소 추가 필요 |

## 3단계: Native AOT 빌드 및 테스트

### 3.1 Native AOT 빌드 실행

```bash
# Native AOT 빌드
dotnet publish -c Release

# 빌드 결과 확인
ls -la bin/Release/net8.0/linux-x64/publish/
```

### 3.2 빌드 결과 분석

Native AOT 빌드 결과:

```
total 4072
-rwxrwxrwx 1 rokag3 rokag3 1555952 Jul 18 15:40 AirBridge    # 실행 파일
-rwxrwxrwx 1 rokag3 rokag3 2610008 Jul 18 15:40 AirBridge.dbg # 디버그 정보
```

### 3.3 실행 테스트

```bash
# Native AOT 실행 파일 실행
./bin/Release/net8.0/linux-x64/publish/AirBridge
# 출력: Hello, World!
```

## 4단계: 배포 환경 시뮬레이션

### 4.1 독립 실행 파일 복사 및 테스트

실제 배포 환경을 시뮬레이션하기 위해 빌드된 파일들을 별도 디렉토리로 복사합니다:

```bash
# 테스트 디렉토리 생성
mkdir -p ~/test/test-root

# 빌드된 파일 복사
cp -r bin/Release/net8.0/linux-x64/publish/* ~/test/test-root/

# 복사된 파일 확인
ls -la ~/test/test-root/

# 독립 실행 테스트
cd ~/test/test-root
./AirBridge
# 출력: Hello, World!
```

## 5단계: 성능 및 크기 비교

### 5.1 파일 크기 비교

| 빌드 타입 | 실행 파일 크기 | 의존성 | 시작 시간 |
|-----------|---------------|--------|-----------|
| Framework-dependent | ~75KB | .NET 런타임 필요 | 느림 (JIT 컴파일) |
| Self-contained | ~50MB+ | 모든 의존성 포함 | 보통 |
| Native AOT | ~1.5MB | 완전 독립적 | 빠름 |

### 5.2 장단점 분석

**Native AOT 장점:**
- ✅ 빠른 시작 시간 (JIT 컴파일 없음)
- ✅ 메모리 효율성 (필요한 코드만 포함)
- ✅ 배포 간편성 (런타임 의존성 없음)
- ✅ 보안 강화 (불필요한 코드 제거)

**Native AOT 단점:**
- ❌ 빌드 시간이 오래 걸림
- ❌ 디버깅이 어려움
- ❌ 일부 .NET 기능 제한

## 6단계: RHEL 환경 호환성 검증

### 6.1 Ubuntu vs RHEL 차이점

Ubuntu와 RHEL은 다음과 같은 차이점이 있지만, .NET 애플리케이션의 경우 대부분 호환됩니다:

- **패키지 관리자**: Ubuntu (apt) vs RHEL (yum/dnf)
- **시스템 서비스**: 둘 다 systemd 사용
- **파일 시스템 구조**: 거의 동일
- **.NET 런타임**: Microsoft 공식 지원으로 동일

### 6.2 실제 RHEL 환경에서의 테스트

Native AOT로 빌드된 실행 파일은 다음과 같은 환경에서 테스트 가능합니다:

```bash
# RHEL/CentOS 환경에서
./AirBridge
# 출력: Hello, World!

# Docker RHEL 컨테이너에서
docker run --rm -v $(pwd):/app rhel/ubi8 /app/AirBridge
```

## 결론

이 과정을 통해 다음과 같은 결과를 얻을 수 있었습니다:

1. **크로스 플랫폼 호환성 확인**: Windows에서 개발한 .NET 애플리케이션이 Linux에서 정상 작동
2. **Native AOT 빌드 성공**: 완전히 독립적인 실행 파일 생성
3. **배포 환경 시뮬레이션**: WSL2를 통한 RHEL 환경 테스트
4. **성능 최적화**: 빠른 시작 시간과 메모리 효율성 달성

Native AOT는 특히 컨테이너 환경이나 서버리스 환경에서 .NET 애플리케이션의 성능과 배포 편의성을 크게 향상시킬 수 있습니다.

## 참고 자료

- [.NET Native AOT 공식 문서](https://docs.microsoft.com/en-us/dotnet/core/deploying/native-aot/)
- [Native AOT 전제 조건](https://aka.ms/nativeaot-prerequisites)
- [.NET 8.0 릴리즈 노트](https://docs.microsoft.com/en-us/dotnet/core/whats-new/dotnet-8)

---

*이 글은 .NET 8.0, WSL2 Ubuntu 24.04 환경에서 작성되었습니다.*

`eof`