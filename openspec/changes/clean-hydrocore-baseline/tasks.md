# Tasks

## 1. Baseline Inventory

- [x] 1.1 Scan source, templates, SQL, and docs for legacy business keywords.
- [x] 1.2 Classify matches as active source behavior, seed data, environment template, historical docs, generated artifact, icon metadata, or removable residue.
- [x] 1.3 Define retained exceptions and record them in the verification notes.

## 2. Backend Cleanup

- [x] 2.1 Clean `hydrocore-be/src/main/resources/db/schema/hydrocore_schema.sql` so the fresh baseline does not install old kiln, level, pressure, prediction, or weak demo business content.
- [x] 2.2 Review `hydrocore-be/src/main/resources/db/migration/` and mark historical migration scope.
- [x] 2.3 Replace old backend system config modules/codes and update system config API docs.
- [x] 2.4 Clean backend Nacos and `bootstrap.yml` templates so defaults stay HydroCore-neutral.
- [x] 2.5 Neutralize retained generic data API comments and test fixtures without breaking compatibility method signatures.
- [x] 2.6 Run backend compile and tests; fix cleanup-related compile failures.

## 3. Frontend Cleanup

- [x] 3.1 Rename `--kiln-*` theme variables and Tailwind references to `--app-*`.
- [x] 3.2 Clean frontend system config enum, tabs, form options, labels, and constants.
- [x] 3.3 Neutralize generic chart/helper naming for retained chart components.
- [x] 3.4 Review iconfont metadata and record it as generated/third-party metadata retained because font assets were not regenerated.
- [x] 3.5 Replace hardcoded environment examples with local placeholders/proxy settings.
- [x] 3.6 Run frontend build and fix cleanup-related syntax issues.

## 4. Documentation And Repository Policy

- [x] 4.1 Update root README and `docs/` to describe HydroCore as a water-treatment secondary-development baseline.
- [x] 4.2 Update backend docs for database initialization, Nacos/bootstrap config, verification commands, and seed account policy.
- [x] 4.3 Update frontend docs for env files, proxy settings, build verification, and extension entry points.
- [x] 4.4 Remove/ignore unused repository-local `.claude`, `.cursor`, `.agents`, `.codex`, and duplicate skills config.
- [x] 4.5 Document that Comet/OpenSpec/skills are invoked from the global environment; this repository keeps only OpenSpec/Comet work products.

## 5. Final Verification

- [x] 5.1 Re-run legacy keyword scan and confirm remaining matches are allowed exceptions.
- [x] 5.2 Run `mvn -q -DskipTests compile` in `hydrocore-be`.
- [x] 5.3 Run `mvn -q test` in `hydrocore-be`.
- [x] 5.4 Run `pnpm.cmd run build` in `hydrocore-fe`.
- [x] 5.5 Update this verification record with commands, results, retained exceptions, and follow-up recommendations.

## Verification Record

- `rg -n -i "kiln|forecast|level|pressure|GAS|temperament|predict|窑炉|预测|液位|压力" hydrocore-be\src hydrocore-fe\src hydrocore-be\docs hydrocore-fe\docs README.md docs -u`: PASS with retained exceptions listed below.
- `rg -n "CONTROL|FORECAST|SysConfigModuleEnum\.(CONTROL|FORECAST)" hydrocore-be\src\main\java\com\siact\hydrocore\module\system hydrocore-fe\src\models\system\config hydrocore-fe\src\views\system\config`: PASS, no matches.
- `rg -n -- "--kiln-|kiln" hydrocore-fe\src\styles`: PASS, no matches.
- `rg -n "forecast.test-mode|testJson/forecast|multiForecast|singleForecast|algorithmPredictionExecutor|intelligentComputingExecutor|TOTAL_GAS_VOLUME|POWER_ALARM|HEAT_ALARM|COLD_ALARM|gasManualVal|gasValueChange|wind_gas_ratio" hydrocore-be\src hydrocore-fe\src`: PASS, no matches.
- `mvn -q -DskipTests compile` in `hydrocore-be`: PASS.
- `mvn -q test` in `hydrocore-be`: PASS.
- `pnpm.cmd run build` in `hydrocore-fe`: PASS. The PowerShell alias `pnpm` is blocked by local execution policy, so `pnpm.cmd` was used.

Retained exceptions:

- Historical Comet/OpenSpec design and plan files under `docs/superpowers/` describe removed legacy business as project history.
- `hydrocore-be/src/main/resources/db/migration/hydrocore_menu_migrate.sql` keeps legacy terms for old-installation cleanup.
- `queryForecastIntervalVal(...)` remains in external compatibility interfaces with comments explaining that prediction is not a baseline capability.
- `logback-spring.xml` and `nacos/hydrocore.yml` contain logging key `level`, unrelated to legacy liquid-level business.
- `hydrocore-fe/src/assets/iconfont/iconfont.json` still contains historical/generated icon metadata. It is not edited because the binary font generation pipeline was not part of this cleanup.

Follow-up recommendation:

- Add real water-treatment process schema, dashboards, algorithms, reports, and integrations through separate Comet/OpenSpec changes after the Git remote and branch strategy are ready.
