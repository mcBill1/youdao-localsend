# LocalSend 词典笔版

把 [LocalSend](https://github.com/localsend/localsend) 移植到网易有道词典笔（iot_miniapp_sdk 小程序宿主）的互传应用。与官方 LocalSend v2 协议（含新版 1.18.x 的 rustls 强制 TLS 1.3 + mTLS）兼容，支持双向发现、双向互传。

- 当前版本：**v1.10.0**
- 两个机型（同一套逻辑层，两套 UI，构建期按机型注入）：

| 系列 | 屏幕 | 布局 | APPID |
|---|---|---|---|
| **Space** | 568×1210 竖屏（现机） | 上中下（header/列表/footer） | `8010001822789980` |
| **X** | 240×320 面板横置（横屏，绘制区约 320×240） | 首页三栏：左=刷新/接收记录，中=设备列表，右=LocalSend/设置/日志 | `8010001822789981` |

## 目录结构

```
youdao-localsend/
├── README.md / docs/        # 本文件 + 开发文档
│   ├── DEVELOPMENT.md       # ⭐ 开发文档：协议实测、架构、构建坑清单、调试手段、硬约束
│   ├── CHANGELOG.md         # 版本开发日志（每版更新了啥、解决了啥）
│   └── localsend-porting-guide.md
├── env/                     # 环境（勿改动）
│   ├── iot_miniapp_sdk-master/   # 官方 SDK；apps/localsend/ 为构建工作区，resources/presetpkgs/ 放产物
│   └── toolchain/                # Linaro aarch64 交叉编译器（Windows 可直跑）
├── src/                     # 源码
│   ├── core/app.core.js     # 逻辑层（协议层/service/注册段，两机型共用；版本号在此）
│   ├── ui-space/            # Space 系列 UI（theme.js + pages.js）
│   ├── ui-x/                # X 系列 UI（横屏三栏）
│   ├── native/              # C 模块（libjsapi_localsend.c + 自签证书头，两机型共用）
│   └── assets/              # 打包资源（图标、libjsapi_bridge.so 等）
├── tools/
│   ├── build.py             # ⭐ 一键构建脚本
│   ├── amr/                 # dump.exe（JS→字节码）+ pack_amr.py（打包 .amr）
│   ├── tls/                 # mbedTLS 3.6.4 源码 + 静态库
│   └── debug/               # mock_localsend.py（本机模拟对端抓包）、tls_probe.py
├── build/                   # 构建产物（.so），可随时清
└── ref/                     # 参考资料（预留）
```

## 编译

环境：Windows + Python 3，SDK 与工具链已内置于 `env/`，无需额外安装。

```powershell
cd youdao-localsend

python tools\build.py                          # Space 系列（默认），版本沿用并提醒
python tools\build.py --model x                # X 系列
python tools\build.py --ver 1.10.1 --install   # 指定版本号 + 构建后 adb 装机
```

- `--ver`：严格 `X.X.X` 格式；省略则沿用现版本（脚本会醒目提醒 bump）
- `--model space|x`：选择 UI 系列与 APPID（默认 space）
- `--install`：adb 推送并安装（adb 取自 `PenCheck\bin\adb.exe`，需已连接设备）

构建五步链（脚本自动完成）：①版本/日期填充 → ②组装 app.js（core 两个标记位注入对应 ui 目录）→ ③gcc 交叉编译 .so（mbedTLS 静态链入）→ ④dump.exe 编字节码 → ⑤pack_amr.py 打包。产物在 `env\iot_miniapp_sdk-master\resources\presetpkgs\<APPID>.amr`。

细节与坑清单（漏 dump.exe、`-Bsymbolic`、PSA 随机数等——逐条都是真机踩出来的）见 [DEVELOPMENT.md 第 4 节](docs/DEVELOPMENT.md)。

## 运行时文件（笔端）

自有文件全部收在 `/userdisk/localsend/` 下：

```
/userdisk/localsend/
├── （接收的文件）
├── data/prefs.conf      # 设置（别名/PIN/主题/日志/TLS）
└── tmp/                 # 运行临时文件（日志、fd 表）
```
