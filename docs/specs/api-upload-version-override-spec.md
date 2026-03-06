# API Upload Version Override Spec

**Status:** Implemented (fork, no Swagger/i18n sync by design)
**Version:** 1.0
**Last Updated:** 2026-03-06

## 1. Background
`/api/apps/upload` 在上传 `.dmg` 等文件时，版本信息可能无法从 parser 稳定解析。为保证上传链路可控，需要允许调用方显式传入 `release_version/build_version`。

### 1.1 Why DMG Cannot Show Release/Build Version in Upload List
- Zealot 的上传元数据解析依赖 `AppInfo.parse(file)` 自动识别文件格式并提取 `release_version/build_version`。
- 当前解析链路（`app-info` gem）主要覆盖：`apk/aab/ipa/mobileprovision/provisionprofile/app.zip/dSYM.zip/exe/zip(内含exe)` 及新版本的 HarmonyOS 包格式。
- `dmg` 是磁盘镜像容器，不在现有解析分支中；当前链路未实现“挂载 dmg -> 读取内部 `.app` -> 提取 `Info.plist`”流程，因此无法自动得出版本号。
- 在 Zealot 当前使用版本 `5.3.7` 下，`/api/apps/upload` 仍以自动解析为主（无直接稳定的版本字段覆盖能力是本需求要解决的问题）。
- 这导致：
  - API 上传 `dmg`：常见结果是 `release_version/build_version = null`
  - 后台手动上传：可人工填写版本字段并展示（Web 表单路径允许填写）

## 2. Goal
为 `/api/apps/upload` 提供可选版本覆盖能力，规则为：
- 手动传值优先
- 未传或空值时由 parser 兜底
- 保持历史调用兼容（不强制新增必填）

## 3. Scope
### In Scope
- API 参数白名单增加 `release_version`、`build_version`（可选）
- Release 版本字段赋值逻辑改为仅在 `blank?` 时使用 parser

### Out of Scope
- Swagger 定义与多语言文案更新
- 自动生成 swagger 产物更新
- 数据库 schema 变更
- 生产部署编排文件标准化

## 3.1 Change Strategy (Fork Sync)
- 本需求遵循“最小化改动”原则，仅修改与功能直接相关的代码路径。
- 目标是降低与上游（official）后续同步冲突，避免无关文件噪音 diff。
- 本地测试相关变更与生产配置隔离，不作为上游同步基线。

## 4. API Contract
### Endpoint
- `POST /api/apps/upload`

### New Optional Request Fields
- `release_version: string | null`
- `build_version: string | null`

### Response Contract (relevant fields)
- `release_version: string | null`
- `build_version: string | null`

## 5. Functional Rules
### Rule R1: Manual value precedence
当调用方传入非空 `release_version/build_version`，响应中对应字段必须保持该值，不允许被 parser 覆盖。

### Rule R2: Parser fallback on blank
当字段未传或为空字符串时，对应字段允许由 parser 写入（若 parser 可解析）。

### Rule R3: Backward compatibility
不传 `release_version/build_version` 时，行为与历史版本一致：依赖原有解析逻辑，不新增必填约束。

## 6. Behavior Matrix (Acceptance)
| Case | Input `release_version` | Input `build_version` | Expected `release_version` | Expected `build_version` |
|---|---|---|---|---|
| A | `9.9.9` | `999` | `9.9.9` | `999` |
| B | `8.8.8` | missing | `8.8.8` | parser result or `null` |
| C | missing | `12345` | parser result or `null` | `12345` |
| D | missing | missing | parser result or `null` | parser result or `null` |
| E | `""` | `""` | parser result or `null` | parser result or `null` |

## 7. Implementation Mapping
- Controller:
  - `app/controllers/api/apps/upload_controller.rb`
  - `release_params` 增加 `:release_version, :build_version`
- Parser:
  - `app/models/concerns/release_parser.rb`
  - 版本字段赋值改为：
    - `self.release_version = parser.release_version if release_version.blank?`
    - `self.build_version = parser.build_version if build_version.blank?`

## 8. Verification Standard
### Minimal API verification
至少执行以下 5 个上传请求并检查响应字段：
- Case A: both provided
- Case B: release only
- Case C: build only
- Case D: none
- Case E: both empty string

### Pass Criteria
- 满足 Section 6 的行为矩阵
- 上传接口返回成功
- 不影响原有不传版本调用链路
- 对于 parser 无法解析版本的文件类型，`release_version/build_version` 为 `null` 视为符合预期

## 9. Deployment Notes
- 无 migration
- 可通过镜像替换滚动升级
- 回滚仅需回到上一镜像 tag
- 本地演练使用 `docker-compose.local-test.yml`（测试专用）。
- `docker-compose.local-test.yml` 不代表生产部署配置，不纳入“功能变更必要项”。

## 10. Risks
- 仅影响 `/api/apps/upload` 的版本字段赋值行为
- 依赖 parser 的场景中，无法解析时字段仍可能为 `null`（符合既有行为）

## Appendix A: Example Request
```bash
curl -X POST "https://<host>/api/apps/upload" \
  -F "token=<token>" \
  -F "channel_key=<channel_key>" \
  -F "file=@/path/to/file.dmg" \
  -F "release_version=9.9.9" \
  -F "build_version=999"
```

## Appendix B: Usage Examples
### B1. Provide both versions (manual override)
```bash
curl -X POST "https://<host>/api/apps/upload" \
  -F "token=<token>" \
  -F "channel_key=<channel_key>" \
  -F "file=@/path/to/file.dmg" \
  -F "release_version=9.9.9" \
  -F "build_version=999"
```

### B2. Provide release_version only
```bash
curl -X POST "https://<host>/api/apps/upload" \
  -F "token=<token>" \
  -F "channel_key=<channel_key>" \
  -F "file=@/path/to/file.dmg" \
  -F "release_version=8.8.8"
```

### B3. Provide build_version only
```bash
curl -X POST "https://<host>/api/apps/upload" \
  -F "token=<token>" \
  -F "channel_key=<channel_key>" \
  -F "file=@/path/to/file.dmg" \
  -F "build_version=12345"
```

### B4. Provide no version fields (parser fallback)
```bash
curl -X POST "https://<host>/api/apps/upload" \
  -F "token=<token>" \
  -F "channel_key=<channel_key>" \
  -F "file=@/path/to/file.dmg"
```

### B5. Provide empty strings (treated as blank)
```bash
curl -X POST "https://<host>/api/apps/upload" \
  -F "token=<token>" \
  -F "channel_key=<channel_key>" \
  -F "file=@/path/to/file.dmg" \
  -F "release_version=" \
  -F "build_version="
```
