# dotnet-skills

DSH 技能分组仓：**dotnet-skills**

- **上游**：https://github.com/dotnet/skills
- **结构**：本仓的树 = **上游最新树**（2026-09-13 起对齐），上游的目录层级原样保留。
  本地改动叠在对应文件上（改中文、平台分节、改 frontmatter 的 `name:` 等）——
  `git diff upstream/main` 就是「本机改了什么」的权威答案。
- **本文件**（`README.dsh-local.md`）是本地附加的说明，上游没有；上游的 `README.md` 原样保留。

## 本机改写过的技能（92 个）

- `analyzing-dotnet-performance`
- `android-tombstone-symbolication`
- `apple-crash-symbolication`
- `assertion-quality`
- `author-component`
- `binlog-failure-analysis`
- `binlog-generation`
- `build-parallelism`
- `build-perf-baseline`
- `build-perf-diagnostics`
- `check-bin-obj-clash`
- `clr-activation-debugging`
- `code-testing-agent`
- `code-testing-extensions`
- `collect-user-input`
- `configure-auth`
- `configuring-opentelemetry-dotnet`
- `convert-blazor-server-to-webapp`
- `convert-to-cpm`
- `coordinate-components`
- `coverage-analysis`
- `crap-score`
- `create-blazor-project`
- `csharp-scripts`
- `detect-static-dependencies`
- `directory-build-organization`
- `dotnet-aot-compat`
- `dotnet-maui-doctor`
- `dotnet-pinvoke`
- `dotnet-trace-collect`
- `dotnet-webapi`
- `dump-collect`
- `eval-performance`
- `exp-mock-usage-analysis`
- `exp-test-maintainability`
- `extension-points`
- `fetch-and-send-data`
- `filter-syntax`
- `find-untested-sources`
- `generate-testability-wrappers`
- `grade-tests`
- `including-generated-files`
- `incremental-build`
- `item-management`
- `maui-app-lifecycle`
- `maui-collectionview`
- `maui-data-binding`
- `maui-dependency-injection`
- `maui-safe-area`
- `maui-shell-navigation`
- `maui-theming`
- `microbenchmarking`
- `migrate-dotnet10-to-dotnet11`
- `migrate-dotnet8-to-dotnet9`
- `migrate-dotnet9-to-dotnet10`
- `migrate-mstest-v1v2-to-v3`
- `migrate-mstest-v3-to-v4`
- `migrate-nullable-references`
- `migrate-static-to-wrapper`
- `migrate-vstest-to-mtp`
- `migrate-xunit-to-mstest`
- `migrate-xunit-to-xunit-v3`
- `minimal-api-file-upload`
- `msbuild-antipatterns`
- `msbuild-modernization`
- `mtp-hot-reload`
- `nuget-trusted-publishing`
- `optimizing-ef-core-queries`
- `plan-ui-change`
- `platform-detection`
- `property-patterns`
- `resolve-project-references`
- `run-tests`
- `setup-local-sdk`
- `support-prerendering`
- `system-text-json-net11`
- `target-authoring`
- `technology-selection`
- `template-authoring`
- `template-comparison`
- `template-discovery`
- `template-instantiation`
- `template-smart-defaults`
- `template-validation`
- `test-analysis-extensions`
- `test-anti-patterns`
- `test-gap-analysis`
- `test-smell-detection`
- `test-tagging`
- `thread-abort-migration`
- `use-js-interop`
- `writing-mstest-tests`

由 `dsh-extensions/install-skill.sh` 软链进 `~/.dsh/skills/`。
