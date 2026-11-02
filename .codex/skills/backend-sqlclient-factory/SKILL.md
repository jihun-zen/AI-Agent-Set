---
name: backend-sqlclient-factory
description: "Microsoft.Data.SqlClient 및 SqlClientFactory를 활용한 ADO.NET 비동기 리소스 관리(await using), 쿼리 튜닝, 트랜잭션 안전성 및 Native AOT 호환 데이터 액세스 구현 스킬입니다."
---

# SqlClientFactory & ADO.NET 비동기 데이터 액세스 스킬

이 스킬은 .NET 10 환경에서 Microsoft.Data.SqlClient 및 SqlClientFactory를 직접 사용하여 고성능 비동기 데이터베이스 연동을 구현할 때 준수해야 하는 아키텍처 규칙과 코드 작성 가이드라인을 제공합니다.

## When to Use (적용 대상)

다음과 같은 작업을 수행할 때 이 스킬을 적극적으로 참고하십시오:
- `Microsoft.Data.SqlClient`를 사용하여 직접 SQL Server 통신 모듈을 작성하거나 리팩토링할 때
- DB 연결, 명령(Command), 데이터 읽기(Reader), 트랜잭션 등의 ADO.NET 리소스를 관리할 때
- 쿼리 실행 속도 최적화, SQL Server 실행 계획(Plan Cache) 재사용을 도모할 때
- .NET 10 Native AOT 환경에서 트리밍 경고 없이 실행되는 데이터 액세스 레이어를 구축할 때
- 백그라운드 서비스 내에서 대량 동기화 및 트랜잭션 파이프라인을 작성할 때

---

## 핵심 구현 원칙 및 주의사항

### 1. `await using`을 통한 비동기 리소스 관리 (IAsyncDisposable)
ADO.NET의 핵심 리소스 객체들(`SqlConnection`, `SqlCommand`, `SqlDataReader`, `SqlTransaction` 등)은 네트워크 I/O를 수반하여 해제되는 작업이 있으므로 `IAsyncDisposable` 인터페이스를 구현하고 있습니다. 
비동기(`async/await`) 메소드 내에서는 동기식 `using` 대신 반드시 **`await using`**을 사용하여 리소스를 해제해야 합니다. 이를 통해 연결을 반환하거나 버퍼를 비우는 과정에서 스레드 풀의 스레드가 차단(blocking)되는 것을 방지합니다.

- **대상 객체**:
  - `DbConnection` (예: `SqlConnection`)
  - `DbCommand` (예: `SqlCommand`)
  - `DbDataReader` (예: `SqlDataReader`)
  - `DbTransaction` (예: `SqlTransaction`)

```csharp
// ❌ 피해야 할 동기식 using 패턴 (비동기 환경 내)
using var connection = await connectionFactory.CreateConnectionAsync(ct);
using var command = connection.CreateCommand();

// ✅ 권장하는 비동기 await using 패턴
await using var connection = await connectionFactory.CreateConnectionAsync(ct);
await using var command = CreateCommand(connection, commandText, parameters, null, commandType);
```

### 2. SQL 파라미터 바인딩 시 데이터 형식 및 크기 명시
SQL Server의 성능 최적화와 SQL Injection 방지를 위해 매개변수화된 쿼리(Parameterized Query)를 필수적으로 사용하며, 파라미터 추가 시 `SqlDbType`과 `Size`를 구체적으로 설정해야 합니다.

- **실행 계획 재사용(Plan Cache)**: 문자열(`NVarChar`, `VarChar`) 파라미터 추가 시 `Size`를 명시하지 않으면, 입력 값의 길이에 따라 매번 서로 다른 실행 계획이 생성되어 SQL Server 메모리(Plan Cache)가 오염되고 컴파일 오버헤드가 발생합니다.
- **암시적 형변환 방지**: .NET 데이터 형식과 DB 데이터 형식을 일치시켜서, SQL Server가 인덱스를 정상적으로 활용하지 못하고 인덱스 스캔(Index Scan)을 수행하는 비효율성을 차단하십시오.

```csharp
// ❌ 안 좋은 예: 타입과 크기를 생략하거나 묵시적으로 바인딩
command.Parameters.AddWithValue("@SkuId", skuId); // SqlDbType과 Size가 암시적으로 설정됨

// ✅ 좋은 예: 명시적 SqlDbType 및 Size 지정
var skuParam = new SqlParameter("@SkuId", SqlDbType.VarChar, 50) { Value = skuId };
command.Parameters.Add(skuParam);
```

### 3. CancellationToken의 철저한 전파
백그라운드 서비스의 정상적인 종료 및 취소 요청에 즉각적으로 반응하기 위해, 모든 비동기 DB 작업에 호출 환경의 `CancellationToken`을 전달해야 합니다.
- `OpenAsync(ct)`
- `ExecuteReaderAsync(ct)`
- `ExecuteNonQueryAsync(ct)`
- `ExecuteScalarAsync(ct)`
- `ReadAsync(ct)`

### 4. Native AOT 호환성을 위한 리플렉션 지양
- Entity Framework Core나 리플렉션 의존도가 높은 Dapper 계열의 가벼운 ORM은 .NET 10 Native AOT 환경에서 트리밍 경고(`IL2026`, `IL3050`)를 유발합니다.
- 데이터 매핑 시 `SqlDataReader`에서 데이터를 로우(raw) 레벨로 수동 매핑하거나, 소스 생성기 구조를 활용하십시오.
- 컬럼 인덱스 또는 `reader.GetOrdinal("ColumnName")` 값을 미리 로컬 변수에 캐싱하여 안전하고 빠른 리플렉션 프리 맵핑을 구현해야 합니다.

### 5. 안전한 비동기 트랜잭션 관리
- 트랜잭션도 `await using`을 사용하여 생명 주기를 제어합니다.
- 명령 객체(`SqlCommand`)에 트랜잭션 인스턴스를 명시적으로 지정해야 합니다. (`command.Transaction = transaction`)
- 예외 상황이 발생하면 `catch` 블록 내에서 `await transaction.RollbackAsync(ct)`를 호출하여 비동기식으로 안전하게 롤백을 마쳐야 합니다.

---

## 추천 코드 스타일 및 예제 패턴

### 1. DB 연결 팩토리 (DbConnectionFactory)
Native AOT 컴파일에 완벽히 호환되도록 `DbProviderFactories` 동적 리플렉션을 사용하는 대신, `SqlClientFactory.Instance`를 직접 사용하거나 `SqlConnection`을 직관적으로 생성하는 방식을 권장합니다.

```csharp
using System.Data.Common;
using Microsoft.Data.SqlClient;

public sealed class DbConnectionFactory(string connectionString)
{
    private readonly string _connectionString = connectionString ?? throw new ArgumentNullException(nameof(connectionString));

    public async Task<DbConnection> CreateConnectionAsync(CancellationToken ct = default)
    {
        // SqlClientFactory를 사용하는 경우
        var connection = SqlClientFactory.Instance.CreateConnection();
        if (connection == null)
        {
            throw new InvalidOperationException("SqlClientFactory가 DbConnection 인스턴스를 생성하지 못했습니다.");
        }
        
        connection.ConnectionString = _connectionString;
        await connection.OpenAsync(ct);
        return connection;
    }

    public async Task<SqlConnection> CreateSqlConnectionAsync(CancellationToken ct = default)
    {
        // 직접 SqlConnection을 사용하는 경우
        var connection = new SqlConnection(_connectionString);
        await connection.OpenAsync(ct);
        return connection;
    }
}
```

### 2. 비동기 트랜잭션 및 다중 명령(Command) 처리 패턴
```csharp
using System;
using System.Data;
using Microsoft.Data.SqlClient;

public sealed class SyncHistoryRepository(DbConnectionFactory connectionFactory)
{
    public async Task SaveSyncResultWithLogAsync(SyncHistory history, string logMessage, CancellationToken ct)
    {
        await using var connection = await connectionFactory.CreateSqlConnectionAsync(ct);
        
        // 1. 비동기 트랜잭션 시작 및 await using 선언
        await using var transaction = await connection.BeginTransactionAsync(ct) as SqlTransaction;
        if (transaction == null)
        {
            throw new InvalidOperationException("SqlTransaction을 시작할 수 없습니다.");
        }

        try
        {
            // 2. 첫 번째 명령: 동기화 히스토리 삽입
            await using var cmdHistory = connection.CreateCommand();
            cmdHistory.Transaction = transaction; // 트랜잭션 할당 필수
            cmdHistory.CommandText = """
                INSERT INTO SyncHistory (SyncId, TargetDomain, Status, SyncTime)
                VALUES (@SyncId, @TargetDomain, @Status, @SyncTime);
                """;
            
            cmdHistory.Parameters.Add(new SqlParameter("@SyncId", SqlDbType.UniqueIdentifier) { Value = history.SyncId });
            cmdHistory.Parameters.Add(new SqlParameter("@TargetDomain", SqlDbType.NVarChar, 255) { Value = history.TargetDomain });
            cmdHistory.Parameters.Add(new SqlParameter("@Status", SqlDbType.VarChar, 50) { Value = history.Status });
            cmdHistory.Parameters.Add(new SqlParameter("@SyncTime", SqlDbType.DateTime2) { Value = history.SyncTime });

            await cmdHistory.ExecuteNonQueryAsync(ct);

            // 3. 두 번째 명령: 세부 로그 기록
            await using var cmdLog = connection.CreateCommand();
            cmdLog.Transaction = transaction; // 트랜잭션 할당 필수
            cmdLog.CommandText = """
                INSERT INTO SyncLogs (SyncId, LogMessage, CreatedAt)
                VALUES (@SyncId, @LogMessage, GETUTCDATE());
                """;

            cmdLog.Parameters.Add(new SqlParameter("@SyncId", SqlDbType.UniqueIdentifier) { Value = history.SyncId });
            cmdLog.Parameters.Add(new SqlParameter("@LogMessage", SqlDbType.NVarChar, -1) { Value = logMessage }); // -1은 MAX 크기에 대응

            await cmdLog.ExecuteNonQueryAsync(ct);

            // 4. 트랜잭션 커밋
            await transaction.CommitAsync(ct);
        }
        catch (Exception)
        {
            // 5. 예외 발생 시 비동기 롤백 수행 후 에러 전파
            await transaction.RollbackAsync(ct);
            throw;
        }
    }
}

public sealed record SyncHistory(Guid SyncId, string TargetDomain, string Status, DateTime SyncTime);
```
