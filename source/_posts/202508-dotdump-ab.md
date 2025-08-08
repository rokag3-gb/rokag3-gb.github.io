---
title: '내가 개발한 서버 프로세스 메모리가 하룻밤 새 2MB → 19MB? 장기 실행 서비스의 메모리 최적화 전략'
date: 2025-08-08
description: 'dotnet-dump로 .NET 장기 실행 서비스의 메모리 증가(2MB→19MB)를 분석하고 GC/LOH/버퍼/스레드 스택 원인과 SFTP·Serilog 최적화까지 실전 대응 전략을 정리합니다.'
categories: [Dev]
tags: [.NET, dotnet-dump, 메모리 최적화, 메모리 분석, 메모리 누수, GC, LOH, 장기 실행 서비스, 프로파일링, Serilog, SFTP, 백엔드]
index_img: img/202508-dotdump-ab/Process-Structure.png
keywords: ['.NET memory optimization', 'dotnet-dump analyze', 'dumpheap -stat', 'eeheap -gc', 'clrstack', 'long-running service memory', 'LOH threshold 85KB', 'SFTP buffer size', 'Serilog buffered false', '메모리 누수 진단', '.NET 장기 실행 서비스 최적화']
---

# 내가 개발한 서버 프로세스 메모리가 하룻밤 새 2MB → 19MB? dotnet-dump로 파헤친 .NET 장기 실행 서비스의 메모리 최적화 전략

![](img/202508-dotdump-ab/Process-Structure.png)

최근에 서버 개발을 하고 있는데, 가칭 AB 서버 앱은 앞으로 특정 호스트에서 절대 죽지 않고 계속 떠있어야 합니다. 그래서 만일의 경우를 대비하여 AB.Awaker를 개발하여 NamedPipe 방식 (IPC 기반)으로 ping/pong을 주고 받아 AB 서버 프로세스가 불능 상태이거나 프로세스가 죽었다면 AB 서버를 재기동 시키도록 개발했습니다.

그런데 이 AB 서버를 하룻밤 동안 실행하였더니 메모리 사용량이 약 2MB에서 19MB로 증가하였습니다. 본 글은 해당 현상을 `dotnet-dump`로 수집/분석한 결과와, 코드 레벨에서의 개선 제안 및 실제 적용한 내용을 정리한 기록입니다.


### 환경
- **OS**: Windows 11 x64
- **런타임**: .NET 8 기반
- **주요 구성요소**: AB, AB.Core, AB.Awaker, `Renci.SshNet` (SFTP), Serilog, Crono (Cron 백그라운드), Named Pipe (IPC 기반), YamlDotNet

### 애플리케이션 아키텍처

![](img/202508-dotdump-ab/AB_AA.png)

### 문제 현상
- 장시간(하룻밤) 실행 후 AB 서버 프로세스의 메모리가 2MB → 19MB로 증가하였습니다.
- 즉시 누수로 보이는 증거는 없으나, 장기 실행 중 버퍼/캐시/스레드 스택 등 비관리/네이티브 영역과 관리 힙 일부가 증가한 것으로 보입니다.


### 분석 절차
- 덤프 수집
```powershell
dotnet-dump collect -p <PID>
```
- 기본 분석
```powershell
dotnet-dump analyze <dump_file>
> dumpheap -stat
> eeheap -gc
> clrstack -a
```


### 분석 결과 요약
1) **전체 메모리 관점**
- GC Heap 총 크기: 약 6.36 MB (0x65d978 ≈ 6,674,808 bytes)
- 작업 관리자에서 보이는 19MB에는 네이티브 메모리, 스레드 스택, JIT 코드, 핸들 등이 포함됩니다.

2) **상위 타입 (dumpheap -stat)**
- `System.Byte[]`: 약 2.1MB (대형 I/O 버퍼)
- `System.String`: 약 2.6MB (경로/로그 메시지 등)
- `Renci.SshNet.*`: 합산 수백 KB (SFTP 연결 유지 관련 객체)
- 특징: SFTP 관련 버퍼와 문자열이 눈에 띄며, 로그 템플릿/문자열 캐시도 일정 비중을 차지합니다.

3) **세대별 힙 (eeheap -gc)**
- Gen0: ~3.5MB, Gen1: ~85KB, Gen2: ~724KB, LOH: ~2.0MB
- LOH 2MB는 주로 85KB 이상 `byte[]` 버퍼로 추정됩니다.

4) **스택 (clrstack -a)**
- 대기 상태(스케줄 대기/Task Wait)에서의 스냅샷으로, 실행 중 로직은 거의 없고 이전 작업의 버퍼/스트림/캐시가 잔류한 상태였습니다.

5) **결론(분석 단계)**
- .NET 관리 힙 자체는 크지 않으나, SFTP 라이브러리 내부 버퍼/스트림과 Serilog 문자열/템플릿 캐시, 네이티브 메모리(스레드 스택, 핸들 등)가 늘어난 것으로 보입니다.
- GC는 늘어난 커밋 메모리를 OS에 바로 반환하지 않으므로, 작업 관리자 수치가 큰 상태로 유지될 수 있습니다.

---

## 코드 개선 제안

아래 항목은 메모리 잔류(버퍼/캐시/스레드)를 줄이고, LOH 할당 및 장기 생존 객체를 최소화하기 위한 코드 레벨 권장 사항입니다.

- **SFTP 연결 수명 단축**: `SftpClient`를 필드로 오래 유지하지 않고, 매 호출마다 생성/연결/해제합니다.
  - 버퍼와 스트림이 장시간 붙잡히는 것을 방지합니다.
  - `BufferSize`를 85KB 미만(예: 32KB)으로 설정해 LOH 진입을 회피합니다.

- **LoggerFactory 직접 생성 금지**: `new LoggerFactory()`를 각 서비스에서 생성하지 않습니다.
  - DI를 통해 `ILogger<T>`를 주입하거나 `NullLogger<T>.Instance`를 사용합니다.

- **비밀키 경로 조건문 수정**:
  - 기존 OR 조건(`path is not null || File.Exists(path)`)을 AND + 널/공백 체크로 수정합니다.

- **`YamlConfigHelper` DI 사용**: `Program.cs`에서 직접 `new` 대신 DI에서 `YamlConfigHelper`를 꺼내 사용합니다.

- **`FileWatcher`에서 SFTP 서비스 지역 수명 사용**: 필드 보관 대신 루프 내부 지역 `using`으로 생성·해제합니다. 파일 단위 상세 로그는 `Debug`로 낮춰 문자열 누적을 줄입니다.

- **Serilog 파일 싱크 버퍼링 점검**: 필요 시 `buffered: false`로 설정하여 메모리 버퍼를 줄입니다.

- **`Task.Run` 남용 제거**: SFTP I/O 호출을 불필요하게 쓰레드풀로 감싸지 않습니다.


### 코드 예시 (제안)

- SFTP 호출 단위로 생성/해제 및 버퍼/타임아웃 설정
```csharp
// AzureBlobService.cs (핵심 아이디어)
private SftpClient CreateAndConnectClient()
{
    if (_config is null) throw new InvalidOperationException("Configuration is null");

    var client = CreateSftpClient(_config);
    client.ConnectionInfo.Timeout = TimeSpan.FromSeconds(15);
    client.OperationTimeout = TimeSpan.FromMinutes(5);
    client.KeepAliveInterval = TimeSpan.FromSeconds(30);
    client.BufferSize = 32 * 1024; // LOH 방지: 85KB 미만
    client.Connect();
    return client;
}

public Task UploadFile(string localFilePath, string remoteFilePath)
{
    try
    {
        using var sftp = CreateAndConnectClient(); // 클래스 전역 변수 SftpClient를 사용하지 않고 각 메소드 내부에서 매번 생성/해제 될 수 있도록 함.
        using var fileStream = File.OpenRead(localFilePath);
        sftp.UploadFile(fileStream, remoteFilePath);
        _logger.LogInformation("파일 업로드 완료: {Local} -> {Remote}", localFilePath, remoteFilePath);
        return Task.CompletedTask;
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "파일 업로드 중 오류 발생");
        throw;
    }
}
```

- LoggerFactory 직접 생성 제거 및 DI/NullLogger 사용
```csharp
// AzureBlobService.cs
using Microsoft.Extensions.Logging.Abstractions;

public AzureBlobService(ILogger<AzureBlobService> logger, AzureBlobSftpConfig config)
{
    _logger = logger ?? NullLogger<AzureBlobService>.Instance;
    _config = config ?? throw new ArgumentNullException(nameof(config));
}

public AzureBlobService(AzureBlobSftpConfig config)
    : this(NullLogger<AzureBlobService>.Instance, config) {}
```

- 비밀키 경로 조건문 수정
```csharp
if (!string.IsNullOrWhiteSpace(conf.private_key_path) && File.Exists(conf.private_key_path))
{
    using var fs = new FileStream(conf.private_key_path, FileMode.Open, FileAccess.Read);
    var privateKey = new PrivateKeyFile(fs);
    authMethods = new AuthenticationMethod[]
    {
        new PrivateKeyAuthenticationMethod(conf.username, new[] { privateKey })
    };
}
else if (!string.IsNullOrEmpty(conf.password))
{
    authMethods = new[] { new PasswordAuthenticationMethod(conf.username, conf.password) };
}
else
{
    throw new InvalidOperationException("Either password or private_key_path must be provided");
}
```

- `Program.cs`에서 `YamlConfigHelper`를 DI로 가져오기
```csharp
// Program.cs
var yamlConfigHelper = host.Services.GetRequiredService<YamlConfigHelper>();
Conf.Current = yamlConfigHelper.Load();
```

- `FileWatcher`에서 지역 수명/로그 레벨 조정
```csharp
// FileWatcher.cs (루프 내부)
var config = Conf.Current.target.azure_blob_sftp;
if (config is null) { _logger.LogError("AzureBlobSftpConfig is null."); continue; }

using var blob = new AzureBlobService(_loggerFactory.CreateLogger<AzureBlobService>(), config);
await blob.MkDir($"/{yyyyMMddHHmmss}");
await blob.UploadFile(_fi.FullPath, $"/{yyyyMMddHHmmss}/{fi.Name}");

_logger.LogDebug("_fi.Name = {Name}, _fi.FullPath = {Path}", _fi.Name, _fi.FullPath);
```

- Serilog 파일 싱크 버퍼링 비활성화
```csharp
// Program.cs (Serilog 설정 일부)
.WriteTo.File(
  "logs/AB-.log",
  rollingInterval: RollingInterval.Day,
  outputTemplate: "{Timestamp:yyyy-MM-dd HH:mm:ss.fff} [{Level:u4}] {Message:lj}{NewLine}{Exception}",
  fileSizeLimitBytes: 50 * 1024 * 1024,
  retainedFileCountLimit: 60,
  buffered: false // 즉시쓰기. 메모리 버퍼 최소화
)
```


### 실제 적용/결과
아래는 실제로 반영한 변경과 커밋/푸시 이력을 요약한 내용입니다.

- [ ] SFTP 연결을 호출 단위로 생성/해제하고 `BufferSize`/타임아웃을 설정하였습니다.
- [ ] `LoggerFactory` 직접 생성을 제거하고 DI/`NullLogger`로 대체하였습니다.
- [ ] 비밀키 경로 조건문을 수정하였습니다. (요건 버그에 가까운 이슈)
- [ ] `Program.cs`에서 `YamlConfigHelper`를 DI에서 꺼내 사용하도록 변경하였습니다.
- [ ] `FileWatcher`에서 `AzureBlobService`를 지역 수명으로 사용하고 로그 레벨을 조정하였습니다.
- [ ] Serilog 파일 싱크 `buffered: false`를 적용하였습니다.

측정 결과는 장시간 대기 시 불필요한 버퍼/캐시 잔류가 줄어들어 관리 힙 및 커밋 메모리의 상한이 안정화되는 것을 기대합니다. 실제 수치는 운영 환경에서 장시간 관찰 후 추가 개선할 예정입니다.

---

## 배운 점
- 장기 실행 서비스에서 “메모리는 언젠가 줄어든다”는 가정은 위험합니다. 버퍼/캐시/스레드 스택/네이티브 핸들 등은 쉽게 OS로 반환되지 않습니다.
- SFTP 등 네트워크 I/O 라이브러리는 연결 수명과 버퍼 크기가 메모리 발자국을 크게 좌우합니다. 호출 단위로 짧게 생성·해제하고, LOH 임계(약 85KB)를 넘지 않도록 버퍼를 설정하면 유리합니다.
- 로깅은 성능과 관찰성에 중요하지만, 과도한 문자열 포맷/버퍼링은 장시간 실행 시 메모리 잔류로 이어질 수 있습니다.
- DI 컨테이너를 적극 활용하면 리소스 수명과 책임을 명확히 하고, 불필요한 팩토리/핸들 생성을 줄일 수 있습니다.


### 부록: dotnet-dump 기본 명령 모음
```text
dotnet-dump collect -p <PID>
dotnet-dump analyze <dump_file>
> dumpheap -stat        // 타입별 객체/크기 합계
> eeheap -gc            // 세대별 힙/LOH 크기
> clrstack -a           // 스택과 대기 지점 확인
```

궁극적으로 본 개선들은 “누수”를 해결한다기보다, 장시간 대기형 서비스가 정상적으로 사용하는 캐시/버퍼/네이티브 리소스의 상한을 낮추고, 필요 이상의 생존 시간을 줄이는 데 목적이 있습니다. 운영 환경 관찰을 통해 지속적으로 조정해보려 합니다.

`eod`