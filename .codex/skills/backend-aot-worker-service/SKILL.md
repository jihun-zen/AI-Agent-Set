---
name: backend-aot-worker-service
description: .NET 10 Native AOT 환경에서 주기 작업, 큐 소비자, 동기화 배치, 아웃박스 처리 작업자를 설계·구현·검토합니다. BackgroundService 수명주기, Scoped DI, 멱등성, 재시도, 다중 인스턴스 동시성, 정상 종료, 관측성과 AOT 호환성이 필요한 경우 사용합니다.
---

# Native AOT Worker Service

## 작업 유형과 전달 보장

1. 작업을 주기 실행, 큐 소비, 아웃박스 처리, 외부 시스템 동기화 중 하나로 분류합니다.
2. 처리 보장 수준을 명시합니다. 기본값은 적어도 한 번(at-least-once) 전달로 보고, 각 작업을 멱등하게 설계합니다.
3. 작업 식별자 또는 비즈니스 키를 저장해 중복 처리 여부를 판별합니다. 성공 기록은 비즈니스 변경과 같은 트랜잭션 경계에서 처리합니다.
4. 여러 프로세스·컨테이너에서 실행될 수 있으면 `SemaphoreSlim`만으로 중복 실행을 막지 않습니다. DB 잠금, 고유 제약, 큐의 소비자 의미론, 분산 잠금 중 해당 인프라에 맞는 방법을 선택합니다.

## 구현 규칙

1. `BackgroundService`를 상속하고 모든 I/O에 `stoppingToken`을 전달합니다.
2. Worker는 싱글톤입니다. Scoped 의존성은 생성자로 직접 주입하지 않고 `IServiceScopeFactory.CreateAsyncScope()` 안에서 해결합니다.
3. 주기 작업에는 작업 완료 뒤 다음 주기를 기다리는 `PeriodicTimer` 또는 명시적 스케줄을 사용합니다. 의도하지 않은 중첩 실행을 만들지 않습니다.
4. 취소 요청은 정상 종료로 처리합니다. 취소 예외를 오류로 기록하거나 종료 후 새 작업을 시작하지 않습니다.
5. 예외를 일괄적으로 무시하지 않습니다. 일시적 오류, 영구 오류, 취소를 구분하고, 일시적 오류에만 제한된 지수 백오프와 지터를 적용합니다.
6. 큐 메시지가 반복 실패하면 시도 횟수·원인·원본 식별자를 기록하고 재처리 또는 dead-letter 경로로 보냅니다.
7. 외부 호출과 DB 변경 사이에는 재시도 가능한 경계를 둡니다. 분산 트랜잭션을 가정하지 말고 필요하면 아웃박스·보상·상태 전이 패턴을 사용합니다.
8. 처리량, 마지막 성공 시각, 실패 수, 대기열 지연을 관측합니다. 준비 상태와 작업 상태를 분리한 헬스 체크를 기존 운영 규칙에 맞춰 제공합니다.

## Native AOT 규칙

1. 런타임 리플렉션, `dynamic`, 런타임 코드 생성을 사용하지 않습니다.
2. JSON 페이로드와 오류 모델은 `JsonSerializerContext`에 등록하고 `JsonTypeInfo<T>` 오버로드로 직렬화합니다.
3. HTTP 연동에는 등록된 `IHttpClientFactory` 또는 Typed Client를 사용합니다.
4. ADO.NET 리소스는 `await using`으로 해제하고, 명령·트랜잭션·리더에 취소 토큰을 전달합니다.

## 최소 구조

```csharp
public sealed class SyncWorker(
    IServiceScopeFactory scopeFactory,
    ILogger<SyncWorker> logger) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using var timer = new PeriodicTimer(TimeSpan.FromMinutes(1));

        while (await timer.WaitForNextTickAsync(stoppingToken))
        {
            try
            {
                await using var scope = scopeFactory.CreateAsyncScope();
                var job = scope.ServiceProvider.GetRequiredService<ISyncJob>();
                await job.RunAsync(stoppingToken);
            }
            catch (OperationCanceledException) when (stoppingToken.IsCancellationRequested)
            {
                break;
            }
            catch (TransientWorkerException ex)
            {
                logger.LogWarning(ex, "동기화 작업의 일시적 실패");
            }
            catch (Exception ex)
            {
                logger.LogError(ex, "동기화 작업 실패");
            }
        }
    }
}
```

`ISyncJob.RunAsync`는 멱등성, 재시도, 데이터 변경의 트랜잭션 경계를 소유합니다. Worker 자체에 비즈니스 로직이나 Scoped 서비스를 보관하지 않습니다.

## 검증

1. 단일 실행, 중복 전달, 일시적 실패, 영구 실패, 취소·정상 종료 시나리오를 테스트합니다.
2. 다중 인스턴스에서 중복 실행 방지 수단을 검증합니다.
3. 기존 포맷·테스트 명령을 실행한 뒤 `dotnet publish -c Release /p:PublishAot=true`로 AOT 게시를 검증합니다.

