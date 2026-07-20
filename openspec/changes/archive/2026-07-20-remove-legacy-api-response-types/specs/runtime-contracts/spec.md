## MODIFIED Requirements

### Requirement: Unified API response envelope
HydroCore backend APIs SHALL expose one canonical JSON response envelope for normal and error responses. The envelope MUST include `success`, `code`, `message`, `data`, and `traceId` fields with stable semantics. Frontend-facing backend API entry points SHALL return `ApiResponse<T>` directly. The codebase MUST not use `common.R`, `common.result.R`, or `com.siact.hydrocore.common.entity.ResponseEntity` in controllers, exception handlers, or security handlers for frontend responses. Upstream Feign types such as `com.siact.api...R` remain allowed only in integration code and are not frontend API return types.

#### Scenario: Successful controller response
- **WHEN** a backend controller returns a successful business payload
- **THEN** the HTTP response body uses the canonical envelope with `success=true`, a success `code`, a non-empty `message`, the payload in `data`, and the current request `traceId`

#### Scenario: Business or validation failure
- **WHEN** a backend request fails because of business validation, argument validation, authentication, authorization, or an unhandled server exception
- **THEN** the HTTP response body uses the same canonical envelope with `success=false`, an accurate `code`, an accurate `message`, `data` empty unless explicitly needed, and the current request `traceId`

#### Scenario: No legacy response wrapper output
- **WHEN** backend controllers, exception handlers, and security handlers return API results to frontend clients
- **THEN** they use `ApiResponse<T>` directly and do not return `common.R`, `common.result.R`, or `com.siact.hydrocore.common.entity.ResponseEntity`

#### Scenario: Upstream Feign integration
- **WHEN** backend integration code calls upstream services through Feign
- **THEN** it MAY continue to use `com.siact.api...R` response types and this does not count as a frontend-facing API return type

#### Scenario: Static scan for legacy wrappers
- **WHEN** the backend runtime-contract scan runs
- **THEN** it finds no frontend-facing usages of `common.R`, `common.result.R`, or `com.siact.hydrocore.common.entity.ResponseEntity` in API return paths

#### Scenario: Response advice removal
- **WHEN** all backend API entry points have been migrated to the canonical response type
- **THEN** `ResponseBodyAdvice` is removed or disabled for normal API wrapping, so response structure is explicit in controller and handler code

#### Scenario: Advice-free result flow
- **WHEN** an API result is returned from frontend-facing code
- **THEN** no normal response wrapping depends on `ResponseBodyAdvice`
