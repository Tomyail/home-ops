Thanos 部署记录
一、为什么选择部署 Thanos？
在现代云原生监控体系中，Prometheus 以其强大的功能和易用性占据了核心地位。然而，在特定场景下，原生的 Prometheus 部署可能会遇到一些挑战。本文档记录了笔者在实际环境中部署 Thanos 的过程，旨在解决以下痛点：

Prometheus 内存占用问题： Prometheus 将所有监控数据存储在本地磁盘，并需要在内存中维护索引等信息。随着监控时间的增长和数据量的增加，Prometheus 实例对内存的消耗会显著上升，尤其是在需要保留较长时间（如数月甚至数年）历史数据的场景下，这可能成为一个资源瓶颈。

跨地域/网络不稳定的数据抓取： 笔者最初将 Prometheus 部署在位于国内的 Homelab 环境中，通过 Tailscale VPN 抓取部署在美国 VPS 上的 Kubernetes 集群的监控数据。由于跨国网络的波动性，尤其是在夜间网络高峰期，Prometheus 抓取数据时常失败，导致监控数据不连续，并频繁触发告警。为了解决这个问题，决定将 Prometheus 实例迁移至 VPS 集群内部署，以保证抓取集群内部指标的实时性和成功率。同时，将 Prometheus 的本地数据保留时间从之前的14天大幅缩短至2天，以降低其资源消耗。

长期数据存储的成本与效率： 直接在 Prometheus 中存储长期数据不仅占用大量本地高性能磁盘资源，导致成本较高，而且在查询大量历史数据时，单个 Prometheus 实例的性能也可能受到影响。Thanos 提供了将监控数据备份到廉价且高可用的对象存储（如 AWS S3, Google Cloud Storage, Azure Blob Storage, MinIO 等）的方案，并通过其组件实现对这些长期数据的有效管理和查询，从而显著降低存储成本并提高长期数据查询的灵活性。

综上所述，通过部署 Thanos，期望实现以下目标：

高可用的实时监控： Prometheus 部署在应用集群内部，保证核心指标的实时采集。

经济高效的长期存储： 利用对象存储保存历史数据，降低成本。

全局数据视图： 通过 Thanos Query 组件，能够统一查询实时数据和历史数据。

减少网络波动影响： Prometheus Sidecar 将数据块定期上传到对象存储，相比 Prometheus 主动拉取远端 target，这种推送模式对网络波动的容忍度更高。

二、Thanos 的基本组件及其作用
Thanos 并非一个单一的应用程序，而是一个由多个可独立部署和扩展的组件构成的系统。理解每个组件的功能对于成功部署和运维 Thanos至关重要。以下是本次部署中涉及的核心组件：

Thanos Sidecar (边车)

作用： Sidecar 与每一个 Prometheus 实例一同部署（通常在同一个 Pod 或主机上）。它的主要职责有：

上传数据块： 定期将 Prometheus 生成的本地 TSDB (Time Series Database) 数据块上传到配置的对象存储（如 S3）。默认情况下，Prometheus 每2小时生成一个新的数据块。

提供实时数据查询接口： Sidecar 也实现了 Thanos Store API，允许 Thanos Query 组件直接查询 Prometheus 实例中最近的、尚未上传到对象存储的实时数据（通常是最近几个小时的数据）。

配置和元数据： Sidecar 还可以向对象存储上传 Prometheus 的外部标签 (external labels) 和其他元数据，这些信息对于全局视图和数据隔离非常重要。

关键配置： Prometheus 地址、对象存储凭证和桶信息、外部标签。

Thanos Store Gateway (存储网关)

作用： Store Gateway 使得 Thanos Query 组件能够访问和查询存储在对象存储中的历史数据块。它会定期扫描对象存储桶，发现新的数据块，并将其元数据加载到内存中。当收到查询请求时，它会从对象存储中拉取实际的监控数据。

关键配置： 对象存储凭证和桶信息、数据块同步间隔。

Thanos Query (查询器) / Querier

作用： Query 组件是用户与 Thanos 系统交互的主要入口（例如 Grafana 的数据源就指向它）。它实现了 Prometheus 的 HTTP API，可以接收 PromQL 查询请求。Query 组件会将查询请求分发给所有配置的下游 Store API 实现者，包括：

Thanos Sidecar： 用于查询各个 Prometheus 实例上的实时数据。

Thanos Store Gateway： 用于查询对象存储中的历史数据。

其他 Thanos Query 实例： 支持分层查询。

Thanos Rule/Ruler 实例： 用于查询告警规则和记录规则产生的数据。
Query 组件会聚合来自不同来源的查询结果，并进行去重（deduplication），最终返回给用户一个全局、统一的监控数据视图。

关键配置： 需要连接的 Store API 端点列表（例如，--store=dnssrv+_grpc._tcp.thanos-sidecar-headless.monitoring.svc.cluster.local，--store=dnssrv+_grpc._tcp.thanos-store.monitoring.svc.cluster.local）。

Thanos Compact (压缩器) (可选但强烈推荐)

作用： Compact 组件负责对象存储中数据块的维护工作，主要包括：

压缩 (Compaction)： 将对象存储中较小、较短时间范围的数据块合并成更大、更长时间范围的数据块。例如，将多个2小时的块合并成12小时或24小时的块。这可以显著减少对象存储中的对象数量，降低存储成本，并提高历史数据的查询性能。

降采样 (Downsampling)： 生成数据的低分辨率版本。例如，原始数据可能每15秒一个点，Compact 可以生成一个每5分钟或1小时一个点的降采样数据。查询长时间范围的数据时，使用降采样数据可以大大加快查询速度。

保留策略 (Retention)： 根据配置的保留策略，删除对象存储中过期的数据块和降采样数据。

部署模式： 通常作为单个实例或高可用对运行，因为它需要对整个对象存储桶进行操作。

关键配置： 对象存储凭证和桶信息、压缩和降采样参数、保留策略。

Thanos Ruler (规则评估器) (可选)

作用： Ruler 组件用于在 Thanos 系统中执行 Prometheus 的记录规则 (recording rules) 和告警规则 (alerting rules)。它可以基于 Thanos Query 查询到的全局数据进行规则评估，并将结果（新的时间序列或告警）写回到对象存储（通过 Sidecar 模式）或直接暴露给其他系统。

部署场景： 当你需要基于全局数据（跨多个 Prometheus 实例）进行告警或生成聚合指标时非常有用。如果你的告警和记录规则仅限于单个 Prometheus 实例的数据，那么可以直接在 Prometheus 本身进行评估。

在本次部署中，我们主要关注 Sidecar、Store Gateway、Query 和 Compact 组件的部署与配置。

三、Thanos 部署步骤

以下步骤概述了笔者在 Kubernetes 环境中，采用 GitOps 方法（具体通过 FluxCD 管理 HelmRelease 和 Kustomization 资源）部署 Thanos 并集成现有 Prometheus 的过程。详细的配置可以参考笔者的 home-ops GitHub 仓库。

1. 前提条件
- 一个正在运行的 Kubernetes 集群，并已配置 GitOps 工具（如 FluxCD）。
- 已部署的 Prometheus 实例（例如通过 kube-prometheus-stack Helm Chart）。
- 一个对象存储服务（例如 AWS S3, MinIO），并已准备好存储桶和访问凭证。
- kubectl 命令行工具配置完成，可以访问您的 Kubernetes 集群。
- ExternalSecrets Operator 已在集群中部署，用于安全地管理外部密钥。

2. 部署流程详解

**（1）对象存储密钥自动同步**
- 通过 ExternalSecret 资源，将对象存储（如 MinIO/S3）的访问密钥从外部密钥管理系统（如 Bitwarden）同步到 Kubernetes Secret。
- 参考：`kubernetes/apps/observability/thanos/app/externalsecret.yaml`

**（2）部署 Thanos 组件（Query、Store Gateway、Compactor、Ruler 等）**
- 使用 HelmRelease 资源，通过 Bitnami Helm Chart 部署 Thanos。
- 主要参数：
  - `existingObjstoreSecret` 指定上一步生成的 Secret。
  - 各组件（如 query、storegateway、compactor、ruler）可按需启用，并配置资源、持久化存储等。
- 参考：`kubernetes/apps/observability/thanos/app/helmrelease.yaml`

**（3）Kustomization 资源管理**
- 通过 Kustomization 资源，将 Thanos 相关的 ExternalSecret 和 HelmRelease 纳入 GitOps 管理。
- 参考：`kubernetes/apps/observability/thanos/app/kustomization.yaml`、`kubernetes/apps/observability/thanos/ks.yaml`
- 在 observability 目录的 kustomization.yaml 中引入 thanos/ks.yaml，确保 Flux 能自动部署和管理 Thanos。

**（4）Prometheus 集成 Thanos Sidecar**
- 在 kube-prometheus-stack 的 HelmRelease 配置中：
  - 启用 `thanosService` 和 `thanosServiceMonitor`，暴露 Sidecar 的 gRPC 服务和监控指标。
  - 在 `prometheusSpec` 下配置 thanos 字段，指定 Sidecar 镜像和对象存储 Secret。
  - 增加 Secret 自动 reload 注解，确保对象存储 Secret 更新时 Prometheus Pod 能自动 reload。
- 参考：`kubernetes/apps/observability/kube-prometheus-stack/app/helmrelease.yaml`

**（5）Grafana 集成 Thanos 数据源和仪表盘**
- 在 Grafana 的 HelmRelease 配置中：
  - 将默认 Prometheus 数据源地址切换为 Thanos Query 前端服务（如 `thanos-query-frontend.observability.svc.cluster.local:9090`）。
  - 设置 `prometheusType: Thanos`，确保兼容性。
  - 自动导入 Thanos 官方仪表盘（overview、query、store、sidecar、compact 等），便于全局监控和可视化。
- 参考：`kubernetes/apps/observability/grafana/app/helmrelease.yaml`

3. 相关配置 commit 参考
- [Thanos 组件 GitOps 部署](https://github.com/Tomyail/home-ops/commit/8ee2436daceace4fbe7e7b0b7529a8d747f6917d)
- [Prometheus 集成 Thanos Sidecar](https://github.com/Tomyail/home-ops/commit/bd1d8b7014ea6496a176c55ae7a7efaa89f9c8b2)
- [Grafana 集成 Thanos 数据源和仪表盘](https://github.com/Tomyail/home-ops/commit/a35ca0a19264675af1d9f05060a5b52418f2c793)

通过上述步骤，即可实现 Thanos 在 Kubernetes 集群中的自动化、模块化部署，并与现有 Prometheus 和可视化系统（Grafana）无缝集成，实现监控数据的高可用、低成本、全局统一管理。

四、效果验证
部署完成后，可以通过以下方式验证 Thanos 系统是否按预期工作：

观察对象存储 (S3) 目录：

登录到您的对象存储控制台或使用 S3 客户端工具。

检查 Thanos 配置的桶中是否开始出现数据块。

Thanos Sidecar 上传的数据块通常以 Prometheus 的外部标签（如果配置了）和时间戳命名，形成类似 ulid/<block_id>/ 的目录结构，其中包含 meta.json, index, chunks/ 等文件。

您应该能看到新的数据块（通常每2小时一个）被持续上传。

访问 Thanos 组件的 Web UI：

Thanos Query UI: 访问 Thanos Query 的 HTTP 端口 (例如 http://<thanos-query-ip>:10902 或通过 Kubernetes Service 暴露的端口)。

在 "Stores" 页面，您应该能看到所有已连接的 Store API 端点，包括您的 Prometheus Sidecar(s) 和 Store Gateway(s)，以及它们的状态（UP/DOWN）和报告的时间范围。

尝试执行一些 PromQL 查询，确认能够查询到实时数据（来自 Sidecar）和历史数据（来自 Store Gateway）。

Thanos Store Gateway UI: 访问 Store Gateway 的 HTTP 端口。

"Blocks" 页面会显示 Store Gateway 从对象存储中加载的数据块列表，包括每个块的 ID、时间范围 (Min Time, Max Time)、数据量等信息。

您提供的截图 Pasted image 20250518135431.png 和 Pasted image 20250518135446.png 就是 Store Gateway UI 的典型视图。可以看到，新产生的数据块（例如当天的）是2小时一个分块，而经过 Thanos Compact 处理的较旧数据块（例如前一天的）可能已经被合并成了更大的时间分块（如8小时）。这验证了 Sidecar 的上传和 Compact 的压缩功能。
[图片：Store Gateway UI 显示数据块]
[图片：Store Gateway UI 显示数据块详情]

Thanos Compact UI (如果部署了): 访问 Compact 组件的 HTTP 端口。

可以查看 Compact 的状态、执行的压缩轮次、降采样进度以及应用的保留策略等信息。

Grafana 图表验证：

在 Grafana 中打开或创建仪表盘，选择配置了 Thanos Query 的数据源。

尝试查询覆盖了 Prometheus 本地保留期（例如2天）之前的数据。如果能够成功加载历史数据，说明 Store Gateway 正在正常工作。

对比查询实时数据和历史数据的连续性，确保数据没有丢失。

检查新添加的 Thanos 相关仪表盘是否正常显示数据。

五、一个注意点：历史数据的迁移
正如您在提纲中指出的，Thanos Sidecar 不会自动上传 Prometheus 在 Sidecar 启动之前就已经存在的历史数据块。 Sidecar 只会监控 Prometheus 新生成的 TSDB 块，并在它们生成后将其上传到对象存储。

如果您希望将 Prometheus 中已有的、早于 Sidecar 部署时间的本地历史数据也迁移到 Thanos 的对象存储中，您需要手动进行操作。可以使用 thanos tools bucket upload 命令：

定位 Prometheus 数据目录： 找到 Prometheus 存储其 TSDB 数据块的目录 (通常是 data/ 或 /prometheus/ 下的 wal/ 目录之外的那些以 ULID 命名的子目录)。

停止 Prometheus (推荐但非必需)： 为了确保数据一致性，最好在上传前临时停止 Prometheus 服务，或者至少确保它不会再写入到您要上传的旧数据块中。

执行上传命令：

thanos tools bucket upload \
    --objstore.config-file /path/to/your/objstore.yml \
    --source /path/to/prometheus/data/ \
    [--external.prefix <external_prefix>] \
    [--labels 'cluster="my-cluster"' --labels 'replica="prometheus-0"']

--objstore.config-file: 指向与您的 Thanos 组件使用的相同的对象存储配置文件。

--source: 指向 Prometheus 的数据目录 (包含所有数据块子目录的父目录)。

--external.prefix: (可选) 如果您的数据块在对象存储中需要一个特定的前缀目录。

--labels: (可选但重要) 为这些手动上传的数据块添加外部标签，以便 Thanos Query 能够正确地识别和去重它们。这些标签应该与您 Prometheus Sidecar 配置中的外部标签一致。

上传完成后，Thanos Store Gateway 会在下一次同步时发现这些新的历史数据块，并使其可供查询。

通过以上步骤和验证，您应该能够成功部署一个稳定、高效的 Thanos 监控系统，实现监控数据的长期存储和全局查询。
