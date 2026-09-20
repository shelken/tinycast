---
name: patch-rebase
description: 当 fork 的 Sync Upstream Release 失败、patch 无法应用到上游 tag、或需要新增/修改下游 patch 时阅读该技能
---

# 下游补丁重做

这个 fork 的发行版 = 上游代码 + `patches/*.patch`。`git apply` 要求 patch 的上下文行在目标 tag 上原样存在，所以上游一旦重写被 patch 覆盖的文件，同步链路就停在第一步

这不是偶发故障。上游拒收的改动（见文末）只能活在 fork，上游每次重写那块代码都要重做一遍

## 症状

- workflow 死在 `fetch_and_patch`，日志报 `patch does not apply`
- 失败会开一个 `[upstream-sync] <tag> failed` issue，cron 每小时追加一条评论，不会自愈
- 与 commit message、Co-authored-by 无关，fork 里没有 co-author 检查，直接看 patch

## 定位

对同一文件取三方，找出上游改了哪一行、哪个提交改的

```sh
git fetch upstream tag vX.Y.Z --no-tags
git show vX.Y.Z:<文件>                     # 上游现在是什么
git show HEAD:patches/<name>.patch         # patch 期望什么
git log --oneline -5 vX.Y.Z -- <文件>      # 哪个提交改的
```

再做行为对照：`git show HEAD:<文件>`（fork 当前出厂实现）对 `vX.Y.Z:<文件>`。**上游在同一提交里新增的行为必须继承**，只修 patch 卡住的那一层会让发行版静默丢掉上游的改进

## 重做

1. 净室 worktree 检出上游 tag，别在工作分支上改：`git worktree add $TMPDIR/rebuild vX.Y.Z`
2. 在 worktree 里手工实现 patch 的意图，上游新增的调用保留在外面，自己的逻辑挂在里层
3. 补丁自带的测试一起改，测试文件同样会随上游漂移（如 `main()` 变 `throws` 会让补丁整体失配）
4. 重新生成：`git -C $TMPDIR/rebuild diff > patches/<name>.patch`。**不要手改 `.patch` 文本**，上下文行会算错
5. 另起一个 worktree 复验多个补丁能共存：`git apply patches/*.patch`

## 验证

门禁清单以 `.github/workflows/sync-upstream-release.yml` 为准，不要在这里抄一份

本地把该 workflow 的每一步都跑一遍，特别留意 Release 构建配置，Debug 过了不代表它过

## 交付

```sh
git add patches/<name>.patch      # 只加这一个文件
git commit                        # 中文 conventional commits，写明上游哪个提交导致失配
git push origin main
gh workflow run sync-upstream-release.yml -R shelken/tinycast
```

轮询 `gh run list -R shelken/tinycast --limit 3` 到结束。成功标志：出现新的 `vX.Y.Z` release 带三个资产，对应失败 issue 自动关闭

链路在创建 tag 之前失败都没有副作用，`git revert` 即可退回旧补丁

## 新增下游补丁

同一套纪律。补丁必须带测试：上游不收的改动，测试是唯一守门人

**被拒收的改动要按其评审要求组织代码**。`01-support-soft-link.patch` 曾作为上游 PR #458 提交，maintainer 以性能退化拒收并明说只能放 fork；评审时要求符号链接逻辑只走 fallback、不污染快路径。这类约束不写进 patch，下次重做会重新踩一遍
