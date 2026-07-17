# HydroCore Slim Shell Implementation Plan

> **Doc location:** This Comet/Superpowers plan belongs in the root coordination repository under `docs/superpowers/`. Frontend/backend repositories should keep only service-local technical docs.

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Strip kiln / level / pressure business from the existing FE/BE, rename the project to HydroCore, keep digital-twin data APIs, and ship overview + trends empty shells plus system admin.

**Architecture:** In-place refactor of `kic-be` �?`hydrocore-be` and `kic-fe` �?`hydrocore-fe`. Delete kiln-bound modules; retain `system`, `device`, `sec`, `tdengine`, `iot`/`mqtt` (if non-kiln), `core`/`common`. New FE routes `overview` / `trends`; trends calls existing `api/data/queryCommonChartData`. Remove old git remotes; do not push.

**Tech Stack:** Spring Boot 2.6.13, Java 8, MyBatis-Plus, Nacos, TDengine, Vue 3, Vite, Pinia, Naive UI / Element Plus, STOMP WebSocket.

**Spec:** `docs/superpowers/specs/2026-07-17-hydrocore-slim-shell-design.md`

## Global Constraints

- No fabricated process/plant data on overview or trends empty states
- No 3D / UE twin embedding in this phase
- Do not `git push` to any remote until the user provides a new repo URL
- Keep local `.git` history; only remove `origin` (or other remotes)
- First-party Java packages end under `com.siact.hydrocore.*`; do not rename classes inside external jars (e.g. `siact-sec-api-feign`)
- Commits are local only; skip commit steps if the user has not asked to commit yet
- Working directories after Task 1: `d:\project\HydroCore\hydrocore-be`, `d:\project\HydroCore\hydrocore-fe`

## File Map (target)

| Path | Responsibility |
|------|----------------|
| `hydrocore-be/pom.xml` | artifact `hydrocore`, name HydroCore |
| `hydrocore-be/src/main/java/com/siact/hydrocore/HydrocoreApplication.java` | Boot entry |
| `hydrocore-be/.../module/system/**` | Auth / RBAC (keep) |
| `hydrocore-be/.../module/device/**` | Point mapping / realtime (keep) |
| `hydrocore-be/.../sec/**` | Twin Feign + `/api/data` (keep) |
| `hydrocore-be/.../tdengine/**` | dataCode timeseries (keep) |
| `hydrocore-be/.../module/iot/**`, `mqtt/**` | Collection base (keep if compile-clean) |
| `hydrocore-be/src/main/resources/db/hydrocore_menu_migrate.sql` | Drop kiln menus; insert overview/trends |
| `hydrocore-fe/package.json` | name `hydrocore` |
| `hydrocore-fe/src/routes/index.ts` | Only auth, overview, trends, system |
| `hydrocore-fe/src/views/overview/route.tsx` | Overview shell |
| `hydrocore-fe/src/views/trends/route.tsx` | Trends page using charts API |
| `hydrocore-fe/src/constants/RouteName.ts` | `OVERVIEW`, `TRENDS`, system enums |

---

### Task 1: Preconditions �?restore FE src, remove remotes, rename directories

**Files:**
- Rename: `d:\project\HydroCore\kic-be` �?`d:\project\HydroCore\hydrocore-be`
- Rename: `d:\project\HydroCore\kic-fe` �?`d:\project\HydroCore\hydrocore-fe`

**Interfaces:**
- Consumes: none
- Produces: directories `hydrocore-be`, `hydrocore-fe` with local git, no `origin`

- [ ] **Step 1: Confirm FE `src` exists**

Run (PowerShell):

```powershell
Set-Location d:\project\HydroCore\kic-fe
if (-not (Test-Path src)) { git restore src public types 2>$null }
(Get-ChildItem src -Recurse -File | Measure-Object).Count
```

Expected: count �?292 (or �?280)

- [ ] **Step 2: Remove git remotes (both repos)**

```powershell
Set-Location d:\project\HydroCore\kic-fe
git remote -v
git remote remove origin
# if other remotes exist: git remote remove <name>
git remote -v

Set-Location d:\project\HydroCore\kic-be
git remote -v
git remote remove origin
git remote -v
```

Expected: empty remote list (or no `origin`)

- [ ] **Step 3: Rename directories**

```powershell
Set-Location d:\project\HydroCore
Rename-Item kic-be hydrocore-be
Rename-Item kic-fe hydrocore-fe
Get-ChildItem -Name
```

Expected: `hydrocore-be`, `hydrocore-fe`, `docs`

- [ ] **Step 4: Commit (optional / if user requested)**

```powershell
# In each repo, only if user asked to commit:
# git add -A; git commit -m "chore: disconnect remote and prepare HydroCore rename"
```

---

### Task 2: Backend �?delete kiln-bound modules

**Files:**
- Delete directories under `hydrocore-be/src/main/java/com/siact/module/`:
  - `forecast`, `control`, `level`, `pressure`, `predicted`, `model`, `algorithm`, `monitoring`, `snapshot`, `process`
- Delete kiln-specific files under `module/base/` (non-exhaustive; delete all that match):
  - `*Kiln*`, `MonitorController.java`, `MonitorService*.java`, `TemperatureAlarm*`, `ControlIntervalConfig*`, `GasOperation*`, `TempCondition*`, `Rule*` (if only kiln rules), related dto/vo/mapper/xml/task
- Keep under `base/` (unless compile proves otherwise): `Dic*`, `FrontendCache*`, `ConfigFieldStore*`, `Tpl*` (re-evaluate if only kiln menus)
- Delete matching MyBatis XML under `src/main/resources/mapper/` for deleted types
- Do **not** delete: `module/system`, `module/device`, `module/iot`, `module/mqtt`, `sec`, `tdengine`, `core`, `common`, `config`

**Interfaces:**
- Consumes: Task 1 directories
- Produces: backend source tree without kiln modules; may not compile until Task 3 cleans references

- [ ] **Step 1: Delete whole kiln-bound module packages**

```powershell
Set-Location d:\project\HydroCore\hydrocore-be
$kill = @('forecast','control','level','pressure','predicted','model','algorithm','monitoring','snapshot','process')
foreach ($m in $kill) {
  $p = "src\main\java\com\siact\module\$m"
  if (Test-Path $p) { Remove-Item $p -Recurse -Force; Write-Host "removed $m" }
}
```

- [ ] **Step 2: Delete kiln-specific base types**

```powershell
Set-Location d:\project\HydroCore\hydrocore-be\src\main\java\com\siact\module\base
Get-ChildItem -Recurse -File | Where-Object {
  $_.Name -match 'Kiln|TemperatureAlarm|ControlInterval|GasOperation|TempCondition|Monitor'
} | ForEach-Object { Remove-Item $_.FullName -Force; $_.FullName }
# Also remove Rule* if only used by kiln control �?delete RuleController/IRuleService/RuleMeta* if present
```

Remove corresponding files under `src/main/resources/mapper/` with the same name stems (`KilnInfoMapper.xml`, etc.).

- [ ] **Step 3: Probe compile to list remaining errors**

```powershell
Set-Location d:\project\HydroCore\hydrocore-be
mvn -q compile -DskipTests 2>&1 | Select-Object -Last 80
```

Expected: FAIL with missing-symbol / package-not-found errors pointing at leftover imports �?use that list in Task 3.

- [ ] **Step 4: Commit (optional)**

```bash
git add -A
git commit -m "refactor: remove kiln-bound backend modules"
```

---

### Task 3: Backend �?fix compile after deletes; trim `base` / `enmus` / mqtt coupling

**Files:**
- Modify any remaining Java that imports deleted packages (search `com.siact.module.forecast|control|level|pressure|predicted|model.algorithm|monitoring|snapshot|process|Kiln`)
- Modify: `HydrocoreApplication` / current `KilnApplication.java` MapperScan if needed
- Possibly strip kiln-only scheduled tasks registered in `core`

**Interfaces:**
- Consumes: Task 2 deletions
- Produces: `mvn compile` success on current package layout (`com.siact.*`)

- [ ] **Step 1: Find leftover references**

```powershell
Set-Location d:\project\HydroCore\hydrocore-be
rg -n "module\.(forecast|control|level|pressure|predicted|monitoring|snapshot|process)|KilnInfo|KilnPublish|TemperatureAlarm|ControlInterval" src --glob "*.java"
```

- [ ] **Step 2: For each hit �?delete the file if kiln-only, or edit import/call out**

Rule: prefer deleting the dependent kiln file over resurrecting deleted modules. If `system`/`device`/`sec` needs a small util from deleted code, move that util into `common` or `device` instead of restoring the module.

- [ ] **Step 3: Compile until clean**

```powershell
mvn -q compile -DskipTests
```

Expected: exit code 0

- [ ] **Step 4: Commit (optional)**

```bash
git add -A
git commit -m "fix: clear references after kiln module removal"
```

---

### Task 4: Backend �?rename artifact, application name, bootstrap; relocate packages to `com.siact.hydrocore`

**Files:**
- Modify: `hydrocore-be/pom.xml` �?`artifactId` �?`hydrocore`, `<name>` �?`hydrocore`, description �?HydroCore / 水处�?- Modify: `src/main/resources/bootstrap.yml` (and profile ymls):
  - `spring.application.name: hydrocore`
  - Nacos `data-id` entries: `hydrocore.yml`, `hydrocore-constant.yml`, `hydrocore-config.properties` (placeholders until Nacos ready; do not keep `kiln-intelligent-control` as application name)
- Create: `src/main/java/com/siact/hydrocore/HydrocoreApplication.java`
- Delete: `src/main/java/com/siact/KilnApplication.java`
- Move trees: `com/siact/{common,config,core,module,sec,tdengine}` �?`com/siact/hydrocore/{...}`
- Global replace in `src`: `package com.siact.` �?`package com.siact.hydrocore.` and `import com.siact.` �?`import com.siact.hydrocore.` **except** do not rewrite imports that only exist in external Feign DTOs if those packages are not under this repo (if a replace breaks Feign, revert those import lines to the jar’s original package)

**Interfaces:**
- Consumes: compiling `com.siact.*` tree
- Produces: `com.siact.hydrocore.HydrocoreApplication`, service name `hydrocore`

- [ ] **Step 1: Update pom.xml identity**

In `pom.xml` set:

```xml
<groupId>com.siact</groupId>
<artifactId>hydrocore</artifactId>
<version>v2</version>
<name>hydrocore</name>
<description>HydroCore water treatment platform</description>
```

- [ ] **Step 2: Update bootstrap application name and data-ids**

```yaml
spring:
  application:
    name: hydrocore
# ...
# extension-configs / shared-configs data-id examples:
# - data-id: hydrocore.yml
# - data-id: hydrocore-constant.yml
# - data-id: hydrocore-config.properties
```

Document in a one-line comment that Nacos must provide these files (or temporarily copy from old kiln configs renamed).

- [ ] **Step 3: Move source packages (PowerShell)**

```powershell
Set-Location d:\project\HydroCore\hydrocore-be\src\main\java\com\siact
New-Item -ItemType Directory -Force -Path hydrocore | Out-Null
foreach ($d in @('common','config','core','module','sec','tdengine')) {
  if (Test-Path $d) { Move-Item $d hydrocore\ }
}
```

- [ ] **Step 4: Rewrite package / import prefixes for first-party code**

```powershell
Set-Location d:\project\HydroCore\hydrocore-be
Get-ChildItem src -Recurse -Include *.java,*.xml,*.yml | ForEach-Object {
  $c = Get-Content $_.FullName -Raw -Encoding UTF8
  $n = $c -replace 'package com\.siact\.(?!hydrocore)', 'package com.siact.hydrocore.' `
          -replace 'import com\.siact\.(?!hydrocore)', 'import com.siact.hydrocore.' `
          -replace 'com\.siact\.module\.\*\.mapper', 'com.siact.hydrocore.module.*.mapper' `
          -replace 'com\.siact\.common', 'com.siact.hydrocore.common'
  if ($n -ne $c) { Set-Content $_.FullName $n -Encoding UTF8 -NoNewline }
}
```

Manually fix any external Feign imports that were incorrectly rewritten (restore jar package names).

- [ ] **Step 5: Replace application class**

Create `HydrocoreApplication.java`:

```java
package com.siact.hydrocore;

import org.mybatis.spring.annotation.MapperScan;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.openfeign.EnableFeignClients;
import org.springframework.scheduling.annotation.EnableScheduling;

@EnableScheduling
@MapperScan(basePackages = {"com.siact.hydrocore.module.*.mapper", "com.siact.hydrocore.common"})
@EnableFeignClients
@SpringBootApplication
public class HydrocoreApplication {
    public static void main(String[] args) {
        SpringApplication.run(HydrocoreApplication.class, args);
    }
}
```

Delete `KilnApplication.java`.

- [ ] **Step 6: Compile**

```powershell
mvn -q compile -DskipTests
```

Expected: exit code 0

- [ ] **Step 7: Commit (optional)**

```bash
git add -A
git commit -m "chore: rename backend to hydrocore and relocate packages"
```

---

### Task 5: Backend �?menu SQL migration script

**Files:**
- Create: `hydrocore-be/src/main/resources/db/hydrocore_menu_migrate.sql`

**Interfaces:**
- Consumes: `sys_menu` / `sys_role_menu` schema from existing DB
- Produces: SQL the operator runs when DB is available

- [ ] **Step 1: Write migration SQL**

```sql
-- HydroCore phase-1 menu migration (run manually when DB ready)
-- 1) Detach roles from kiln menus
DELETE FROM sys_role_menu WHERE menu_id IN (
  SELECT id FROM sys_menu WHERE menu_code IN (
    'yaoluyuce','wenduyuce','yaolujiance','yaolukongzhi','moxingshezhi','kongzhishezhi'
  )
  OR menu_url LIKE '%yaolu%'
  OR menu_name LIKE '%窑炉%'
);

-- 2) Soft-delete or hard-delete kiln menus (prefer hard delete for clean shell)
DELETE FROM sys_menu WHERE menu_code IN (
  'yaoluyuce','wenduyuce','yaolujiance','yaolukongzhi','moxingshezhi','kongzhishezhi'
)
OR menu_url LIKE '%yaolu%'
OR menu_name LIKE '%窑炉%';

-- 3) Insert overview + trends (adjust ids if conflict)
INSERT INTO sys_menu (parent_id, menu_code, menu_name, menu_url, type, menu_icon, model_show, sort, deleted)
VALUES
(0, 'overview', '总览', '/overview', 1, 'tabler:layout-dashboard', 1, 1, 0),
(0, 'trends', '趋势', '/trends', 1, 'tabler:chart-line', 1, 2, 0);

-- 4) Grant to admin role (role id 1 �?adjust to real admin id)
INSERT INTO sys_role_menu (role_id, menu_id)
SELECT 1, id FROM sys_menu WHERE menu_code IN ('overview','trends');
```

Align column names with actual `sys_menu` DDL in `db/schema.sql` before running (fix names if schema differs).

- [ ] **Step 2: Commit (optional)**

```bash
git add src/main/resources/db/hydrocore_menu_migrate.sql
git commit -m "chore: add hydrocore menu migration sql"
```

---

### Task 6: Frontend �?strip kiln routes, views, requests, models

**Files:**
- Delete routes: `src/routes/monitor.ts`, `forecast.ts`, `control.ts`, `config.ts`, `model.ts`, `analysis.ts` (if unused), and kiln-only device route if any
- Delete views: `src/views/monitor`, `forecast`, `control`, `model`, `config`, `energy`, `yield`, `analysis`, kiln-only `settings` pages that are not system
- Delete requests: `src/request/monitor.ts`, `forecast.ts`, `control/**`
- Delete models: `src/models/control/**`, `src/models/forecast/**`
- Delete kiln biz widgets under `src/components/biz` that only serve gas/temp forecast (keep generic `base/**`)
- Modify: `src/routes/index.ts`, `src/constants/RouteName.ts`

**Interfaces:**
- Consumes: Task 1 FE tree
- Produces: FE without kiln routes (overview/trends not yet added �?temporary redirect to system or login OK until Task 7)

- [ ] **Step 1: Slim `RouteName.ts`**

```typescript
export enum RouteName {
  HOME = 'home',
  LOGIN = 'login',

  OVERVIEW = 'overview',
  TRENDS = 'trends',
  SYSTEM = 'system',

  SYSTEM_USER = 'system.user',
  SYSTEM_MENU = 'system.menu',
  SYSTEM_ORG = 'system.org',
  SYSTEM_ROLE = 'system.role',
  SYSTEM_POINTS = 'system.points',
  SYSTEM_REALTIME = 'system.realtime',
  SYSTEM_CONFIG = 'system.config',
  SYSTEM_DEBUG = 'system.debug',
}
```

- [ ] **Step 2: Replace `src/routes/index.ts`**

```typescript
import { RouteRecordRaw } from 'vue-router';
import system from './system';
import auth from './auth';
import overview from './overview';
import trends from './trends';

export default [
  {
    path: '/',
    redirect: { name: RouteName.OVERVIEW },
    name: RouteName.HOME,
    component: () => import('@/views/route'),
    children: [auth, overview, trends, system],
  },
  auth,
] as RouteRecordRaw[];
```

(Create stub `overview.ts` / `trends.ts` in Step 3 if Task 7 not started �?minimal placeholders so imports resolve.)

- [ ] **Step 3: Delete kiln directories/files**

```powershell
Set-Location d:\project\HydroCore\hydrocore-fe
$paths = @(
  'src\routes\monitor.ts','src\routes\forecast.ts','src\routes\control.ts','src\routes\config.ts','src\routes\model.ts','src\routes\analysis.ts',
  'src\views\monitor','src\views\forecast','src\views\control','src\views\model','src\views\config','src\views\energy','src\views\yield','src\views\analysis',
  'src\request\monitor.ts','src\request\forecast.ts','src\request\control',
  'src\models\control','src\models\forecast',
  'src\api'
)
foreach ($p in $paths) { if (Test-Path $p) { Remove-Item $p -Recurse -Force } }
```

- [ ] **Step 4: Grep and remove leftover imports**

```powershell
rg -n "views/(monitor|forecast|control|model|config)|request/(monitor|forecast|control)|models/(control|forecast)|RouteName\.(MONITOR|FORECAST|CONTROL|MODEL|CONFIG)" src
```

Fix or delete each hit.

- [ ] **Step 5: Typecheck**

```powershell
pnpm exec vue-tsc --noEmit
```

Expected: exit 0 (or only errors inside files you will fix in Task 7)

- [ ] **Step 6: Commit (optional)**

```bash
git add -A
git commit -m "refactor: remove kiln frontend pages and routes"
```

---

### Task 7: Frontend �?overview shell + trends page

**Files:**
- Create: `src/routes/overview.ts`, `src/routes/trends.ts`
- Create: `src/views/overview/route.tsx`
- Create: `src/views/trends/route.tsx`
- Keep: `src/request/charts.ts`, `src/models/common/chart.query.ts`, `src/models/common/chart.dto.ts`
- Modify: `src/views/route.tsx` �?logo alt / title 窑炉 �?HydroCore / 水处�?- Modify: `src/plugins/echarts.ts` or chart theme name `kiln` �?`hydrocore` if present

**Interfaces:**
- Consumes: `charts.query(ChartQuery): Promise<ResponseModel<ChartDTO>>` where

```typescript
interface ChartQuery {
  dataCodes: Array<string>;
  startTime: string;
  endTime: string;
  ts: number;
  tsUnit: string;
  formatVal: string;
  calcType: string;
  names: Array<string>;
}
```

- Produces: routes named `overview`, `trends` matching `sys_menu.menu_code`

- [ ] **Step 1: Add route modules**

`src/routes/overview.ts`:

```typescript
import { RouteRecordRaw } from 'vue-router';

export default {
  path: '/overview',
  name: RouteName.OVERVIEW,
  meta: { auth: true },
  component: () => import('@/views/overview/route'),
} as RouteRecordRaw;
```

`src/routes/trends.ts`:

```typescript
import { RouteRecordRaw } from 'vue-router';

export default {
  path: '/trends',
  name: RouteName.TRENDS,
  meta: { auth: true },
  component: () => import('@/views/trends/route'),
} as RouteRecordRaw;
```

- [ ] **Step 2: Overview shell**

`src/views/overview/route.tsx`:

```tsx
import { NCard } from 'naive-ui';

export default defineComponent(() => {
  return () => (
    <div class="p-6">
      <h1 class="text-2xl mb-2">水处理总览</h1>
      <p class="text-muted mb-4">HydroCore 一期占位页。工艺监控与控制将在后续迭代接入�?/p>
      <NCard>暂无工艺数据接入。请从「趋势」查看孪生点位曲线，或在「系统管理」配置点位映射�?/NCard>
    </div>
  );
});
```

- [ ] **Step 3: Trends page (real API, honest empty/error)**

`src/views/trends/route.tsx` �?minimal behavior:

1. Inputs: comma-separated `dataCodes`, start/end time (dayjs), optional `names`
2. Button「查询」calls `charts.query` with:

```typescript
{
  dataCodes: codes,
  names: names.length ? names : codes,
  startTime,
  endTime,
  ts: 1,
  tsUnit: 'm',
  formatVal: 'yyyy-MM-dd HH:mm:ss',
  calcType: 'avg',
}
```

3. On success: render line chart via existing `@/components/base/charts` `Line` using `xAxisData` + series from `list`
4. On failure: show `message` / error text from response �?do not invent series
5. On empty `list`: show「无数据�?
Keep the component in one file for phase-1; reuse `charts` request module �?do not add new backend endpoints.

- [ ] **Step 4: Branding in layout**

In `src/views/route.tsx`, change logo `alt` and any visible「窑炉」copy to `HydroCore` / `水处理`.

Rename echarts theme id from `'kiln'` to `'hydrocore'` in chart init call sites and theme registration so search for `kiln` returns no FE hits (except historical docs).

- [ ] **Step 5: package.json name**

```json
"name": "hydrocore"
```

- [ ] **Step 6: Verify**

```powershell
pnpm exec vue-tsc --noEmit
pnpm run build
```

Expected: both succeed

- [ ] **Step 7: Commit (optional)**

```bash
git add -A
git commit -m "feat: add overview and trends shells for HydroCore"
```

---

### Task 8: End-to-end acceptance checklist

**Files:** none new �?verification only

- [ ] **Step 1: Backend compile**

```powershell
Set-Location d:\project\HydroCore\hydrocore-be
mvn -q compile -DskipTests
```

Expected: exit 0; no `KilnApplication`; `rg KilnApplication src` empty

- [ ] **Step 2: Frontend build**

```powershell
Set-Location d:\project\HydroCore\hydrocore-fe
pnpm run build
```

Expected: exit 0

- [ ] **Step 3: Grep kiln leftovers (allow docs/history)**

```powershell
rg -n "窑炉|KilnApplication|kiln-intelligent-control|yaolukongzhi" hydrocore-be/src hydrocore-fe/src
```

Expected: no matches in `src` (SQL migrate file may mention old codes intentionally)

- [ ] **Step 4: Remotes still absent**

```powershell
git -C hydrocore-be remote -v
git -C hydrocore-fe remote -v
```

Expected: empty

- [ ] **Step 5: Manual UI (when env available)**

1. Login �?lands on overview  
2. Sidebar: 总览、趋势、系统管�?only (after SQL migrate)  
3. Trends: real query or real error  
4. System points/realtime still open  

---

## Spec coverage (self-review)

| Spec requirement | Task |
|------------------|------|
| Remove kiln FE/BE business | 2, 3, 6 |
| Keep twin data APIs | 2�? (sec/tdengine retained), 7 (charts) |
| Overview + trends + system shells | 6, 7 |
| Rename to HydroCore | 1, 4, 7 |
| Git remove remote, no push | 1, 8 |
| Menu SQL | 5 |
| Acceptance | 8 |
| No 3D / no fake data | 7 |

## Placeholder / consistency check

- Chart query field names match existing `ChartQuery` / `charts.query`
- Route names `overview` / `trends` match SQL `menu_code`
- Java base package `com.siact.hydrocore` consistent across Application, MapperScan, moves
