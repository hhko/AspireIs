# Aspire Is

## 설치
### 참고 자료
- .NET Aspire setup and tooling: https://learn.microsoft.com/en-us/dotnet/aspire/fundamentals/setup-tooling?tabs=visual-studio

### 사전 준비
- .NET 8.0: https://dotnet.microsoft.com/en-us/download/dotnet/8.0
- Docker Desktop: https://www.docker.com/products/docker-desktop/
- Visual Studio Code: https://code.visualstudio.com/

### Aspire 설치
```shell
dotnet workload update
dotnet workload install aspire
dotnet workload list

설치된 워크로드 ID      매니페스트 버전                       설치 원본
-----------------------------------------------------------------------
aspire                 8.0.0-preview.2.23619.3/8.0.100      SDK 8.0.100
```

## 프로젝트
### 프로젝트 생성
```shell
dotnet new list

템플릿 이름                        약식 이름           언어    태그
---------------------------------  ----------------  ------  ---------------------------------------------------------
.NET Aspire Application            aspire            [C#]    Common/.NET Aspire/Cloud/Web/Web API/API/Service
.NET Aspire Starter Application    aspire-starter    [C#]    Common/.NET Aspire/Blazor/Web/Web API/API/Service/Cloud

# 기본 프로젝트 템플릿
dotnet new aspire

# 예제 프로젝트 템플릿
dotnet new aspire-starter -o HelloAspireStarter
# HelloAspireStarter.ApiService              // WebApi 프로젝트
# HelloAspireStarter.AppHost                 <-- 시작 프로젝트
# HelloAspireStarter.ServiceDefaults
# HelloAspireStarter.Web
```

### 디버깅
- VSCode 디버깅 환경 구축
  - 실행 > 구성 추가...
  - Run > Add Configuration...

# 디버깅 Web Browser 자동 열기
```
info: Aspire.Dashboard.DashboardWebApplication[0]
      Now listening on: http://localhost:15164
Aspire.Dashboard.DashboardWebApplication: Information: Now listening on: http://localhost:15164
info: Aspire.Dashboard.DashboardWebApplication[0]
      OTLP server running at: http://localhost:16178
Aspire.Dashboard.DashboardWebApplication: Information: OTLP server running at: http://localhost:16178
```
```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": ".NET Core Launch (console)",
            "type": "coreclr",
            "request": "launch",
            "preLaunchTask": "build",
            "program": "${workspaceFolder}/HelloAspireStarter.AppHost/bin/Debug/net8.0/HelloAspireStarter.AppHost.dll",
            "args": [],
            "cwd": "${workspaceFolder}/HelloAspireStarter.AppHost",
            "console": "internalConsole",
            "stopAtEntry": false,
            "serverReadyAction": {            // 스크립트
                "action": "openExternally",
                // Now listening on: http://localhost:15164
                "pattern": "\\bNow listening on:\\s+(http?://\\S+)"
           },
           "env": {                          // 환경 변수
            "ASPNETCORE_ENVIRONMENT": "Development"
          }
        }
    ]
}
```

## Metrics 정의
### 참고 자료
- Measure Your Application’s Performance in .NET: https://www.youtube.com/watch?v=8kDugxr3Hdg&t=602s

### 코드 요약
- HelloAspireStarter.ServiceDefaults 프로젝트
  - Extensions.cs
    - BuiltInMeters 추가
- HelloAspireStarter.ApiService 프로젝트
  - Program.cs
    - 서비스 등록
    - MinimalApi Metrics 추가
  - WeatherMetrics.cs
    - 사용자 정의 Metrics 정의

### 코드
```cs
// HelloAspireStarter.ServiceDefaults
//  - Extensions.cs
private static MeterProviderBuilder AddBuiltInMeters(this MeterProviderBuilder meterProviderBuilder) =>
    meterProviderBuilder.AddMeter(
        "Microsoft.AspNetCore.Hosting",
        "Microsoft.AspNetCore.Server.Kestrel",
        "System.Net.Http",
        "HelloAspireStarter.ApiService");        // <-= MeterName 이름
```

```cs
// HelloAspireStarter.ApiService
//   - Program.cs: 서비스 등록
builder.Services.AddMetrics();
builder.Services.AddSingleton<WeatherMetrics>();
```
```cs
// HelloAspireStarter.ApiService
//   - Program.cs: MinimalApi Metrics 추가
app.MapGet("/weatherforecast", async (WeatherMetrics weatherMetrics) =>
{
    using var _ = weatherMetrics.MeasureRequestDuration();

    try
    {
        await Task.Delay(Random.Shared.Next(5, 100));

        var forecast = Enumerable.Range(1, 5).Select(index => ...).ToArray();
        return forecast;
    }
    finally
    {
        weatherMetrics.IncreaseWeatherRequestCount();
    }
});
```
```cs
// HelloAspireStarter.ApiService
//   - WeatherMetrics.cs: 사용자 정의 Metrics 정의
using System.Diagnostics.Metrics;

namespace HelloAspireStarter.ApiService;

public class WeatherMetrics
{
    public const string MeterName = "HelloAspireStarter.ApiService";

    private readonly Counter<long> _weatherRequestCounter;
    private readonly Histogram<double> _weatherRequestDuration;

    public WeatherMetrics(IMeterFactory meterFactory)
    {
        var meter = meterFactory.Create(MeterName);
        _weatherRequestCounter = meter.CreateCounter<long>("http.weather_requests.count");
        _weatherRequestDuration = meter.CreateHistogram<double>("http.weather_requests.duration", "ms");
    }

    public void IncreaseWeatherRequestCount()
    {
        _weatherRequestCounter.Add(1);
    }

    public TrackedRequestDuration MeasureRequestDuration()
    {
        return new TrackedRequestDuration(_weatherRequestDuration);
    }
}

public class TrackedRequestDuration : IDisposable
{
    private readonly long _requestStartTime = TimeProvider.System.GetTimestamp();
    private readonly Histogram<double> _histogram;

    public TrackedRequestDuration(Histogram<double> histogram)
    {
        _histogram = histogram;
    }

    public void Dispose()
    {
        var elapsed = TimeProvider.System.GetElapsedTime(_requestStartTime);
        _histogram.Record(elapsed.TotalMilliseconds);
    }
}
```

## TODO
- 디버깅 종료 인식: Ctrl+C

