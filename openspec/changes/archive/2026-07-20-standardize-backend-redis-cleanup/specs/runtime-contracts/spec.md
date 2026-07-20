## ADDED Requirements

### Requirement: No legacy backend ResultCode contract
HydroCore backend SHALL use `com.siact.hydrocore.common.api.ApiResponseCode` as the canonical REST response code source. The legacy `com.siact.hydrocore.common.result.ResultCode` and `IErrorCode` types MUST NOT remain in production code after they are confirmed unused, and they MUST NOT be reintroduced as frontend-facing API error code contracts.

#### Scenario: Legacy result code types are absent
- **WHEN** backend production source is scanned after the cleanup
- **THEN** `com.siact.hydrocore.common.result.ResultCode` and `com.siact.hydrocore.common.result.IErrorCode` are absent

#### Scenario: API responses continue to use ApiResponseCode
- **WHEN** controllers, exception handlers, or security handlers construct REST response envelopes
- **THEN** they continue to use `ApiResponse<T>` and `ApiResponseCode` semantics rather than legacy result-code types
