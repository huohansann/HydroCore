# runtime-contracts Specification

## Purpose
TBD - created by archiving change standardize-runtime-contracts. Update Purpose after archive.
## Requirements
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

### Requirement: Frontend HTTP adapter consumes one contract
The frontend HTTP adapter SHALL be the only default layer that interprets the backend response envelope. Successful requests MUST resolve to business `data`, and failed envelope responses MUST reject with a typed HTTP error.

#### Scenario: Successful API call
- **WHEN** the backend returns the canonical envelope with `success=true`
- **THEN** the frontend HTTP adapter resolves the request promise with the envelope `data` value

#### Scenario: Failed API call
- **WHEN** the backend returns the canonical envelope with `success=false`
- **THEN** the frontend HTTP adapter rejects the request promise with an `AppHttpError` containing the backend `code`, `message`, `traceId`, HTTP status when available, and raw response context

#### Scenario: Transport error
- **WHEN** Axios receives a network error, timeout, or non-envelope transport failure
- **THEN** the frontend HTTP adapter emits the existing HTTP error event with a typed `AppHttpError` and does not pretend the request returned successful business data

### Requirement: Configurable runtime thread pools
HydroCore backend SHALL define runtime thread pools through a unified configurable mechanism. IO, CPU, background, and event thread pools MUST have explicit names, bounded queues, configurable sizing, rejection policy, and shutdown behavior.

#### Scenario: Default thread pool creation
- **WHEN** the backend application starts without deployment-specific overrides
- **THEN** it creates IO, CPU, background, and event executors using documented safe defaults and distinct thread name prefixes

#### Scenario: Environment-specific sizing
- **WHEN** deployment configuration overrides a thread pool size, queue capacity, keep-alive value, or shutdown wait setting
- **THEN** the corresponding executor uses the configured value without source code changes

#### Scenario: Rejected task behavior
- **WHEN** a thread pool reaches its configured capacity
- **THEN** the configured rejection policy is applied consistently and the behavior is documented for operators and developers

### Requirement: Accurate structured logging
HydroCore runtime logging SHALL use one SLF4J-compatible style and MUST accurately describe the operation, failure stage, relevant identifiers, and exception details for actionable troubleshooting.

#### Scenario: Exception handling log
- **WHEN** global exception handling records an exception
- **THEN** the log entry includes the operation or handler context, exception type/message, request trace identifier when available, and the exception object for stack trace capture

#### Scenario: Validation or expected denial
- **WHEN** the system rejects a request because of validation, authentication, authorization, or missing optional upstream data
- **THEN** the log level and message reflect the expected condition accurately and do not describe it as an unknown system crash

#### Scenario: Forbidden runtime console output
- **WHEN** backend runtime code records operational problems
- **THEN** it MUST NOT use `System.out.println` or `printStackTrace`; it uses the unified logging style or removes non-useful debug output

### Requirement: Contract verification gates
The change implementation SHALL include verification that protects the runtime contracts from drifting during future secondary development.

#### Scenario: Backend verification
- **WHEN** backend verification runs
- **THEN** it covers response envelope behavior, exception response behavior, thread pool configuration binding, and a static scan for forbidden runtime console output

#### Scenario: Frontend verification
- **WHEN** frontend verification runs
- **THEN** it covers successful envelope unwrapping, failed envelope rejection, transport error mapping, and the production build

