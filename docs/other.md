# 源码分析

## envd
envd 应用分析
是什么
envd (Environment Daemon) 是一个运行在 E2B Sandbox（沙箱虚拟机）内部的守护进程。它是 SDK 与沙箱之间的桥梁，让外部客户端可以通过 RPC/HTTP 调用来操控沙箱内的文件系统和进程。

架构概览
外部 SDK / Client
│
▼ HTTP (port 49983)
┌─────────────────────────────────────────┐
│               envd daemon               │
│  ┌──────────┐  ┌──────────┐  ┌───────┐ │
│  │ REST API │  │FS gRPC   │  │Process│ │
│  │(OpenAPI) │  │(ConnectRPC)│ │ gRPC  │ │
│  └──────────┘  └──────────┘  └───────┘ │
│       │               │           │    │
│  CORS + Auth Middleware (X-Access-Token / Signature) │
└─────────────────────────────────────────┘
│
▼ 操作沙箱内部
文件系统 / 子进程 / 系统时钟

接口清单
1. REST API (OpenAPI, /spec/envd.yaml)
   Method	Path	说明
   GET	/health	健康检查，返回 204
   GET	/metrics	获取 CPU/内存指标
   POST	/init	初始化：设置 env vars、access token，同步宿主机时钟
   GET	/envs	获取当前所有环境变量
   GET	/files	下载文件（支持 signature 鉴权）
   POST	/files	上传文件，自动创建父目录（支持 signature 鉴权）
2. gRPC (ConnectRPC) - Filesystem Service (/spec/filesystem/filesystem.proto)
   RPC	说明
   Stat	获取文件/目录元信息
   MakeDir	创建目录
   Move	移动/重命名文件
   ListDir	列出目录内容（支持深度）
   Remove	删除文件/目录
   WatchDir	流式监听目录变更事件 (CREATE/WRITE/REMOVE/RENAME/CHMOD)
   CreateWatcher	创建非流式 watcher
   GetWatcherEvents	轮询 watcher 事件
   RemoveWatcher	删除 watcher
3. gRPC (ConnectRPC) - Process Service (/spec/process/process.proto)
   RPC	说明
   Start	启动进程，流式返回 start/data/end 事件
   Connect	连接已有进程，流式接收输出
   List	列出所有运行中的进程
   Update	更新 PTY 窗口大小
   SendInput	向进程发送 stdin/pty 输入
   StreamInput	流式发送输入（保证顺序）
   SendSignal	向进程发送 SIGTERM/SIGKILL

核心流程
启动流程
1. main() 解析 flag (port、debug、startCmd 等)
2. 创建 chi router，挂载 Filesystem gRPC handler 和 Process gRPC handler
3. 创建 REST API handler（api.New），维护全局 envVars 和 accessToken
4. 套上 CORS middleware + Auth middleware 启动 HTTP server
5. 如果传了 --cmd flag，自动以 root 用户启动初始进程
   进程执行流程 (Process.Start)
   SDK 调用 Start(cmd, args, envs, cwd, pty?)
   → 等待宿主机时钟同步 (WaitForSync)
   → 鉴权 (GetAuthUser)
   → 创建 Handler: exec.Cmd + UID/GID 设置 + 环境变量合并
   → 若有 PTY: pty.StartWithSize → 读 pty 输出
   → 若无 PTY: 建立 stdout/stderr/stdin pipe
   → 启动进程，调整 OOM Score
   → 流式推送 StartEvent(pid) → DataEvent(stdout/stderr/pty) → EndEvent(exitCode)
   鉴权机制
   ● Access Token: 通过 POST /init 设置，之后所有 gRPC 请求必须带 X-Access-Token header
   ● Signature 鉴权: 专门用于文件上传/下载（GET/POST /files），通过 path:operation:username:token[:expiration] 的 SHA256 hash 生成签名，支持过期时间，无需在 header 带 token
   时钟同步
   ● POST /init 触发后台异步调用 /bin/bash -c "date -s @$(/usr/sbin/phc_ctl /dev/ptp0 get ...)" 从宿主机 PTP 硬件时钟同步时间
   ● Process.Start 在启动前会等待时钟同步完成（使用读写锁），确保进程内时间准确

## template-manager
Template-Manager 核心接口与逻辑分析
一、是什么
Template-Manager 是内嵌在 orchestrator 二进制中的一个子服务，通过环境变量 ORCHESTRATOR_SERVICES=template-manager 激活，部署在 Nomad 的 build 节点池，监听端口 5009。
核心职责：将用户提供的 Docker 镜像，构建成 Firecracker 虚拟机快照（rootfs + memfile + snapfile），上传到对象存储（Aliyun OSS），供后续沙箱快速启动使用。

二、gRPC 接口（template-manager.proto）
service TemplateService
RPC	说明
TemplateCreate	触发异步构建，立即返回 Empty，构建在后台执行
TemplateBuildStatus	轮询构建状态：Building / Failed / Completed
TemplateBuildDelete	取消正在进行的构建 + 删除存储中的构建产物
HealthStatus (deprecated)	返回 Healthy / Draining，已被 InfoService 替代
TemplateConfig 关键字段：
● templateID / buildID
● memoryMB / vCpuCount / diskSizeMB
● kernelVersion / firecrackerVersion
● startCommand — 构建时在沙箱内执行的启动命令
● readyCommand — 判断沙箱是否就绪的命令
● hugePages — 是否启用大页内存

三、核心构建流程
TemplateCreate 收到请求后立即返回，构建在 goroutine 中异步执行：
TemplateCreate (RPC 返回)
│
└─ [goroutine] builder.Build()
│
▼
Step 1: 获取 envd 版本号
│
▼
Step 2: 构建 ext4 根文件系统 (rootfs.go)
├─ 从 ArtifactsRegistry 拉取 Docker 镜像（OCI）
├─ 向镜像注入额外 OCI Layer：
│    ├─ /etc/hostname、/etc/hosts、/etc/resolv.conf
│    ├─ /usr/bin/envd（envd 守护进程二进制）
│    ├─ /etc/systemd/system/envd.service（开机自启）
│    ├─ /usr/bin/busybox + /usr/bin/init（第一阶段 init）
│    ├─ /etc/inittab + /etc/init.d/rcS（BusyBox init 配置）
│    ├─ /usr/local/bin/provision.sh（预配置脚本）
│    └─ chrony.service 符号链接（时钟同步）
├─ OCI 镜像 → ext4 文件系统
└─ 按目标 DiskSizeMB 调整大小
│
▼
Step 3: 创建空 memfile（内存文件，大小 = MemoryMB）
│
▼
Step 4: 第一阶段 VM 启动 —— BusyBox init（provision）
├─ 启动 Firecracker VM，init = BusyBox
├─ 执行 provision.sh：
│    ├─ apt 安装：systemd、openssh、chrony、linuxptp、e2fsprogs
│    ├─ 配置 shell profile，清除 root 密码
│    ├─ 配置 chrony（PTP 硬件时钟同步）
│    ├─ 配置 SSH（允许 root 空密码登录）
│    ├─ 创建 128MB swap 文件
│    ├─ ln -sf systemd /usr/sbin/init（接管 init）
│    └─ 写出退出码 → /provision.result
├─ 等待 VM 退出
└─ 校验 /provision.result == "0"
│
▼
Step 5: ext4 完整性检查 + 根据 provision 后大小变化扩容磁盘
│
▼
Step 6: 第二阶段 VM 启动 —— systemd（正式启动）
├─ 启动 Firecracker VM，init = systemd
├─ WaitForEnvd()：轮询直到 envd 守护进程响应（最长 60s）
└─ 注册到 sandboxes map + proxy（供后续 envd 调用路由）
│
▼
Step 7: 执行 configure.sh（通过 envd Process gRPC 调用）
├─ swapon 启用 swap
├─ 创建 user 用户，赋予 sudo NOPASSWD
├─ chown /home/user，chmod /usr/local，mkdir /code
└─ （以 root 身份在沙箱内执行）
│
▼
Step 8: 并发执行 startCommand + readyCommand
├─ startCmd（用户自定义）：在 /home/user 以 root 执行，携带镜像 ENV
├─ readyCmd：轮询直到退出码为 0（默认 sleep 20s，超时 5min）
└─ 通过 startCmdConfirm channel 协调：确保 start 先启动，ready 再检查
│
▼
Step 9: Pause VM → 生成快照
├─ snapfile（VM CPU/寄存器状态）
├─ memfile diff（内存差分）
└─ rootfs diff（磁盘差分）
│
▼
Step 10: 上传快照到 OSS
└─ 上传 snapfile + memfile_diff + rootfs_diff
│
▼
buildCache.SetSucceeded(buildID, {rootfsSizeMB, envdVersion})

四、构建状态管理（BuildCache）
使用内存 TTL 缓存（10 分钟过期）跟踪每个 buildID 的状态：
Create → Building
├── SetSucceeded() → Completed（附带 rootfsSizeMB、envdVersion）
└── SetFailed()   → Failed
● TemplateBuildStatus 直接查询缓存返回状态
● TemplateBuildDelete 如果 build 还在缓存中（仍在构建），先 Cancel() context 终止 goroutine，再删除存储产物

五、关键设计点
设计	说明
两阶段启动	Phase 1 用 BusyBox 安装 systemd，Phase 2 用 systemd 正式启动，确保完整服务环境
异步构建	TemplateCreate 立即返回，客户端轮询 TemplateBuildStatus
通过 envd 执行命令	configure.sh / startCmd / readyCmd 都通过本地 sandbox proxy 路由到 envd 的 Process gRPC 执行，template-manager 本身也是 envd 的客户端
优雅排空	关闭时先标记 Draining（等 5s Consul 摘流），再 wg.Wait() 等所有构建完成，再等 15s 后退出
磁盘精确控制	provision 前后各做一次 ext4 扩容 + 完整性检查（e2fsck），确保磁盘大小精确匹配用户配置的 DiskSizeMB

## orchestrator
现在有了完整的信息，给出系统性的中文分析：

Orchestrator 服务核心分析
一、整体定位
Orchestrator 是运行在每个计算节点上的核心服务（端口 5008），负责沙箱（Sandbox）的全生命周期管理：创建、更新、暂停（快照）、删除。它是 API 层和 Firecracker 虚拟机之间的直接调度层。

二、gRPC 接口（orchestrator.proto）
service SandboxService
RPC	说明
Create	从模板快照恢复启动一个沙箱 VM
Update	更新沙箱的过期时间（延长生命周期）
List	列出当前节点所有运行中的沙箱
Delete	停止并销毁一个沙箱
Pause	对沙箱打快照后停止（用于用户侧 pause 操作）
ListCachedBuilds	列出当前节点内存中已缓存的模板构建文件
SandboxConfig 关键字段：
字段	说明
templateId / buildId	指定使用哪个模板快照
sandboxId / executionId	沙箱唯一标识（executionId 用于连接池隔离）
vcpu / ramMb / totalDiskSizeMb	计算资源规格
envVars	注入到沙箱的环境变量
envdAccessToken	envd 访问令牌
autoPause	是否自动暂停
maxSandboxLength	最大生命周期（小时）
snapshot	是否基于快照恢复
hugePages	是否启用大页内存

三、核心流程
3.1 Create（沙箱创建 = 从快照恢复）
Create(RPC)
│
▼
ResumeSandbox()
├─ templateCache.GetTemplate(templateId, buildId, kernel, fc)
│       └─ 若缓存未命中 → 异步从 OSS 下载 snapfile/memfile/rootfs 到本地
│
├─ [异步] 从 networkPool 获取网络 Slot（IP + tap 设备）
│
├─ 创建 rootfs Overlay（NBD + COW）
│       └─ 只读 base rootfs + 可写 overlay，保存沙箱私有写操作
│
├─ 启动 UFFD（userfaultfd）内存服务
│       └─ 按需从 memfile 加载内存页（懒加载），记录脏页
│
├─ fc.NewProcess() → fc.Resume()
│       ├─ 通过 Firecracker API 从 snapfile 恢复 VM 状态
│       ├─ 注入 MMDS 元数据（sandboxId、templateId、teamId、traceId）
│       └─ 等待 UFFD Ready 信号后才启动 VM
│
├─ WaitForEnvd()
│       └─ 循环 POST /init 到 envd（携带 envVars + accessToken）
│          直到返回 204（envd 完全就绪），默认超时 10s
│
├─ 注册到 sandboxes map + SandboxProxy 路由表
│
└─ [goroutine] 监听 VM 退出 → 执行 cleanup
├─ 停止 FC 进程
├─ 停止 UFFD
├─ 释放 network Slot
└─ 从 sandboxes map 移除
3.2 Pause（快照 = 用户暂停沙箱）
Pause(RPC)
│
├─ 从 sandboxes map 移除（禁止新连接路由进来）
│
├─ sbx.Pause()
│       ├─ 停止健康检查
│       ├─ FC API: Pause VM（暂停 CPU 执行）
│       ├─ 禁用 UFFD（停止内存懒加载）
│       ├─ FC API: CreateSnapshot
│       │       ├─ snapfile → 写入 tmpfs（速度快）
│       │       └─ memfile → 写入 tmpfs
│       │
│       ├─ pauseProcessMemory（后处理内存）
│       │       ├─ 从 UFFD 脏页集合提取差分
│       │       ├─ 与原始 memfile header Mapping 合并
│       │       └─ 生成 memfileDiff + memfileDiffHeader
│       │
│       └─ pauseProcessRootfs（后处理磁盘）
│               ├─ 从 NBD Overlay 提取脏块
│               ├─ 与原始 rootfs header Mapping 合并
│               └─ 生成 rootfsDiff + rootfsDiffHeader
│
├─ templateCache.AddSnapshot()
│       └─ 将新快照加入本地缓存（供后续 Resume 直接使用，无需重新下载）
│
└─ [异步 goroutine] 上传 snapfile + memfile_diff + rootfs_diff 到 OSS
└─ [异步] sbx.Stop() 释放资源
3.3 Delete（删除沙箱）
Delete(RPC)
├─ 立即从 sandboxes map 移除
├─ Healthcheck(alwaysReport=true) — 记录最终健康状态
└─ sbx.Stop() → cleanup chain
├─ Stop health checks
├─ FC 进程停止（kill firecracker 进程）
├─ UFFD 停止
├─ rootfs overlay 关闭
└─ network Slot 归还到 Pool

四、关键子系统
4.1 TemplateCache（模板缓存）
● TTL = 25 小时（应大于沙箱最长生命周期）
● 第一次 GetTemplate 时触发异步预下载：并发拉取 snapfile / memfile / rootfs 到本地
● Pause 后生成的新快照直接 AddSnapshot 进缓存，下次 Resume 直接命中
● 使用 SetOnce 保证并发安全（多个沙箱同时请求同一 buildId 时只下载一次）
4.2 SandboxProxy（反向代理，端口 5007）
● 作为 HTTP 反向代理，将外部请求路由到沙箱内的 envd
● 路由 key = Host header 中的 {sandboxId}.{port} 格式
● 通过 sandboxes map 查找沙箱 IP，转发到 {sandboxIP}:{port}
● 用 executionId 作为连接池 key（防止 pause/resume 时网络 Slot 复用导致的连接串扰）
● 空闲超时 620s（大于 GCP LB 的 600s）
4.3 Network Pool
● 预分配网络 Slot（tap 设备 + IP 地址）
● 分两类：新建 Slot 池（NewSlotsPoolSize）和复用 Slot 池（ReusedSlotsPoolSize）
● 沙箱启动时异步获取，销毁时归还
4.4 健康检查（Checks）
● 每 20 秒调用 envd /health 接口（超时 100ms）
● 只在状态发生变化时上报日志（healthy → unhealthy 或反向），避免日志刷屏
● Delete/Pause 前强制执行一次并上报最终状态

五、架构全景
API 服务
│ gRPC (SandboxService)
▼
Orchestrator（每个计算节点）
├─ SandboxService gRPC Server (port 5008)
│       Create/Update/List/Delete/Pause
│
├─ SandboxProxy HTTP Reverse Proxy (port 5007)
│       {sandboxId}.{port} → envd at {VM IP}:{port}
│
├─ TemplateCache（本地磁盘 + 内存，TTL 25h）
│       snapfile / memfile / rootfs diff 文件
│
├─ Network Pool（预分配 tap + IP）
│
├─ NBD Device Pool（Network Block Device，用于 rootfs COW）
│
└─ Sandboxes Map（运行中沙箱的内存注册表）
每个 Sandbox 包含：
├─ FC Process（Firecracker VM 进程）
├─ UFFD（内存按需分页）
├─ rootfs Overlay（COW 磁盘）
├─ Network Slot（IP + tap）
└─ Checks（健康检查协程）

## client-proxy
client-proxy 分析
整体定位
client-proxy 是整个 E2B 集群的边缘节点（Edge Node），是外部流量进入集群的唯一入口，承担两大职责：
1. sandbox 流量代理：将用户对 {sandboxId}.{port} 的 HTTP 请求转发到正确的 orchestrator 节点
2. 集群控制平面（Edge API）：对外提供集群管理 REST API 和 gRPC 直通代理

端口与服务
端口	环境变量	默认值	用途
PROXY_PORT	3002	sandbox 流量反向代理（HTTP）
EDGE_PORT	3001	Edge REST API + gRPC pass-through（cmux 复用）
两个服务在不同端口独立运行，共享同一进程。

三大核心模块
1. Sandbox 流量代理（internal/proxy/proxy.go）
   监听 PROXY_PORT，核心逻辑是从 HTTP 请求的 Host 头中解析 {sandboxId}.{port}，然后解析出目标 orchestrator 节点 IP，将请求转发到 {nodeIP}:5007（orchestrator 内部的 SandboxProxy）。
   节点解析有两种模式，可组合使用：
   ● Catalog Resolution（推荐）：从 Redis/内存 catalog 中查询 sandboxId → orchestratorId → orchestratorIP
   ● DNS Resolution（降级）：向 Consul DNS（api.service.consul:5353）查询 {sandboxId} 的 A 记录，返回 127.0.0.1 代表沙箱不存在
   Catalog 解析失败后自动 fallback 到 DNS，两者都失败则返回 404。
   连接池使用固定 key "client-proxy"（不像 orchestrator proxy 那样按 sandboxId 分池），idle timeout 610s（略大于 GCP LB 的 600s 限制）。
2. Edge REST API（internal/edge/）
   使用 Gin + OpenAPI 3 验证，通过 EDGE_SECRET 静态 Token 鉴权（ApiKeyAuth）。暴露以下接口：
   接口	说明
   GET /health	LB 健康检查（Healthy/Draining 均返回 200）
   GET /health/traffic	sandbox 流量健康检查（仅 Healthy 返回 200）
   GET /health/machine	机器实例健康检查（Terminating 返回 503，供 autoscaling 使用）
   GET /v1/info	本节点信息
   GET /v1/service-discovery/nodes	列出集群所有节点（orchestrators + edges）
   GET /v1/service-discovery/orchestrators	列出 orchestrator 节点及其资源指标
   POST /v1/service-discovery/nodes/{id}/drain	触发指定节点的 Draining 状态（可跨节点转发）
   DELETE /v1/service-discovery/nodes/{id}	Kill 指定节点
   POST /v1/sandbox/catalog	注册 sandbox 到 catalog（由 orchestrator 调用）
   DELETE /v1/sandbox/catalog/{sandboxId}	从 catalog 删除 sandbox
   GET /v1/template/build/{buildId}/logs	查询模板构建日志
3. gRPC Pass-Through 代理（internal/edge-pass-through/proxy.go）
   与 REST API 共用 EDGE_PORT，通过 cmux 按 Content-Type 区分 HTTP/2 gRPC 流量。
   本质是一个透明 gRPC 代理：从请求 metadata 中读取 EdgeRpcServiceInstanceIDHeader，在 OrchestratorsPool 中找到对应节点，直接复用该节点的 gRPC *ClientConn 进行双向流量透传（server→client 和 client→server 两个 goroutine 并行）。
   这样上层（如 api 服务）可以通过任意一个 edge 节点直接向指定 orchestrator 发送 SandboxService、TemplateService 等 gRPC 调用，无需知道 orchestrator 的实际地址。

OrchestratorsPool（internal/edge/pool/）
每个 OrchestratorNode 对应一个 orchestrator 实例，维护：
● gRPC 客户端：Sandbox、Template、Info 三个 stub
● 实时指标（atomic）：VCpuUsed、MemoryUsedInMB、DiskUsedInMB、SandboxesRunning
● 节点状态：Healthy / Draining / Unhealthy
后台每 10s 调用 Info.ServiceInfo() 同步上述信息（最多 3 次重试）。
Pool 本身每 10s 通过 ServiceDiscovery 刷新节点列表：新节点建立 gRPC 连接 + 初始同步；消失的节点关闭连接并移出 map。

SandboxesCatalog（internal/edge/sandboxes/）
分两种实现：
实现	场景	说明
MemorySandboxCatalog	单实例/本地开发	纯内存 map
RedisSandboxCatalog	多实例生产	Redis 持久化 + 本地 5s TTL 缓存
Redis catalog 存储 key 为 sandbox.dns.{sandboxId}，value 为 SandboxInfo（orchestratorId、executionId、startTime、maxLength）。写入使用 Redsync 分布式锁（全局唯一锁 sandboxes-catalog-cluster-lock）保证多实例一致性。删除时校验 executionId 防止误删（pause/resume 场景下 sandbox 可能已被新 execution 接管）。

Service Discovery（internal/service-discovery/）
支持三种节点发现方式，通过环境变量 SERVICE_DISCOVERY_{ORCHESTRATOR|EDGE}_PROVIDER 配置：
Provider	环境变量 key	使用场景
DNS	DNS_QUERY + DNS_RESOLVER_ADDRESS	Consul DNS（生产）
EC2-INSTANCES	EC2_REGION + EC2_TAGS	AWS EC2 按标签发现
STATIC	STATIC（IP 列表）	本地/静态环境

优雅关闭流程
SIGTERM/SIGINT
→ SetDraining()                  # 标记为 Draining，流量健康检查返回 503
→ sleep(15s)                     # 等待 LB 感知，停止发送新 sandbox 请求
→ trafficProxy.Shutdown()        # 等待所有现有 sandbox 连接自然关闭
→ SetUnhealthy()                 # 标记为 Unhealthy，Edge API 健康检查返回 503
→ sleep(15s)                     # 等待 LB 感知，停止发送新 API 请求
→ grpcSrv.GracefulStop()         # 等待所有 gRPC 流量（sandbox 创建/pause 等）完成
→ restSrv.Shutdown()             # 关闭 REST API
→ SetTerminating()               # 机器健康检查返回 503，通知 autoscaling 可安全回收
→ muxServer.Close()              # 关闭监听器

架构位置总结
外部用户/SDK
↓ (HTTP, Host: {sbxId}.{port})
client-proxy :3002 (traffic proxy)
→ catalog/DNS 解析 sandboxId → orchestratorIP
→ 转发到 orchestrator :5007 (SandboxProxy)
→ 再转发到 VM IP :port (envd)

client-proxy :3001 (edge API + gRPC pass-through)
REST API: 集群管理、sandbox catalog、健康检查
gRPC透传: api 服务 → 指定 orchestrator (SandboxService/TemplateService)

## api
API Service 分析
整体定位
api 服务 (内部名 orchestration-api) 是整个 E2B 平台的业务逻辑控制平面，是用户/SDK 调用的唯一对外 REST API 入口，负责：鉴权、模板管理、sandbox 生命周期编排、以及自带 DNS 服务器。

HTTP 服务结构（端口 80）
使用 Gin + OpenAPI 3 双重验证，ReadTimeout/WriteTimeout 75s，IdleTimeout 620s。
鉴权体系（三种方式并存）
方式	Header	说明
API Key	X-API-Key: e2b_...	v2 接口主要方式，从 DB 查 Team/Tier 并缓存
Access Token	Authorization: Bearer ...	v1 接口，从 DB 查 UserID
Supabase JWT	X-Supabase-Token + X-Supabase-Team	Supabase 集成，HMAC-HS256 验证，支持多 secret 轮换
认证结果缓存在 TeamAuthCache（内存），避免每次请求都查 DB。
v1 / v2 路由分层
● v2 路由 (/v2/templates/...)：直接注册，X-API-Key 鉴权，不经过 OpenAPI validator
● v1 路由 (OpenAPI spec 中定义)：经过 OpenAPI schema 验证；部分路径有 v1BridgeAuth 拦截，透明转发到 v2 handler（兼容老版 SDK）
● 跳过 tracing 的高频路径：/health, /sandboxes/:id/refreshes, /templates/.../logs, /templates/.../status

核心 REST API 接口
Sandbox 生命周期
方法	路径	说明
POST	/sandboxes	创建 sandbox
GET	/sandboxes	列出运行中的 sandboxes
DELETE	/sandboxes/:sandboxID	删除/Kill sandbox
POST	/sandboxes/:sandboxID/pause	暂停 sandbox（触发 snapshot）
POST	/sandboxes/:sandboxID/resume	从 snapshot 恢复 sandbox
POST	/sandboxes/:sandboxID/refreshes	续期 sandbox（延长超时）
GET	/sandboxes/:sandboxID/logs	获取 sandbox 日志（from Loki）
GET	/sandboxes/:sandboxID/metrics	获取 sandbox 指标
模板管理
方法	路径	说明
POST	/templates	创建新模板（触发构建）
POST	/templates/:templateID	重新构建已有模板
GET	/templates	列出模板
DELETE	/templates/:templateID	删除模板
POST	/v2/templates	v2 创建模板（API Key）
POST	/v2/templates/:id/builds/:buildID	提交 SDK 构建步骤
GET	/v2/templates/:id/builds/:buildID/status	查询构建状态
GET	/v2/templates/:id/files/:hash	获取构建产物预签名 URL
GET	/templates/:id/builds/:buildID/logs	构建日志（SSE 流）

核心子系统
1. Orchestrator 客户端（internal/orchestrator/）
   API 服务侧的核心调度逻辑：
   Node 管理
   ● 通过 Nomad API (Status == "ready" and NodePool == "default") 发现 orchestrator 节点
   ● 每 20s keepInSync：新节点 → 建立 gRPC 连接；消失节点 → 关闭并移除
   ● 每次同步：调用 ServiceInfo 更新节点状态、调用 List 获取存活 sandbox、调用 ListCachedBuilds 刷新 build 缓存
   Node 选择（创建 sandbox 时）
   ● Resume 优先同节点：snapshot 在哪个节点，就优先在那个节点 resume（本地 template cache 命中率更高）
   ● 最小负载节点：按 CPUUsage + sbxsInProgress.CPUs 排序；每个节点最多同时启动 3 个 sandbox（maxStartingInstancesPerNode）
   ● 重试机制：最多 3 次，失败节点加入 exclude 列表
   InstanceCache 钩子（insert / delete 时触发）
   ● Insert 钩子：累加节点 CPUUsage/RamUsage；写 DNS 记录（sandboxId → nodeIP）；触发 start 分析事件
   ● Delete 钩子：
   ○ AutoPause=true：调用 PauseInstance → 写 DB snapshot → 调 orchestrator gRPC Pause → 上传到 OSS
   ○ AutoPause=false：调用 orchestrator gRPC Delete
   ○ 减少节点 CPUUsage/RamUsage；删除 DNS 记录；发送 stop 分析事件
2. DNS 服务器（internal/dns/server.go）
   API 服务内嵌一个 UDP DNS 服务器（Consul 可配置端口），解析 {sandboxId} → orchestrator 节点 IP：
   ● 使用 Redis（生产）或内存 map（本地）作为后端存储
   ● key: sandbox.dns.{sandboxId}，24h TTL
   ● 未找到时返回 127.0.0.1（让 client-proxy 返回 sandbox-not-found，而非 502）
   ● DNS 记录在 sandbox insert/delete 钩子中实时维护
3. Edge Cluster Pool（internal/edge/）
   管理多个 client-proxy 集群（edge clusters）的 gRPC 连接池：
   ● 每 60s 从 DB GetActiveClusters 同步
   ● 每个 Cluster 对应一个 client-proxy 集群，维护 HTTP client（REST Edge API）+ gRPC client（pass-through proxy）
   ● 用于向 client-proxy 注册/注销 sandbox catalog 条目、以及通过 pass-through 转发 gRPC 请求
4. TemplateManager 客户端（internal/template-manager/）
   ● 维护两类 gRPC 连接：local client（本机 orchestrator 的 template-manager 角色）和 cluster edge client（通过 edge pass-through 转发到具体节点）
   ● BuildsStatusPeriodicalSync：每 1min 从 DB 查询 Building 状态的构建，调用 TemplateBuildStatus 同步最新状态
5. InstanceCache（internal/cache/instance/）
   内存中的全量 sandbox 状态缓存，带超时机制：
   ● 超时后自动触发 delete 钩子（auto-pause 或 delete）
   ● Reserve()：检查团队并发限制（Tier.ConcurrentInstances）
   ● WaitForPause()：等待正在进行中的 pause 操作完成，供 resume 时获取 pause 的目标节点

Sandbox 创建完整流程
POST /sandboxes
→ 鉴权（API Key/Access Token/Supabase JWT → Team + Tier）
→ templateCache.Get() 验证模板存在且 Team 有权限
→ 检查 secure flag → 生成 envd access token（ECDSA/HMAC）
→ orchestrator.CreateSandbox()
→ instanceCache.Reserve() 检查并发限制
→ 选择 least-busy orchestrator 节点
→ orchestrator gRPC: SandboxService.Create(SandboxConfig)
→ instanceCache.Add()
→ insert 钩子: CPUUsage++, DNS 写入, 分析事件
→ 返回 {sandboxID, clientID(节点ID), templateID, envdVersion, envdAccessToken}
Pause 流程
POST /sandboxes/:id/pause
→ GetSandbox() 验证存在且属于该 Team
→ orchestrator.DeleteInstance(sandboxID, pause=true)
→ 从 instanceCache 移除 → 触发 delete 钩子
→ delete 钩子: 写 DB snapshot record
→ 调 orchestrator gRPC: Sandbox.Pause(sandboxId, templateId, buildId)
→ DB: EnvBuildSetStatus = success
→ sbx.Pausing.WaitWithContext() 等待 pause 完成
→ 204 No Content
Resume 流程
POST /sandboxes/:id/resume
→ 检查 sandbox 未在运行
→ WaitForPause() 等待可能进行中的 pause（获取 pause 的目标节点 ID 用于就近 resume）
→ DB: GetLastSnapshot() 获取最近快照（含 build 信息、metadata、secure 标志）
→ 复用 startSandbox()，isResume=true，clientID=pause节点
→ orchestrator gRPC: SandboxService.Create(Snapshot=true, clientID)
→ 201 Created（与新建 sandbox 相同返回格式）

依赖关系总结
API Service
├── Nomad API → 发现 orchestrator 节点列表
├── orchestrator gRPC → Create/Delete/Pause/List sandboxes
├── template-manager gRPC → Build/Status/Delete templates
├── Supabase/PostgreSQL → Team/Tier/Template/Snapshot/Build 数据
├── Redis → DNS 映射 + sandbox catalog（供 client-proxy 使用）
├── ClickHouse → sandbox 指标查询（可选）
├── Loki → sandbox 日志查询
├── Posthog → 用户行为分析
├── edge cluster pool → sandbox catalog 注册 + gRPC pass-through
└── 内嵌 DNS Server → 响应 client-proxy 的 sandbox → nodeIP 查询