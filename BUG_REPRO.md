# Bug Reproduction

## 包的性质

当前 test_model_fix 保存的是被测模型修复后的结果源码，不是初始含 Bug 源码。要复现原始缺陷，必须检出下面固定的 parent SHA；不要在当前修复结果源码上期待重新出现修复前失败。生成系统使用的可信验证补丁和完整验证日志仅在本地留存，不提交到结果分支。

## 问题现象

合规同事确认某条审计事件存在，按它的 `request_id` 查询时，`limit=1` 返回空页面，`total` 却是全部事件数；调大页容量后目标记录又出现。当前只做诊断，不改代码；生产代码、测试和配置继续保持现状。需要对照接口响应、存储查询参数和调用顺序，判断过滤与分页谁先执行，并解释 `total` 偏离的原因。

## 含 Bug 版本

- 仓库：zhanglei10281852-gif/ai-13
- 仓库地址：https://github.com/zhanglei10281852-gif/ai-13.git
- parent SHA：48e69b14b581253534c7edbfdba4d57df795343e

## 复现步骤

```bash
git clone -- https://github.com/zhanglei10281852-gif/ai-13.git bug-repro
cd bug-repro
git checkout --detach 48e69b14b581253534c7edbfdba4d57df795343e
go test ./internal/service -run ^TestAuditRequestFilterIsAppliedBeforePagination$ -count=1
```

## 双架构完整错误信息

### linux/amd64

- 容器内复现预期退出码：1
- 容器内复现实际退出码：1

stdout：

```text
$ go test ./internal/service -run ^TestAuditRequestFilterIsAppliedBeforePagination$ -count=1
--- FAIL: TestAuditRequestFilterIsAppliedBeforePagination (0.48s)
    annotation_repo_behavior_test.go:105: filtered audit page = {Items:[] Total:15}
FAIL
FAIL	github.com/zhanglei10281852-gif/ai/internal/service	0.486s
FAIL

```

stderr：

```text
(empty)
```

### linux/arm64

- 容器内复现预期退出码：1
- 容器内复现实际退出码：1

stdout：

```text
$ go test ./internal/service -run ^TestAuditRequestFilterIsAppliedBeforePagination$ -count=1
--- FAIL: TestAuditRequestFilterIsAppliedBeforePagination (1.13s)
    annotation_repo_behavior_test.go:105: filtered audit page = {Items:[] Total:15}
FAIL
FAIL	github.com/zhanglei10281852-gif/ai/internal/service	1.319s
FAIL

```

stderr：

```text
(empty)
```

## 通过条件

诊断必须定位 internal/repository/store.go 的 AuditFilter.BroadPage 和 internal/service/query.go 的 QueryService.Audit，证明 BroadPage 清除 RequestID 后数据库先对宽集合计数和分页，服务再对当前页做内存过滤；需用 limit=1 的空 items、宽集合 Total 及扩大 limit 后目标出现的结果串联证据。定向复现完成且目标仓库零改动。
