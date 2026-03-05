# API Upload Version Override Implementation Plan (Minimal)

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** 让 `/api/apps/upload` 支持通过 API 参数写入 `release_version/build_version`，并保持“不传则自动解析”的现有行为。

**Architecture:** 仅修改 API 参数白名单与 Release 解析优先级；采用“手动传值优先，字段为空时 parser 兜底”；不改 Swagger/i18n/生成产物，避免 fork 仓库产生文档噪音。

**Tech Stack:** Rails 7, RSpec。

---

### Task 1: 放开 Upload API 版本参数白名单

**Files:**
- Modify: `app/controllers/api/apps/upload_controller.rb`

**Step 1: 修改 `release_params`，仅新增两个可选字段**

```ruby
params.permit(
  :file, :release_type, :source, :branch, :git_commit,
  :ci_url, :changelog, :devices, :custom_fields,
  :release_version, :build_version
)
```

**Step 2: 不改鉴权与其他逻辑**

保持 `before_action`、`authorize`、事务逻辑不变。

**Step 3: 语法检查**

Run: `bundle exec ruby -c app/controllers/api/apps/upload_controller.rb`
Expected: `Syntax OK`.

**Step 4: Commit**

```bash
git add app/controllers/api/apps/upload_controller.rb
git commit -m "feat(api): permit release/build version in upload params"
```

---

### Task 2: 调整版本赋值优先级为“blank 才兜底”

**Files:**
- Modify: `app/models/concerns/release_parser.rb`

**Step 1: 仅改两行版本赋值**

```ruby
self.release_version = parser.release_version if release_version.blank?
self.build_version = parser.build_version if build_version.blank?
```

**Step 2: 保持异常处理与其余解析逻辑不变**

不修改 `rescue AppInfo::UnknownFormatError`、`rescue => e`、`ensure parser&.clear!`。

**Step 3: 语法检查**

Run: `bundle exec ruby -c app/models/concerns/release_parser.rb`
Expected: `Syntax OK`.

**Step 4: Commit**

```bash
git add app/models/concerns/release_parser.rb
git commit -m "fix(release): preserve manual version and fallback on blank"
```

---

### Task 3: 最小行为验证（仅功能）

**Files:**
- Optional Test: `spec/models/release_parser_spec.rb`（如仓库已有等价测试可复用）

**Step 1: 验证场景 A（全传）**

- 调用 `/api/apps/upload` 传 `release_version` + `build_version`
- 期望响应中的两个字段与传入值一致

**Step 2: 验证场景 B（只传一个）**

- 仅传 `release_version`
- 期望：`release_version` 保留传入值，`build_version` 由 parser 兜底（若可解析）

**Step 3: 验证场景 C（都不传）**

- 不传两个字段
- 期望：维持历史行为，版本由 parser 自动解析（若可解析）

**Step 4: 执行最小测试命令（仅在该 spec 已创建时）**

Run:

```bash
test -f spec/models/release_parser_spec.rb && bundle exec rspec spec/models/release_parser_spec.rb || echo "skip model spec"
```

Expected:  
- 文件存在时：测试 PASS  
- 文件不存在时：输出 `skip model spec` 并继续执行后续 API 验证

**Step 5: 执行最小 API 验证（覆盖白名单改动）**

- 使用你内部可用的上传样本文件（可解析或不可解析均可）调用 `/api/apps/upload`
- 至少验证一次传参：
  - `release_version=9.9.9`
  - `build_version=999`
- 期望响应中的 `release_version/build_version` 与传入一致

示例（按你的实际地址/token/文件路径替换）：

```bash
curl -X POST "http://<host>/api/apps/upload" \
  -F "token=<token>" \
  -F "channel_key=<channel_key>" \
  -F "file=@/path/to/app-file" \
  -F "release_version=9.9.9" \
  -F "build_version=999"
```

备注：若本地出现 `Could not find 'bundler' (2.5.10)`，使用项目既有容器环境执行测试。

---

### Task 4: 制作并发布镜像（GHCR）

**Files:**
- None

**Step 1: 准备镜像标签（示例）**

```bash
export IMAGE=ghcr.io/<github-username>/zealot
export TAG=5.3.7-version-override
export PLATFORM=linux/amd64
```

**Step 2: 登录 GHCR**

```bash
echo "$GITHUB_TOKEN" | docker login ghcr.io -u <github-username> --password-stdin
```

**Step 3: 检查 `buildx` 可用性**

```bash
docker buildx version
```

Expected: 输出版本信息。

若不可用：  
- 优先在 CI 中执行镜像构建推送；或  
- 在与生产同架构机器上使用 `docker build` + `docker push` 作为回退方案。

**Step 4: 构建并推送镜像（显式指定平台）**

```bash
docker buildx build \
  --platform ${PLATFORM} \
  -t ${IMAGE}:${TAG} \
  --push \
  .
```

**Step 5: 设置镜像可见性（一次性）**

- 在 GitHub `Packages` 将该镜像设为 `Public`（内部可直接拉取也可保持私有）。

---

### Task 5: 部署、回归与回滚（无文档变更）

**Files:**
- Modify: 你的生产 `docker-compose.yml`（部署仓库）

**Step 1: 发布部署**

- 将 `zealot.image` 改为完整镜像名（示例：`ghcr.io/<github-username>/zealot:5.3.7-version-override`）
- 执行：

```bash
docker compose pull zealot
docker compose up -d --no-deps zealot
```

**Step 2: 回归检查**

- `/api/apps/upload` 传版本时：返回手动值
- `/api/apps/upload` 不传版本时：保持原有解析行为

**Step 3: 回滚**

- 将 `zealot.image` 改回上一个稳定 tag（例如 `ghcr.io/tryzealot/zealot:5.3.7`）
- 执行：

```bash
docker compose pull zealot
docker compose up -d --no-deps zealot
```

- 无 migration，不涉及数据库回退

---

## Explicit Non-Goals (for minimal fork diff)

- 不修改 `spec/swagger_helper.rb`
- 不修改 `config/locales/zealot/api.en.yml`
- 不修改 `config/locales/zealot/api.zh-CN.yml`
- 不重新生成 `swagger/v1/swagger_en.json` 与 `swagger/v1/swagger_zh-CN.json`

---

## Verification Checklist

- `release_params` 已允许 `release_version/build_version`
- `ReleaseParser` 对版本字段采用 `blank?` 兜底策略
- 全传/部分传/不传三类场景行为符合预期
- 无 Swagger/i18n/生成产物噪音 diff
