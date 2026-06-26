# Felorx Connect 通用远程连接设计

## 背景

`apps/reevibe-server` 目前是 ReeVibe 专用 WebRTC 信令服务，只负责会话码、peer 加入、offer/answer/ICE 转发。新的目标是把它重构为通用连接层，用于所有远程操作场景，首先服务于 Ops 的远程终端，并作为可选服务挂到 `sync_node` 中。最终移除 `apps/reevibe-server`，将可复用代码迁入 `packages/sync/felorx_connect`。

现有代码中有几个可复用基础：

- `apps/reevibe/lib/pages/session_page.dart` 已验证 Flutter 侧 `flutter_webrtc` + DataChannel + 终端 IO 的基本模式。
- `apps/ops/lib/pages/terminal/local_terminal_page.dart` 已有成熟本地终端体验，底层是 `xterm` + `flutter_pty`。
- `apps/sync_node/lib/sync_node_runner.dart` 已有可返回运行句柄的启动流程，适合增加可选子服务并在 stop 时统一释放。
- `packages/api/felorx_api_client` 和 `felorx_shared` 已有设备列表、设备绑定和当前用户设备信息能力，可用于同账号发现的第一版身份基础。

## 目标

1. 创建 `packages/sync/felorx_connect`，作为通用远程连接协议、信令服务、host daemon 和客户端核心的共享包。
2. 将 `reevibe-server` 的 WebSocket 信令能力迁入 `felorx_connect`，改成通用 relay/signaling server。
3. 在 `sync_node` 中新增可选 connect relay 服务，随 sync node 一起启动和停止。
4. 新增 Linux 服务器命令行 daemon，可作为 systemd 等服务运行，注册本机并承接远程终端连接。
5. 给 Ops 增加远程终端能力，并明确区分本地终端和远程终端。
6. 默认通过 relay 完成身份、发现和 WebRTC 信令；终端数据优先通过 WebRTC DataChannel 点对点传输。
7. 支持两种连接方式：同账号设备发现，以及设备码 + 连接密码。
8. 配置 TURN 后允许中继兜底；未配置 TURN 时直连失败应明确提示。
9. 从 workspace 移除 `apps/reevibe-server`。

## 非目标

第一期不新增 ABP 后端数据库表或重新生成服务端 API。在线 host 注册表先放在 relay 内存中，后续再按产品需求落库。

第一期不把所有终端数据通过 WebSocket relay 转发作为默认路径。WebSocket relay 只承载注册、发现、授权和 WebRTC SDP/ICE 信令。TURN 只在显式配置时作为 ICE 兜底。

第一期不替代 Ops 现有 SSH/SFTP/Docker/Kubernetes 页面。远程终端是新的连接后端，未来其他远程操作可以基于同一连接层扩展。

## 总体架构

`felorx_connect` 自包含连接层，提供三个主要模块：

- Relay/Signaling Server：HTTP + WebSocket 服务，维护在线 host、同账号发现、设备码认证、连接会话和 SDP/ICE 转发。
- Host Daemon：纯 Dart Linux 服务进程，连接 relay 后注册本机，收到连接请求后创建 WebRTC PeerConnection，DataChannel 打开后连接到本机 PTY shell。
- Client Core：给 Ops 和未来其他应用使用，封装 host 查询、设备码连接、WebRTC 协商、终端 input/output/resize/exit 消息。

依赖方向：

- `apps/sync_node` 依赖 `felorx_connect`，用于启动 relay。
- `apps/ops` 依赖 `felorx_connect`，并继续使用 Flutter 侧 `flutter_webrtc` 实现客户端 WebRTC。
- `apps/reevibe` 后续可以复用 `felorx_connect` 的信令协议，但本次核心范围是移除 `reevibe-server` 并支持 Ops 远程终端。

## 包结构

建议初始文件职责如下：

- `packages/sync/felorx_connect/lib/felorx_connect.dart`：公共导出。
- `packages/sync/felorx_connect/lib/src/protocol/connect_message.dart`：版本化消息模型、解析和编码。
- `packages/sync/felorx_connect/lib/src/protocol/connect_error.dart`：协议错误码和可展示错误。
- `packages/sync/felorx_connect/lib/src/relay/connect_relay_server.dart`：relay 生命周期、HTTP 路由、WebSocket 升级。
- `packages/sync/felorx_connect/lib/src/relay/host_registry.dart`：在线 host 注册、心跳、超时清理、同账号过滤。
- `packages/sync/felorx_connect/lib/src/relay/session_registry.dart`：连接会话、peer 关系、SDP/ICE 转发目标。
- `packages/sync/felorx_connect/lib/src/auth/connect_password.dart`：设备码连接密码 hash 和验证。
- `packages/sync/felorx_connect/lib/src/client/connect_relay_client.dart`：Ops/daemon 共用 relay WebSocket 客户端基础。
- `packages/sync/felorx_connect/lib/src/client/connect_terminal_client.dart`：远程终端客户端抽象。
- `packages/sync/felorx_connect/lib/src/host/connect_host_daemon.dart`：host 注册、连接请求处理、WebRTC 会话编排。
- `packages/sync/felorx_connect/lib/src/host/terminal_host_session.dart`：DataChannel 到 PTY 的桥接。
- `packages/sync/felorx_connect/lib/src/host/pty/terminal_pty.dart`：PTY 抽象。
- `packages/sync/felorx_connect/lib/src/host/pty/linux_terminal_pty.dart`：Linux PTY 实现。
- `packages/sync/felorx_connect/bin/felorx_connect_host.dart`：Linux host daemon CLI 入口。

## Relay 服务

Relay 暴露：

- `GET /connect/health`：健康检查。
- `GET /connect/info`：协议版本、relay 能力、TURN 配置是否可用。
- `WS /connect/ws`：注册、发现、连接请求和 SDP/ICE 信令。

Host daemon 建立 WebSocket 后发送 `registerHost`，内容包含：

- `hostId`：本机稳定设备标识，默认使用 sync/device token 或本地生成的 connect host id。
- `displayName`、`platform`、`systemVersion`。
- `userId`：同账号发现使用；来自登录 token/API key 验证结果或 relay 可验证声明。
- `deviceCode`：设备码连接入口使用。
- `capabilities`：至少包含 `terminal`、`directWebRtc`，配置 TURN 后包含 `turnFallback`。
- `authModes`：`sameAccount`、`deviceCodePassword`。

Relay 对 host 进行心跳管理。host 超过心跳 TTL 后从在线注册表移除，并通知正在查看列表的客户端刷新。

## 授权模型

同账号连接：

1. Ops 登录后连接 relay。
2. Relay 从认证上下文或 token 验证中得到 `userId`。
3. Ops 发送 `hostList` 请求。
4. Relay 只返回同一 `userId` 下在线 host。
5. Ops 选择 host 后发送 `connectRequest`。
6. Relay 验证双方 userId 一致后转发给 host。

设备码 + 连接密码：

1. Host daemon 本地生成或配置 `deviceCode` 和连接密码。
2. Host 向 relay 注册 `deviceCode`，密码只保存在 host 本地或 relay 内存中的 hash 形式。
3. Ops 输入设备码和连接密码后发送 `connectRequest`。
4. Relay 验证设备码存在，再用 hash 校验连接密码。
5. 校验通过后创建短期连接会话，并将请求转发给 host。

密码不写入通用设备模型，不出现在设备列表 API 中。建议 host 支持环境变量和配置文件：

- `PUUPEE_CONNECT_DEVICE_CODE`
- `PUUPEE_CONNECT_PASSWORD`
- `PUUPEE_CONNECT_PASSWORD_HASH`

若同时提供明文和 hash，优先使用 hash，并在日志中提示明文配置被忽略。

## WebRTC 和数据通道

Relay 只转发以下信令：

- `offer`
- `answer`
- `iceCandidate`
- `connectAccept`
- `connectReject`
- `error`

终端数据通过 DataChannel 承载，消息类型包括：

- `terminalInput`：客户端输入，payload 为 UTF-8 文本或字节。
- `terminalOutput`：host 输出，payload 为 UTF-8 文本或字节。
- `resize`：终端列、行、像素宽高。
- `exit`：shell 退出码。
- `heartbeat`：DataChannel 活性检测。
- `error`：会话级错误。

客户端和 host 都带 `protocolVersion`。不兼容版本应在 relay 或 DataChannel 握手阶段拒绝，并返回可展示错误。

默认 ICE 使用 STUN/P2P。配置 TURN 后允许 TURN relay candidate 参与 ICE。Ops UI 需要根据 WebRTC candidate pair 或连接状态显示：

- `直连`
- `TURN 中继`
- `连接失败`

没有 TURN 配置时，直连失败不退回 WebSocket 全流量转发。

## Linux Host Daemon

新增 CLI：`felorx_connect_host`。

主要参数：

- `--relay-url`：relay 地址，例如 `wss://example.com/connect/ws`。
- `--api-key`：用于以服务方式登录当前账号。
- `--token-file`：复用或保存登录 token。
- `--host-id`：可选稳定 host id。
- `--name`：显示名称，默认取 hostname。
- `--device-code`：设备码连接使用；不传则自动生成并打印。
- `--connect-password`：设备码连接密码。
- `--connect-password-hash`：连接密码 hash。
- `--shell`：默认 `/bin/bash`，不存在时降级 `/bin/sh`。
- `--stun-url`：默认公共 STUN，可通过参数覆盖。
- `--turn-url`、`--turn-username`、`--turn-credential`：TURN 兜底配置。

Daemon 需要适合 systemd 运行：

- 前台运行，日志写 stdout/stderr。
- 收到 SIGINT/SIGTERM 后关闭 DataChannel、PTY、WebSocket 并退出。
- 启动失败返回非 0 退出码。
- 不依赖 Flutter。

PTY 实现优先使用可兼容 Dart 3.8+ 的纯 Dart/FFI 方案。如果现成 `pty` 包不兼容当前 SDK，则在 `felorx_connect` 内实现薄 Linux PTY FFI 适配层，接口保持在 `TerminalPty` 抽象之后。

## sync_node 集成

`sync_node` 新增可选参数：

- `--enable-connect-relay`
- `--connect-port`
- `--connect-path-prefix`
- `--connect-stun-url`
- `--connect-turn-url`
- `--connect-turn-username`
- `--connect-turn-credential`
- `--connect-host-heartbeat-timeout`

对应环境变量使用 `FELORX_SYNC_NODE_CONNECT_*` 前缀。`runSyncNode` 启动 `FelorxSyncServer` 后，如果启用 connect relay，则启动 `ConnectRelayServer`。`SyncNodeRunHandle.stop()` 增加 connect relay 停止逻辑，确保端口释放。

第一期 relay 可以单独监听 `connect-port`。后续如果需要同端口挂载到 sync server 的 shelf router，再调整 `FelorxSyncServer` 对外暴露 mount 能力。

## Ops 远程终端体验

Ops 的“终端”能力改为本地和远程两种来源：

- 本地终端继续使用现有 `xterm` + `flutter_pty`。
- 远程终端复用现有终端外观、字体设置、标签页、复制、粘贴、保存输出等交互。
- 远程终端 IO 后端替换为 `felorx_connect` 客户端 DataChannel。

页面结构：

- 模式切换：`本地终端`、`远程终端`。
- 远程终端左侧入口：`我的机器`、`设备码连接`。
- `我的机器` 列表展示同账号在线 host：名称、平台、最近心跳、能力、连接状态。
- `设备码连接` 表单包含设备码和连接密码。
- 成功连接后打开远程终端标签。

移动端可以展示远程终端入口。本地 PTY 仍沿用当前隐藏或降级策略。

实现时建议先抽取共享终端视图组件，避免本地终端、SSH 终端和 connect 远程终端各自维护一套 `TerminalView` 细节。

## 迁移和移除

迁移顺序：

1. 创建 `packages/sync/felorx_connect`。
2. 将 `reevibe-server` 的 session 和 WebSocket 转发逻辑迁入 relay 模块，并改成通用 host/session 命名。
3. 增加协议模型和 relay 单元测试。
4. 增加 Linux host daemon CLI。
5. 在 `sync_node` 增加可选 relay 服务。
6. 在 Ops 增加远程终端入口和 DataChannel IO 后端。
7. 从根 `pubspec.yaml` workspace 移除 `apps/reevibe-server`。
8. 删除 `apps/reevibe-server` 目录。
9. 更新或删除 `scripts/reevibe_server.dart` 中的旧引用。

## 测试策略

`felorx_connect` 单元测试：

- 消息解析和编码拒绝未知版本。
- host 注册后可以被同 userId 客户端发现。
- 不同 userId 的 host 不出现在同账号列表中。
- 设备码密码正确时允许连接，错误时拒绝。
- host 心跳超时后被移除。
- relay 正确转发 offer/answer/iceCandidate 到目标 peer。

`sync_node` 测试：

- 参数解析包含 connect relay 参数。
- 未启用时不启动 connect relay。
- 启用时启动 relay 并在 stop 时关闭。

Ops 测试：

- 本地和远程模式切换不破坏现有本地终端入口。
- 设备码表单校验空设备码、空密码。
- 远程 host 列表加载、错误和空状态可展示。
- 远程终端标签能接收 output、发送 input、发送 resize。

WebRTC 真实联通作为集成 smoke test，使用本地 relay 和本机两个 peer 验证 DataChannel 打开和终端 echo，不放进普通单元测试的必跑路径。

## 验收标准

- `packages/sync/felorx_connect` 存在并可 `dart test`。
- `sync_node` 可以通过参数启用 connect relay。
- Linux 服务器可以运行 `felorx_connect_host` 并注册在线。
- Ops 可以看到同账号在线机器并打开远程终端。
- Ops 可以通过设备码和连接密码打开远程终端。
- 成功连接时终端数据不经过 relay WebSocket 转发。
- 配置 TURN 时可以作为 ICE 兜底，并在 UI 中标记中继模式。
- `apps/reevibe-server` 从 workspace 和文件系统移除。
