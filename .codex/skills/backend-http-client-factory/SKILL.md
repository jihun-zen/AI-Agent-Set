---
name: backend-http-client-factory
description: Native AOT ASP.NET Core 백엔드의 외부 HTTP API 연동을 구현하거나 검토합니다. IHttpClientFactory, Typed Client, 취소 전파, 타임아웃, AOT JSON 역직렬화가 필요한 경우 사용합니다.
---

# HTTP 클라이언트

1. `IHttpClientFactory` 또는 Typed Client를 사용하고 `HttpClient`를 직접 생성·보관하지 않습니다.
2. 요청마다 `HttpRequestMessage`를 만들고 응답·스트림을 적절히 해제합니다.
3. 타임아웃, `CancellationToken`, 실패 상태 코드 처리를 호출 경계까지 전파합니다.
4. JSON 응답은 `JsonSerializerContext`의 `JsonTypeInfo<T>`를 사용하는 AOT 호환 API로 역직렬화합니다.
5. 인증 정보와 민감한 요청 본문을 로그에 기록하지 않습니다. 단순 요청 시작·성공 로그도 남기지 않습니다.
6. 재시도는 멱등 요청, 오류 분류, 최대 시도 횟수가 명확한 경우에만 적용합니다.
