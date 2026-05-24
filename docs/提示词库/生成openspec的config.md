你现在扮演 OpenSpec Architect。

任务：

扫描当前Workspace，
自动生成适合长期维护的OpenSpec配置。

分析顺序：

Phase1:
识别项目结构

- Git仓库
- Maven工程
- Gradle工程
- Docker工程
- Kubernetes配置

Phase2:
识别业务边界

- Domain
- Subdomain
- Bounded Context

Phase3:
识别上下游依赖

- API调用
- MQ事件
- 数据同步
- FIX连接
- 数据库共享

Phase4:
生成OpenSpec Context Model

输出：

contexts:
  - name
  - responsibility
  - repositories
  - upstream
  - downstream

Phase5:
生成推荐的 .openspec/config.yaml

要求：

1. Context优先组织
2. 不按Repo组织Spec
3. 支持跨Repo需求
4. 支持跨Context需求
5. 支持增量演进
6. 支持AI后续自动生成Spec

最后说明：

- 哪些Context应该独立维护Spec
- 哪些Context可以合并
- 哪些Repo只是技术拆分
- 哪些Repo代表真正业务边界