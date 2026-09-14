# CTPStock Minimal MD Login Test

这是一个独立、可审阅的 CTP 个股期权行情登录对照项目，用于排查：

```text
OnFrontConnected 已收到
ReqUserLogin 返回 0
但 60 秒内没有收到登录响应
```

程序不依赖另一个连接测试项目，也没有配置解析、请求号过滤、自动重试或行情订阅。它只执行：

```text
创建行情 API
→ 注册回调与前置地址
→ Init
→ OnFrontConnected
→ ReqUserLogin
→ 打印 OnRspUserLogin / OnRspError / 断线 / 心跳回调
→ Release
```

当前固定测试参数：

| 项目 | 值 |
| --- | --- |
| MD Front | `tcp://101.226.254.157:32213` |
| BrokerID | `1000` |
| UserID | `887120202987` |
| SDK | `v3.7.5_CP_20251125` Windows x64 |

密码在运行时隐藏输入，不写入源码、命令行或日志。行情 API 没有 `ReqAuthenticate`，所以这个测试不需要 APPID 或 AuthCode。程序不订阅行情、不查询账户、不报单。

## 还需要放入一个本地文件

仓库已经包含编译所需的三个 UTF-8 头文件和 3.7.5 x64 import library。原厂 DLL 不上传到公开 GitHub。

从下面的原始 SDK 目录取出：

```text
traderAPI_3.7.5_CP_20251125(1).zip
└─ 3.7.5_CP_api_20251125_win
   └─ 20251125_traderapi64_windows_se
      └─ soptthostmduserapi_se.dll
```

把它放到：

```text
CTPStock_MinimalMDTest\sdk\bin\soptthostmduserapi_se.dll
```

可在 PowerShell 中执行：

```powershell
cd D:\projects\CTPStock_MinimalMDTest
New-Item -ItemType Directory -Force sdk\bin
Copy-Item "D:\你的SDK解压目录\3.7.5_CP_api_20251125_win\20251125_traderapi64_windows_se\soptthostmduserapi_se.dll" sdk\bin\
```

必须使用 `20251125_traderapi64_windows_se`，不能使用 Windows 32 位目录、Linux `.so` 或 3.7.0 DLL。`sdk/bin` 已被 `.gitignore` 排除。

准备完成后的主要结构：

```text
CTPStock_MinimalMDTest\
├─ main.cpp
├─ build_windows.bat
├─ run_windows.bat
└─ sdk\
   ├─ include\
   │  ├─ ThostFtdcMdApi.h
   │  ├─ ThostFtdcUserApiStruct.h
   │  └─ ThostFtdcUserApiDataType.h
   ├─ lib\
   │  └─ soptthostmduserapi_se.lib
   └─ bin\
      └─ soptthostmduserapi_se.dll
```

## 编译和运行

要求：Windows x64，以及安装了“使用 C++ 的桌面开发”和 Windows SDK 的 Visual Studio 或 Build Tools。

```powershell
cd D:\projects\CTPStock_MinimalMDTest
.\build_windows.bat
.\run_windows.bat
```

输入交易密码时屏幕不显示字符，输完按 Enter。

程序开头必须显示：

```text
API: v3.7.5_CP_20251125  9:30:02.f7e78374
```

如果显示 3.7.0，应立即停止测试并检查 DLL 路径。

## 如何判断输出

| 输出 | 含义 |
| --- | --- |
| `CALLBACK OnFrontConnected` | SDK 与行情前置已经建立连接 |
| `RETURN ReqUserLogin immediate_rc=0` | 本地 SDK 接受了请求，不代表柜台已经登录成功 |
| `CALLBACK OnRspUserLogin ... error_id=0` | 收到柜台登录成功响应 |
| `CALLBACK OnRspUserLogin ... error_id!=0` | 柜台明确拒绝登录，按错误码和原文排查 |
| `CALLBACK OnRspError` | SDK/柜台返回通用错误 |
| `OnFrontConnected` 后 60 秒超时 | 登录请求已提交，但本程序没有收到最终登录或错误回调 |

若这个项目和原程序在同一电脑、同一时间、同一账号、同一 3.7.5 SDK 下都表现为连接成功、`immediate_rc=0`、随后登录超时，则原程序的配置解析、状态机和请求号过滤基本可以排除。下一步应由券商查询 `32213` 行情前置服务端日志、账户行情权限及柜台要求的准确 SDK build。

客户端对照只能提供强证据，不能仅凭一次超时证明所有客户端代码在数学意义上绝对无误。

## 已完成的离线检查

- `main.cpp` 使用实际 3.7.5 头文件通过 C++17 语法检查；
- 使用假 SDK 跑通 `Init → OnFrontConnected → ReqUserLogin → OnRspUserLogin → Release`；
- 同步回调发生在 `ReqUserLogin` 返回之前时，等待逻辑仍能正确结束；
- 没有在维护环境执行 Windows 原生 DLL 加载或真实柜台登录，实机输出仍是最终证据。
