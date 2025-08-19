---
title: 'SemVer는 이제 그만! HeadVer로 애플리케이션 버전 관리를 개선했어요'
date: 2025-08-19
description: '복잡한 SemVer 대신 직관적인 HeadVer 기반 버전 관리 시스템을 도입한 경험담이에요. MSBuild, PowerShell/Bash 스크립트, AOT 빌드를 결합하고 활용하여 사람의 손이 덜 가는 자동화된 버전 관리 전략을 만들어봤어요.'
categories: [Dev]
tags: [.NET, 버전관리, HeadVer, MSBuild, AOT, PowerShell, Bash, 자동화, CI/CD, AssemblyInfo, 버전자동증가, 빌드자동화]
index_img: img/202508-headver-apply/headver.jpg
keywords: ['.NET version management', 'HeadVer system', 'MSBuild', 'AOT build versioning', 'PowerShell build script', 'Bash build script', 'AssemblyInfo auto-update', '버전 자동 관리', '.NET 빌드 자동화', 'MSBuild Directory.Build.targets', 'Directory.Build.targets']
---

# SemVer는 이제 그만! HeadVer로 애플리케이션 버전 관리를 개선했어요

> **TL;DR**: SemVer의 복잡함에 지친 개발자를 위한 HeadVer 기반 버전 관리 시스템 도입기예요. MSBuild와 PowerShell/Bash 스크립트를 활용해 AOT 빌드와 함께 동작하는 자동화된 버전 관리 전략을 만들어봤어요.

<br>

# 왜 SemVer에서 벗어나려 했을까요?

### SemVer의 한계
기존의 Semantic Versioning (SemVer)은 `major.minor.patch` 형태로 버전을 관리하는 표준적인 방식이에요. 하지만 실제 개발 현장에서는... 익숙해서 그냥 쓰긴 하지만 그것을 토대로 알 수 있는 정보는 사실 많지 않은 것 같아요. 올해 몇번째 빌드파일인지, hotfix가 몇번 일어났는지, 제품 출시 주기는 어느 정도로 가져가고 있는지 등 은 SemVer 만으로는 알기 어렵죠. 복잡한 내부 약속이 있다면 알아차릴 수 있겠지만 한계가 있는 것은 분명해요.

```bash
# 이런 식의 버전 번호가 익숙하신가요?
1.2.1.456
2.0.1.2610
3.1.0.12
0.0.12.623
```

**몇가지 한계점들이 있었어요:**
- SemVer 내부에 `major.minor.patch.build` 중 어느 것이 무엇인지 기억하기가 너무 어려웠어요.
- SemVer에는 개발자가 직접 관리해야 하는 관리포인트가 꽤 있어요.

SemVer의 한계점 외에도,
- 운영하고 있는 CI/CD 툴에서 제공하는 환경에 최적화되어 있는 파이프라인이라면 CI/CD 툴이 변경되면 버전 관리가 깨질 위험이 있어요.
- AOT 빌드에서 별도 파일이 생성되지 않아야 하는 상황에서 버전 관리를 해야 하는 부담이 있었어요.

### HeadVer의 매력
HeadVer는 **Head-based Versioning**의 줄임말로, Git의 HEAD 커밋을 기반으로 버전을 관리하는 방식이에요.

예전에 영재님 (LINE, ABC Studio)을 몇번 직접 뵙고 기술적인 경험을 공유할 수 있는 기회가 있었는데 HeadVer에 대해서 `우리 조직에 접목해보고 싶다`는 생각을 했었어요.
https://techblog.lycorp.co.jp/ko/headver-new-versioning-system-for-product-teams

**장점들이 있어요:**
- 직관적이고 의미있는 버전 번호를 만들 수 있어요.
- 자동화된 버전 관리가 가능해요.
- Git과의 자연스러운 연동이 돼요.
- 개발자 개입을 최소화할 수 있어요.

<br>

# 우리만의 HeadVer 변형: YearMonth 기반 접근

근데 저는 HeadVer를 그대로 쓰지는 않고 일부 변형해서 적용했어요. 매년 주 번호가 1~52 또는 53 까지 나올 수 있는데 기존 HeadVer 의 YearWeek는 yy + 주번호 였는데, 이 주 번호라는 것이 다소 직관적이지 않았고, 해당 주 번호가 2분기였는지, 3분기였는지, 초여름이었는지, 늦은 여름이었는지 를 재빠르게 알아차리기 어려웠어요. 그래서 저는 YearWeek가 아닌 YearMonth (yyMM) 형태로 변형하여 적용해보았어요.

### 기존 HeadVer vs 우리 방식
```csharp
// 기존 HeadVer: YearWeek 기반
// E.g., 2025년 26주차 → "2526" (대략 6월 중순~말경)

// 우리 방식: YearMonth 기반
// E.g., 2025년 8월 → "2508"
```

**YearMonth를 선택한 이유가 있어요:**
- 월 단위가 더 직관적이었어요.
- 빈도를 조절할 수 있었어요 (주간은 너무 자주, 연간은 너무 느렸어요)
- 비즈니스 사이클과 잘 맞았어요. (매 분기, 매월 마다 달성해야 할 업무와 연관짓기 용이)

<br>

# 구현 과정: 시행착오의 연속

그것을 어떻게 구현할지에 대해서는 여러 고민들이 있었어요.

### 1차 시도: 단순 텍스트 파일

이건 좀 아니죠?ㅋ 개발자가 배포 전에 항상 수정해줘야 하잖아요.

```bash
# version.txt
1.2508.27
```

**문제점들이 있었어요:**
- 파일이 별도로 존재해야 했고,
- AOT 빌드 시 바이너리에 포함되지 않았어요.
- 수동 관리가 너무 번거로운 것은 당연했고요.

### 2차 시도: JSON 설정 파일
```json
{
  "version": {
    "major": 1,
    "yearMonth": 2508,
    "build": 27
  }
}
```

**문제점들이 여전했어요:**
- 여전히 별도 파일이 필요했어요.
- 파싱 로직을 추가해야 했어요.
- 런타임 오버헤드가 있었어요. (읽고 쓸 때 직렬화, 역직렬화 발생)

### 3차 시도: Git 태그 기반
```bash
git tag v1.2508.27
git describe --tags --always
```

**문제점들이 있었어요:**
- Git에 의존해야 했어요. (SVN 이라면 ???)
- CI/CD 환경이 변경되면 문제가 생겼어요.
- 최대한 심플한 Git Flow를 채택하고 있는데, 태그 때문에 너무 복잡해질 것 같았어요.

<br>

# 최종 해결책: MSBuild + 스크립트 + 정적 클래스

### 핵심 아이디어

**"소스 코드 안에 버전 정보를 직접 넣고, 빌드 시 자동으로 업데이트하자!"**

### 구현 구조
```
[Directory.Build.targets] (MSBuild)
    ↓
[bump-build.ps1] or [bump-build.sh] (빌드 OS별 스크립트)
    ↓
[AssemblyInfo.cs] (정적 클래스)
    ↓
AOT 빌드
    ↓
단일 바이너리
```

<br>

# 핵심 구현 코드

### 1. MSBuild 설정 (Directory.Build.targets)

BeforeTargets 항목에 "BeforeBuild" 또는 "BeforePublish" 을 넣어서 빌드될 때마다 작동할지, 게시할 때 마다 작동할지를 결정할 수 있어요.

사실 아래 내용만 보면 심플한데, 이런 저런 복잡한 시도들을 했다가 모두 원복하고 액기스만 남겨놓은 거에요.ㅋ 이런 저런 코드 넣고 시도하다가 복잡해질수록 디버깅이 어려워져서 핵심 내용만 남겨놓고 구체적인 처리는 shell 파일(pwsh / sh) 안에서 처리하는 쪽으로 방향을 수정했어요.

```xml
<Project>
  <Target Name="BumpBuild" BeforeTargets="BeforePublish" 
          Condition=" '$(MSBuildProjectName)' == 'AB' ">
    <!-- Windows: PowerShell 7 이상 -->
    <Exec Command="pwsh -ExecutionPolicy Bypass -File &quot;$(MSBuildThisFileDirectory)bump-build.ps1&quot;" 
          Condition=" '$(OS)' == 'Windows_NT' " />
    
    <!-- Linux/WSL -->
    <Exec Command="bash &quot;$(MSBuildThisFileDirectory)bump-build.sh&quot;" 
          Condition=" '$(OS)' != 'Windows_NT' " />
  </Target>
</Project>
```

**핵심 포인트들이 있어요:**
- `BeforePublish` 타겟에서 실행돼요.
- OS별 스크립트를 자동으로 선택해요.
- 프로젝트별로 조건을 설정할 수 있어요.

### 2. PowerShell 스크립트 (bump-build.ps1)
```powershell
# 현재 날짜 yyMM (예: 2508)
$yearMonth = (Get-Date -Format "yyMM")

# Build 번호 계산
if ($yearMonth -ne $oldYearMonth) {
    $newBuild = 1  # 월이 바뀌면 1부터 시작
} else {
    $newBuild = $oldBuild + 1  # 같은 월이면 증가
}

# Git SHA (7자리 short hash)
$gitSha = (git rev-parse --short=7 HEAD).Trim()
```

### 3. 정적 클래스 (AssemblyInfo.cs)
```csharp
public static class AssemblyInfo
{
    public static int HeadVer_Major { get; } = 1;           // 개발자 직접 수정
    public static int HeadVer_YearMonth { get; } = 2508;    // 자동 생성
    public static int HeadVer_Build { get; } = 27;          // 자동 증가
    public static string HeadVer { get; } = $"{HeadVer_Major}.{HeadVer_YearMonth}.{HeadVer_Build}";
    
    // Git SHA 정보
    public static string GitSha { get; } = "64290d8";
    
    // 다양한 버전 속성들
    public static string AssemblyVersion { get; } = $"{HeadVer_Major}.{HeadVer_YearMonth}.0.0";
    public static string FileVersion { get; } = $"{HeadVer_Major}.{HeadVer_YearMonth}.{HeadVer_Build}";
    public static string ProductVersion { get; } = $"{HeadVer_Major}.{HeadVer_YearMonth}.{HeadVer_Build}";
    public static string InformationVersion { get; } = $"{HeadVer_Major}.{HeadVer_YearMonth}.{HeadVer_Build}+{GitSha}";
}
```

<br>

# 버전 값들의 의미와 용도

### 버전 체계 비교
| 속성 | 형식 | 예시 | 용도 |
|------|------|------|------|
| **HeadVer** | `1.2508.27` | 메인 버전 표시 | 사용자에게 보여주는 버전 |
| **AssemblyVersion** | `1.2508.0.0` | CLR 바인딩 | 어셈블리 로딩, GAC |
| **FileVersion** | `1.2508.27` | 파일 속성 | Windows 탐색기 표시 |
| **ProductVersion** | `1.2508.27` | 제품 버전 | 설치 프로그램 등 |
| **InformationVersion** | `1.2508.27+64290d8` | 상세 정보 | NuGet, dotnet --info |

### Git SHA의 활용
```bash
# Git SHA를 통한 정확한 빌드 식별
1.2508.27+64290d8  # 2025년 8월, 27번째 빌드, 커밋 64290d8
```

<br>

# CI/CD 도구 의존성을 피한 이유

### 기존 방식의 문제점

```yaml
# GitHub Actions 예시
- name: Bump version
  run: |
    echo "VERSION=${{ github.run_number }}" >> $GITHUB_ENV
```

**문제점:**
- CI/CD 도구 변경 시 버전 관리 깨짐
- 로컬 빌드 시 버전 정보 부재
- 도구별 설정 복잡성

### 우리 방식의 장점
- **MSBuild 표준**: 모든 .NET 환경에서 동작
- **로컬 빌드 지원**: 개발자 PC(Win/Mac), WSL에서도 동일하게 동작
- **도구 독립적**: CI/CD 도구와 무관하게 동작하므로 의존성 없음
- **AOT 호환**: 네이티브 빌드에서도 문제 없음
- **관리용이성**: Major 값만 변경해주면 됨

<br>

# Publish 결과

### 스크립트 출력 결과
```bash
$ pwsh -ExecutionPolicy Bypass -File bump-build.sh

Updating AssemblyInfo at C:\source\AirBridge\AirBridge.Core\Model\AssemblyInfo.cs
Updated -> YearMonth=2508, Build=28, GitSha=64290d8

# 빌드 후 확인
$ dotnet build
$ dotnet run --project AirBridge -- --version
AirBridge v1.2508.28 (Build: 28, Git: 64290d8)
```

### 사족: ASCII Art 만들어서 넣어봄

이번에 구글링해보다가 처음 알게된건데 이런걸 ASCII Art라고 부르더라구요?
(회사 업무라서 애플리케이션 이름은 숨기고 엉뚱한 이름으로 대체하였어요)

```ps
$ pwsh -ExecutionPolicy Bypass -File bump-build.ps1

  ___    _    _    _    ____    _    ____
 / _ \  / \  | |  | |  / ___|  / \  |  _ \
| | | |/ _ \ | |  | | | |     / _ \ | |_) |
| |_| / ___ \| |__| | | |___ / ___ \|  _ <
 \___/_/   \_\_____/   \____/_/   \_\_| \_\

     AB Version Management
     =====================
```

<br>

# AOT 빌드와의 완벽한 조화

### AOT 빌드의 장점
```xml
<PropertyGroup>
  <PublishAot>true</PublishAot>
  <IsAotCompatible>true</IsAotCompatible>
</PropertyGroup>
```

**버전 관리 관점에서의 이점들이 있어요:**
- 모든 버전 정보가 단일 바이너리에 포함돼요.
- 별도 파일이 필요 없어요.
- 배포 시 파일 누락 위험을 제거할 수 있어요.
- 런타임 성능이 향상돼요.

### 메모리 내 버전 정보 접근

```csharp
// 런타임에 버전 정보 확인
Console.WriteLine($"현재 버전: {AssemblyInfo.HeadVer}");
Console.WriteLine($"Git 커밋: {AssemblyInfo.GitSha}");
Console.WriteLine($"빌드 날짜: {AssemblyInfo.HeadVer_YearMonth}");
```

## 버전 관리 자동화의 효과

### 개발자 경험 개선
- **Major 버전만 수동으로 관리하면 돼요** (1 → 2 → 3)
- **YearMonth는 자동으로 생성돼요** (2508, 2509, 2510...)
- **Build 번호도 자동으로 증가해요** (1, 2, 3...)
- **Git SHA도 자동으로 업데이트돼요** (최종 커밋의 7 digits short SHA)

### 빌드 프로세스 간소화

```bash
# 기존: 수동 버전 관리
# 1. version.txt 수정
# 2. AssemblyInfo.cs 수정  
# 3. 빌드 실행

# 현재: 자동 버전 관리
dotnet publish -o Release -r linux-x64 # 모든 것이 자동으로 처리됨
```

<br>

# 후기: 산으로 가지 않기 위한 노력

배보다 배꼽이 더 커지면 안된다!

### 핵심 원칙
1. **단순함 유지**: 복잡한 도구나 의존성 지양
2. **표준 활용**: MSBuild, .NET 표준 기능 최대한 활용
3. **자동화**: 반복 작업은 스크립트로 자동화
4. **AOT 친화적**: 네이티브 빌드와의 호환성 고려

### 배운 점
- **과도한 엔지니어링은 금물**: 간단한 해결책이 최고
- **MSBuild의 강력함**: .NET 생태계의 숨겨진 보석
- **크로스 플랫폼 고려**: Windows/Linux 모두 지원
- **Git과의 자연스러운 연동**: 커밋 기반 버전 관리의 편리함

<br>

# 마무리

SemVer의 복잡함에서 벗어나 HeadVer 기반의 직관적이고 자동화된 버전 관리 시스템을 구축하는 과정에서 느끼고 배웠던 점들이 있어요.

**핵심 성과**
- **개발자 경험 향상**: 버전 관리 번거로움을 제거했어요.
- **자동화 구현**: 빌드 시 자동으로 버전이 업데이트돼요.
- **AOT와 호환돼요**: 단일 바이너리에 모든 정보가 포함돼요.
- **크로스 플랫폼 지원**: Windows/Mac/Linux 모두에서 동작해요.
- **도구 독립적**: CI/CD 도구가 변경되어도 영향받지 않아요.

**"고작 버전 하나 관리한다고 산으로 가지 말자"**는 생각을 계속하면서 작업했었어요. 복잡한 도구보다는 간단하고 확실한 해결책이 장기적으로 더 가치 있지 않나 생각해보았어요.

---

**💡 이 글을 읽고 궁금한 점이나 추가로 궁금한 내용이 있다면 댓글로 남겨주세요!**

**🔗 관련 링크**
- [MSBuild Directory.Build.targets 공식 문서](https://docs.microsoft.com/en-us/visualstudio/msbuild/customize-your-build)를 참고해보세요.

- [.NET AOT 컴파일 가이드](https://docs.microsoft.com/en-us/dotnet/core/deploying/native-aot/)도 도움이 될 거예요.

- [Git 버전 관리 전략](https://git-scm.com/book/ko/v2/Git%EB%A7%9E%EC%9D%80-%EB%B2%84%EC%A0%84%EA%B4%80%EB%A6%AC)도 함께 읽어보시면 좋을 것 같아요.

<br>

---
`eod`