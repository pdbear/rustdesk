# 自托管内置配置修改说明

本文记录本项目中“内置 server、key、连接密码”的相关代码位置和修改方式。

## 术语

- ID Server：`hbbs` 地址，客户端用它注册 ID、查询对端、打洞。
- Relay Server：`hbbr` 地址，直连失败时使用。
- API Server：账号、地址簿等 HTTP API 地址，可为空。
- key：自托管 `hbbs` 的公钥，客户端用它校验 rendezvous server。
- 连接密码：需要区分两种：
  - 控制别人时自动带上的默认密码：`default-connect-password`。
  - 别人连接本机时可使用的预置永久密码：`password` + 可选 `salt`。

## 相关代码

- 默认公共 server 和公共 key：
  - `libs/hbb_common/src/config.rs`
  - `RENDEZVOUS_SERVERS`
  - `RS_PUB_KEY`
- server/key 的运行期配置项：
  - `custom-rendezvous-server`
  - `relay-server`
  - `api-server`
  - `key`
- 配置读取优先级：
  - `Config::get_rendezvous_server()` 优先读 `EXE_RENDEZVOUS_SERVER`，再读 `custom-rendezvous-server`，再读内置/缓存/默认 server。
  - `Config::get_options()` 合并顺序为 `DEFAULT_SETTINGS` -> 用户配置 -> `OVERWRITE_SETTINGS`。
- 自定义客户端配置入口：
  - `src/common.rs`
  - `load_custom_client()`
  - `read_custom_client()`
  - `default-settings`
  - `override-settings`
  - 其余顶层字段进入 `HARD_SETTINGS`。
- UI 导入 server 配置：
  - `flutter/lib/common.dart`
  - `ServerConfig`
  - `setServerConfig()`
- 默认连接密码：
  - `src/client.rs`
  - 读取 `OPTION_DEFAULT_CONNECT_PASSWORD`，即 `default-connect-password`。
- 本机预置永久密码：
  - `libs/hbb_common/src/config.rs`
  - `Config::get_preset_password_storage_and_salt()`
  - 读取 `HARD_SETTINGS["password"]` 和 `HARD_SETTINGS["salt"]`。

## 推荐方式：使用 custom.txt 内置自托管配置

`custom.txt` 是自定义客户端配置文件。debug 构建会优先读取当前目录下的 `./custom.txt`；正式包会读取可执行文件同目录的 `custom.txt`，macOS 在 `../Resources/custom.txt`。

配置解码逻辑在 `src/common.rs::read_custom_client()`。文件内容不是普通 JSON，而是签名后的 base64 数据；解码成功后得到 JSON，再按下面规则注入：

- `default-settings` 写入默认配置，用户后续可以覆盖。
- `override-settings` 写入强制配置，用户界面和配置文件无法覆盖。
- 其他顶层字符串字段写入 `HARD_SETTINGS`，适合放预置密码、salt 等硬配置。

推荐模板如下：

```json
{
  "app-name": "YourDesk",
  "override-settings": {
    "custom-rendezvous-server": "hbbs.example.com:21116",
    "relay-server": "hbbr.example.com:21117",
    "api-server": "https://api.example.com",
    "key": "YOUR_HBBS_PUBLIC_KEY_BASE64",
    "remove-preset-password-warning": "Y",
    "disable-change-permanent-password": "Y"
  },
  "default-settings": {
    "verification-method": "use-both-passwords",
    "approve-mode": "password"
  },
  "password": "00BASE64_OF_SHA256_PASSWORD_PLUS_SALT",
  "salt": "RANDOM_32_CHARS_OR_LONGER"
}
```

如果希望用户可以在设置页修改 server/key，把 `override-settings` 改为 `default-settings`。

注意选项归属：`app-name` 是特殊顶层字段；其他顶层字符串字段会进入 `HARD_SETTINGS`，适合 `password`、`salt` 这类 hard settings；`default-settings`/`override-settings` 可以写普通设置和 builtin 设置。不要把 `default-connect-password`、`hide-server-settings`、`remove-preset-password-warning` 等 builtin 选项放在顶层。

如果希望彻底固定 server/key，并隐藏设置入口，可以额外加入：

```json
{
  "override-settings": {
    "custom-rendezvous-server": "hbbs.example.com:21116",
    "relay-server": "hbbr.example.com:21117",
    "api-server": "https://api.example.com",
    "key": "YOUR_HBBS_PUBLIC_KEY_BASE64",
    "hide-server-settings": "Y",
    "hide-network-settings": "Y"
  }
}
```

注意：上面的 JSON 是签名前的明文配置形态。项目当前只内置信任 RustDesk 官方自定义客户端签名公钥，见 `src/common.rs::read_custom_client()` 中的 `KEY` 常量。如果要自己生成并加载 `custom.txt`，需要同时替换这里的验签公钥，或走现有自定义客户端生成链路。

## server 和 key

自托管最少需要设置：

```json
{
  "override-settings": {
    "custom-rendezvous-server": "hbbs.example.com:21116",
    "relay-server": "hbbr.example.com:21117",
    "key": "YOUR_HBBS_PUBLIC_KEY_BASE64"
  }
}
```

端口可省略时的默认逻辑：

- `custom-rendezvous-server` 不带端口时，代码会补 `21116`。
- `relay-server` 为空时，通常由服务端返回；仍为空时会按 rendezvous host 端口 +1 推导为 `21117`。

`key` 必须和自托管 `hbbs` 的公钥一致。`hbbs` 第一次启动会生成密钥文件，部署时应把公钥填入客户端配置。

## 连接密码

### 1. 控制别人时的默认密码

配置项：`default-connect-password`

用途：当前客户端发起连接时，如果没有从对端配置、地址簿等地方拿到密码，会用这个值自动参与密码认证。

示例：

```json
{
  "override-settings": {
    "default-connect-password": "RemoteSidePassword"
  }
}
```

这个值属于 `BUILTIN_SETTINGS`，应放在 `default-settings` 或 `override-settings` 中。

### 2. 别人连接本机时的预置永久密码

配置项：

- `password`
- `salt`

用途：当前设备作为被控端时，如果本地没有用户设置的永久密码，认证逻辑会回退使用这里的预置密码。

不推荐明文，但兼容明文：

```json
{
  "password": "PlainTextPassword"
}
```

推荐使用带 salt 的哈希格式：

```json
{
  "password": "00BASE64_OF_SHA256_PASSWORD_PLUS_SALT",
  "salt": "RANDOM_32_CHARS_OR_LONGER"
}
```

生成方式：

```bash
password='YourStrongPassword'
salt='RANDOM_32_CHARS_OR_LONGER'
printf '%s%s' "$password" "$salt" | openssl dgst -sha256 -binary | openssl base64 -A
```

把输出前面加上 `00`，填入 `password`。

预置密码相关 UI：

- 如果本地没有用户设置的永久密码，但 hard settings 里有可用预置密码，UI 会显示“正在使用预置密码”提示。
- 如需去掉预置密码警告，可设置 `remove-preset-password-warning = "Y"`。
- 如需禁止用户修改永久密码，可设置 `disable-change-permanent-password = "Y"`。

## 运行期导入方式

如果不是做内置包，只是给当前安装实例配置自托管 server，可使用 UI 或 CLI。

UI 路径：

- 设置中的 ID/Relay Server 弹窗最终会调用 `setServerConfig()`。
- 写入：
  - `custom-rendezvous-server`
  - `relay-server`
  - `api-server`
  - `key`

CLI 路径：

```bash
sudo rustdesk --option custom-rendezvous-server hbbs.example.com:21116
sudo rustdesk --option relay-server hbbr.example.com:21117
sudo rustdesk --option api-server https://api.example.com
sudo rustdesk --option key YOUR_HBBS_PUBLIC_KEY_BASE64
```

已安装场景下，命令需要管理员/root 权限。

也可以使用 `--import-config` 导入 `RustDesk.toml` 和对应的 `RustDesk2.toml`。其中 server/key 位于 `RustDesk2.toml` 的 `options` 内。

## 直接改源码默认值

本仓库已支持从 GitHub Actions 的 Variables 或 Secrets 注入编译期默认值。可配置项：

- `RENDEZVOUS_SERVER`：默认 ID Server，例如 `hbbs.example.com` 或 `hbbs.example.com:21116`。
- `RELAY_SERVER`：默认 Relay Server，可选；如果 `hbbr` 和 `hbbs` 同 host 且端口按 `21116/21117` 部署，可以不填。
- `API_SERVER`：默认 API Server，例如 `https://api.example.com`。
- `RS_PUB_KEY`：自托管 `hbbs` 公钥。

这些值只要在编译时非空，就会作为强制内置值使用。运行过程中生成的配置文件、UI 设置、CLI `--option` 写入的同名参数都不会覆盖它们。只有某个编译期变量为空时，程序才会回落到读取本地配置文件或源码默认值。

GitHub Actions 会优先读取 repository/environment Variables，缺省再读取 Secrets：

```yaml
API_SERVER: ${{ vars.API_SERVER || secrets.API_SERVER }}
RENDEZVOUS_SERVER: ${{ vars.RENDEZVOUS_SERVER || secrets.RENDEZVOUS_SERVER }}
RELAY_SERVER: ${{ vars.RELAY_SERVER || secrets.RELAY_SERVER }}
RS_PUB_KEY: ${{ vars.RS_PUB_KEY || secrets.RS_PUB_KEY }}
```

本地构建也可以直接传环境变量：

```bash
API_SERVER='https://api.example.com' \
RENDEZVOUS_SERVER='hbbs.example.com:21116' \
RELAY_SERVER='hbbr.example.com:21117' \
RS_PUB_KEY='YOUR_HBBS_PUBLIC_KEY_BASE64' \
./build.py --flutter
```

实现位置：

- `RENDEZVOUS_SERVER`：`libs/hbb_common/src/config.rs` 中的 `Config::get_rendezvous_server()` / `Config::get_rendezvous_servers()`。
- `RELAY_SERVER`：`src/rendezvous_mediator.rs` 中的 `get_relay_server()`。
- `RS_PUB_KEY`：`src/common.rs` 中的 `get_key()`。
- `API_SERVER`：`src/common.rs` 中的 `get_api_server_()`。
- 环境变量重编译跟踪：`build.rs`、`libs/hbb_common/build.rs`

如果只想 fork 后硬编码默认公共配置，也可以直接改源码：

```rust
pub const BUILD_RENDEZVOUS_SERVER: &str = "hbbs.example.com";
pub const DEFAULT_RENDEZVOUS_SERVER: &str = "rs-ny.rustdesk.com";
pub const RENDEZVOUS_SERVERS: &[&str] = &[DEFAULT_RENDEZVOUS_SERVER];
pub const BUILD_RELAY_SERVER: &str = "hbbr.example.com";
pub const BUILD_RS_PUB_KEY: &str = "YOUR_HBBS_PUBLIC_KEY_BASE64";
```

文件：`libs/hbb_common/src/config.rs`

这种方式只影响没有 `custom-rendezvous-server`、没有 custom client 默认/覆盖配置、也没有本地缓存 server 的场景。已经运行过的客户端可能已有 `RustDesk2.toml` 配置，需要清理或覆盖配置后才会回到源码默认值。

## 建议配置组合

固定自托管 server/key，但让用户自己设置被控密码：

```json
{
  "app-name": "YourDesk",
  "override-settings": {
    "custom-rendezvous-server": "hbbs.example.com:21116",
    "relay-server": "hbbr.example.com:21117",
    "api-server": "https://api.example.com",
    "key": "YOUR_HBBS_PUBLIC_KEY_BASE64"
  }
}
```

固定自托管 server/key，并预置被控永久密码：

```json
{
  "app-name": "YourDesk",
  "override-settings": {
    "custom-rendezvous-server": "hbbs.example.com:21116",
    "relay-server": "hbbr.example.com:21117",
    "api-server": "https://api.example.com",
    "key": "YOUR_HBBS_PUBLIC_KEY_BASE64",
    "verification-method": "use-both-passwords",
    "approve-mode": "password",
    "disable-change-permanent-password": "Y",
    "remove-preset-password-warning": "Y"
  },
  "password": "00BASE64_OF_SHA256_PASSWORD_PLUS_SALT",
  "salt": "RANDOM_32_CHARS_OR_LONGER"
}
```

固定自托管 server/key，并让当前客户端连接别人时自动尝试一个默认密码：

```json
{
  "app-name": "YourDesk",
  "override-settings": {
    "custom-rendezvous-server": "hbbs.example.com:21116",
    "relay-server": "hbbr.example.com:21117",
    "key": "YOUR_HBBS_PUBLIC_KEY_BASE64",
    "default-connect-password": "RemoteSidePassword"
  }
}
```

## 修改后验证

1. 启动客户端后检查设置页 ID/Relay Server 是否显示预期值。
2. 检查日志中注册 server 是否为目标 `hbbs`。
3. 用另一个客户端连接本机，确认预置永久密码是否生效。
4. 清理旧配置后再次启动，确认内置默认值仍能生效。
