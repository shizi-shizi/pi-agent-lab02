# 实践课 3：Pi Agent 源码安装与编译

姓名拼音：`yuanshiwen` · GitHub：`shizi-shizi`

## 1. 检查工具版本

我先在 PowerShell 里查了版本：Node.js 是 `v24.18.0`，npm 是 `11.16.0`，Git 是 `2.55.0.windows.2`。这台电脑已经装了 Node.js 24，不用再装一遍。

![S1：Node.js、npm 和 Git 的实际版本](lab03-s1.jpg)

## 2. 确认本地源码

我继续用实验二克隆好的仓库。进入 `D:\CodexProject\Pi-Agent-Lab02\pi-hunnu-2026` 后，我用 `git remote -v` 确认 `origin` 是小组仓库，不是老师的仓库；分支是 `main`，完整提交号是 `c3cc91632d0a1950fb122605b190233f2aa0006e`。`git status --short` 没有输出，说明这时没有未提交的改动。

实际 clone 地址：<https://github.com/shizishizi111/pi-hunnu-2026.git>

![S2：本地路径、远端、分支和提交号](lab03-s2.jpg)

## 3. 看安装和构建配置

根目录 `README.md` 的 Development 一节给出了安装、联网构建和离线重建命令。根 `package.json` 的 `engines.node` 写的是 `>=22.19.0`，所以 Node.js 24 是实验推荐版本，并不是配置里的最低版本。它的 `workspaces` 把 `packages/`、`apps/` 等目录里的包放在一起管理，根目录下的 `package-lock.json` 记录了解析后的依赖和版本。TypeScript 和 esbuild 等构建工具列在根 `devDependencies` 里。

`npm run build` 读的是当前包 `package.json` 中的 `scripts.build`。在根目录执行时，脚本按顺序进入各个子包，让它们自己执行 `build`。这些命令用 `&&` 连起来，前一步出错，后面的包就不会继续构建。最后进入 `packages/coding-agent`，它自己的 `bin.pi` 指向 `dist/bundle/cli.js`。

![S3：根 package.json 中的 engines.node 和 scripts.build](lab03-s3.jpg)

## 4. 安装依赖

我在仓库根目录运行 `npm install --ignore-scripts`。这个命令按 `package.json` 和锁文件安装依赖，把工具和库装进本地 `node_modules`，也会关联仓库里的工作区包。`--ignore-scripts` 让安装阶段跳过 `prepare`、`postinstall` 等脚本，之后手动运行 `npm run build` 仍然有效。这次没有用 `-g`，不会把全局的 `pi` 换成实验版本。

本次安装新增了 333 个包，审计了 356 个包，退出码是 `0`，根目录下也有了 `node_modules`。日志里的警告和审计结果见第 7 节。

![S4：依赖安装汇总、警告和退出码](lab03-s4.jpg)

## 5. 构建并运行产物

我在仓库根目录运行了 `npm run build`。日志先列出各子包的编译过程，到 `ai` 包时联网生成模型目录。这一步下载的是模型名称、能力等资料，不是模型权重。最后 `coding-agent` 编译 TypeScript、复制程序资源，再打出 `dist/bundle`，日志显示 84 个文件、共 9.3 MiB，构建退出码是 `0`。

接着我用 `Test-Path .\packages\coding-agent\dist\bundle\cli.js` 检查入口，返回 `True`；运行 `node .\packages\coding-agent\dist\bundle\cli.js --version` 得到 `1.5.2`，退出码也是 `0`。这样能确认跑的是这次构建出的文件，不是 PATH 上的全局 `pi`。

![S5：根构建完成、退出码以及本地 CLI 验证](lab03-s5.jpg)

以后如果模型目录数据还有效，可以用 `npm run build:offline` 重新编译，省去联网刷新目录这一步。本次首次构建用的是联网命令。

## 6. 启动源码版并提问

我先运行 `.\pi-test.bat --version`，同样返回 `1.5.2`。随后直接运行 `.\pi-test.bat` 进入交互界面，问了“请用一句话说明源码和编译产物的区别。只回答问题，不修改文件，也不要调用工具。”模型实际回复了，把 TypeScript 代码和 `dist` 中生成的 JavaScript 作为例子。验证到回复以后，我退出了 Pi。

![S6a：源码入口的版本、启动命令和项目提示](lab03-s6a.jpg)

![S6b：Pi 终端里的提示词和模型回复](lab03-s6b.jpg)

图 S6b 从实际终端输出截取，保留了问题和最终回复，省略了思考过程和用量统计。

我看了 `.bat` 和 `.ps1`：前者调用后者，后者让 Node.js 加载源码解析器，再跑 `packages/coding-agent/src/cli.ts`。源码入口适合改好 `src` 后试运行；`dist/bundle/cli.js` 适合检查编译和打包后的版本能不能运行。两个入口这次都显示 `1.5.2`，所以要看运行路径，不能只看版本号。

## 7. 遇到的问题和处理

**安装和构建时有 npm 警告。** 图 S4 里，npm 说不识别 `.npmrc` 中的 `min-release-age-exclude`，还提示 `prebuild-install` 和 `node-domexception` 已弃用。我没有改项目配置，而是先看安装退出码，再继续跑完整构建。两个退出码都是 `0`，这几条警告没有让实验中断。本次审计还报告了 7 项高危问题。这个结果不能因为程序能跑就忽略，也不能把它当成安装失败；我没有运行 `npm audit fix --force` 或改锁文件。

**源码版启动时询问是否信任项目。** 图 S6a 里提示信任后可以加载项目的 `.pi` 配置和扩展。我只需要做一次普通问答，所以选了 `Do not trust (this session only)`。图 S6b 还能看到项目资源被跳过的提示，但会话正常收到了模型回复。本次没有改信任设置，也没有重新登录。
