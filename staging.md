---
theme: keepnice
---
## 实测数据

我在 Windows 环境下，使用 PowerShell 的 `Measure-Command` 对 pnpm 命令进行了计时测试。

断网环境下：

```powershell
Measure-Command { cmd /c "npx pnpm --version" }
# 耗时：1.34 秒
```

联网环境下：

```powershell
Measure-Command { cmd /c "npx pnpm --version" }
# 耗时：2.33 秒
```

作为对比，直接执行本地安装的 pnpm：

```powershell
Measure-Command { cmd /c "pnpm --version" }
# 耗时：0.50 秒
```

从数据可以看出：
- 断网时 npx 耗时 1.34 秒，联网时耗时 2.33 秒
- 两者都明显慢于直接执行本地命令的 0.50 秒
- 联网比断网还要慢约 1 秒

另外，在不同的网络环境和设备环境下，数据可能会有差异。

## 原因分析

npx 的设计目标是允许用户执行任何 npm 包命令，无需提前安装。为实现这一目标，npx 在执行时需要完成以下工作：

1. 查询 npm registry，获取目标包的最新版本信息
2. 检查本地是否已安装该包
3. 根据版本信息决定是否需要下载或更新
4. 执行命令

断网时，npx 会尝试连接 registry 但超时失败，这个过程需要等待，所以仍有 1.34 秒的开销。

联网时，npx 成功查询 registry，但网络请求本身需要时间，因此耗时更长，达到 2.33 秒。

即使目标包已经存在于项目的 `node_modules/.bin` 目录或全局包目录中，registry 查询这一步仍然会执行。这是 npx 的设计逻辑决定的：它需要确保执行的版本是最新的，或者至少是与用户预期一致的。

简单来说，npx 是为"通用场景"设计的——它不知道你本地有没有装，所以每次都要去 registry 确认一下。这种设计在临时执行未安装包时很有用，但在执行本地已安装包时就成了多余开销。

## 替代工具推荐

如果使用场景仅限于执行项目本地已安装的包，这里推荐一个轻量级替代工具：**binpx**。

它的核心逻辑很简单：
- 从当前目录向上查找 `node_modules/.bin` 目录
- 找到目标命令后直接执行
- 不查询 registry，不检查版本更新

在相同环境下测试：

```powershell
Measure-Command { cmd /c "binpx pnpm --version" }
# 耗时：0.68 秒
```

与直接执行本地 bin 的 0.50 秒非常接近，比 npx 的 2.33 秒快得多。

binpx 还有一些额外功能：
- 支持 Node.js 模块调用，可作为其他工具的依赖（具体详见项目 README）
- 本地找不到时自动降级到系统 PATH，兼容全局安装的包
- 统一的命令格式，避免不同包管理器（npm、pnpm、yarn）的差异

可以直接使用 npm 安装 binpx：

```bash
# 添加至项目开发依赖
npm install -D binpx
# 或全局安装
npm install -g binpx
```

项目链接：
- GitHub: https://github.com/zhuxiaojt/binpx
- npm: https://www.npmjs.com/package/binpx
