
> 由于结课项目的工程需要 monorepo 因此记录一下基于 pnpm 的 monorepo 包管理实践

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1b4b8c61cfdc4e09831b0a06dddb7274~tplv-k3u1fbpfcp-zoom-1.image)

## Monorepo 的优点

-   **工作流的一致性**

-   **项目基建成本的降低**

-   **降低依赖管理的复杂度**

-   **团队协作也更加容易**

## 限制只能使用pnpm

pnpm 的 monorepo 项目在 node_modules 以及开发中，项目依赖 pnpm workspace 使用 npm 或 yarn 运行时会出现问题。

因此需要在安装依赖之前对包管理器进行检查。

```
"scripts": {
  "preinstall": "node ./scripts/preinstall.js"
}
```

实现当运行 npm install 或 yarn，就会发生错误并且不会继续安装。

```
if (!/pnpm/.test(process.env.npm_execpath || '')) {
  console.warn(
    `\u001b[33mThis repository requires using pnpm as the package manager ` +
      ` for scripts to work properly.\u001b[39m\n`
  )
  process.exit(1)
}
```

## 安装初始化依赖

```
pnpm install
```

pnpm install 会同时为根目录和所有 workspace 中指定的 package 安装需要的依赖相应依赖

## 添加全局依赖

```
pnpm add rimraf -D -W
```

**-D** 把依赖作为 devDependencies 安装；**-W** 把依赖安装到根目录的 `node_modules`

虽然 packages 下的项目都没有安装 rimraf ，但是倘若在项目中使用到，就会逐级往上寻找

**devDependencies 与 dependencies**

`dependencies`：**运行时依赖**，模块消费者会安装这里的依赖。

`devDependencies`：**开发时依赖**，消费者不会安装，只为生产者服务。

`peerDependencies`：**宿主依赖**，指定了当前模块包在**使用前**需要**安装的依赖**

## 添加局部依赖

pnpm install jquery -r --filter @monorepo/package-a

-   对于某些依赖，可能仅存在于某几个 package 中，我们就可以单独为他们安装，当然，可以通过`cd packges/xxx` 后，执行 pnpm add xxx 但这样重复操作多次未免有些麻烦，pnpm 提供了一个快捷指令 **——filter**。
-   比如我们只在 package-a 应用中用到 react，那就可以为它单独安装。首先要拿到它的package name，在本例中是@monorepo/package-a：
-   在packages/package-a/package.json中，我们可以看到：

```
 "dependencies": {
   "react": "^17.0.0"
 }
```

-   packages/package-a的node_modules中看到react被添加了进来。

## 内部包的相互引用

pnpm add @monorepo/package-a -r --filter @monorepo/package-b

-   在 monorepo 中，我们往往需要 package 间的引用，比如本例中的@monorepo/package-a就会被@monorepo/package-b引用。
-   在packages/package-b/package.json中，我们可以看到：

```
"dependencies": {
  "@monorepo/package-a": "workspace:^1.0.0"
}
```

-   这时你会有一个疑问：当这样的工具包被发布到平台后，如何识别其中的workspace呢？
-   那就需要执行 **pnpm publish**，会把基于的workspace的依赖变成外部依赖

```
// before
"dependencies": {
  "@monorepo/package-a": "workspace:^1.0.0"
}
// after
"dependencies": {
  "@monorepo/package-a": "^1.0.0"
}
```

## 项目 git Lint 配置

**在多人协同的项目中，需要代码提交时，约束 commit 信息**。

一个 monorepo 仓库可能被**不同的开发者提交不同子项目**的代码，如果没有规范化的 commit 信息，在故障排查或版本回滚时毫无意外会遭遇灾难。因此，千万不要小看 commit 信息格式化的重要性

为了我们能够一目了然的追踪每次代码变更的信息，我们使用 commitlint 工具作为格式化 commit 信息的不二之选。

顾名思义，`commitlint` 可以帮助我们检查提交的 commit 信息，它强制约束我们的 commit 信息必须在开头附加指定类型，用于标示本次提交的大致意图，支持的类型关键字有：

-   `feat`：表示添加一个新特性；
-   `chore`：表示做了一些与特性和修复无关的「家务事」；
-   `fix`：表示修复了一个 Bug；
-   `refactor`：表示本次提交是因为重构了代码；
-   `style`：表示代码美化或格式化；
-   ...

除了限定 commit 信息类型外，commitlint 还支持（虽然不是必须的）显示指定我们本次提交所对应的子项目名称。假如我们有一个名为 `@monorepo/package-a` 的子项目，我们针对该项目提交的 commit 信息可以写为：

```
git commit -m "feat(package-a): add a default page" 
```

我们可以通过下面的命令安装 `commitlint` 以及周边依赖：

```
pnpm add @commitlint/cli @commitlint/config-conventional commitlint husky -W -D
```

通过安装 husky ，它能够帮助我们在提交 commit 信息时自动运行 `commitlint` 进行检查

其他的如何配置项目的 Eslint prettier 就不多赘述了，网上文章很多可以进行参考。

书接上回，上一篇文章讲解了如何使用 pnpm 管理 monorepo 仓库，以及一个 monorepo 项目如何初始化。当我们的项目开发完成需要对他进行打包，发包，部署。如何高效有序的完成这一系列任务，也就此成为了一个难题。如何让整个流程可以全自动一条龙，成为 monorepo 方案中需要处理的重点。

## 社区方案

Nx 和 Turborepo 都是目前社区中较为完善的方案，两个方案都有一个共同点，那就是整个方案体量较大，可以说是 monorepo 当前阶段的一些最佳实践的集合，Lerna + yarn workspace 的方案也是之前社区的主流方案，当然 pnpm workspace 方案相比于 yarn workspace 有很多新的优点，管理上也更加方便，其他的详细的可以看一看社区的文章，本文只是记录一下个人对于社区上 pnpm monorepo 的初步实践，私下从开源社区中了解到 ChangeSet 也是一个很好用的 monorepo 发包工具，下面便以此方案为主。

PS. 方案个有千秋，不拉不踩只对踩坑过程进行记录

## 打包方案

```json
{
  "name": "package-a",
  "version": "1.0.0",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1",
    "dev": "node index.js",
    "build": "tsc"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "description": ""
}
```

pnpm 提供了 --filter 方法可以在项目根目录下运行 monorepo 子包的 package scripts 因此我们可以使用

```shell
pnpm run build --filter package-a
```

当我们需要一次性打包 monorepo 所有子包的时候，pnpm 也提供了递归执行子包的 package scripts 的方法

```shell
pnpm run build -r
```

```shell
pnpm run build -r
```

当我们对代码修改后需要重新发包穿透的方式需要手动修改 `package.json` 中的版本，并更新目录中的 changelog 当然很多时候我们并没有更新 changelog 的习惯，但是作为一个开源项目的话，changelog 非常重要，可以帮助使用者和开发者更好的了解项目使用项目维护项目，下面我们使用 Changesets 来管理 monorepo 的版本。

## 使用 Changesets 自动版本管理

根目录下安装 @changesets/cli 并初始化

```shell
pnpm install @changesets/cli -W -D  && npx changeset init
```

### **使用 changeset add 记录版本修改**

```shell
npx changeset add 
```

输入后会让你选择要记录更改的包，使用空格进行选择，并要求填写 changelog 记录

```shell
npx changeset add
🦋 Which packages would you like to include? · package-a
🦋 Which packages should have a major bump? · package-a
🦋 Please enter a summary for this change (this will be in the changelogs).
🦋 (submit empty line to open external editor)
🦋 Summary · feat: done a function
🦋  
🦋 === Summary of changesets ===
🦋 major: package-a
🦋  
🦋 Note: All dependents of these packages that will be incompatible with
🦋 the new version will be patch bumped when this changeset is applied.
🦋  
🦋 Is this your desired changeset? (Y/n) · true
🦋 Changeset added! - you can now commit it
🦋  
🦋 warn This Changeset includes a major change and we STRONGLY recommend adding more information to the changeset:
🦋 warn WHAT the breaking change is
🦋 warn WHY the change was made
🦋 warn HOW a consumer should update their code
🦋 info /Users/wangxinyuan/Documents/code/learn-monorepo/.changeset/stale-shrimps-design.md
```

于是在根目录下 `.changeset`文件夹下生成了 `stale-shrimps-design.md`

```md
---
"package-a": major
---

feat: done a function
```

版本号一般有三个部分，以`.`隔开，就像`X.Y.Z`，其中

- X：主版本号，不兼容的大改动，major
- Y：次版本号，功能性的改动，minor
- Z：修订版本号，问题修复，patch

每个部分为整数（>=0），按照递增的规则改变。

### **使用 changeset version 提交版本修改**

```shell
npx changeset version
```

执行后之前生成的 `stale-shrimps-design.md`会被消费掉并修改对应的子包下的 `package.json`并生成`CHANGELOG.md`

package.json

```json
{
  "name": "package-a",
  "version": "2.0.0",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1",
    "dev": "node index.js",
    "build": "tsc"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "description": ""
}
```

CHANGELOG.md

```md
# package-a

### Major Changes

- feat: done a function
```

到现在可以说我们在发包前的准备工作都做完了

可能报的错：

1. 未登录  
  `npm ERR!` code ENEEDAUTH  
  `npm ERR!` need auth auth required for publishing  
  `npm ERR!` need auth You need to authorize this machine using `npm adduser`
  
  **解决办法：`npm adduser` 后输入用户名，密码，邮箱**
  

2. 仓库地址不对  
  `npm ERR!` code E409  
  `npm ERR!` Registry returned 409 for PUT on https://registry.npmmirror.com/
  
  **解决办法：可以切换为 npm 源**
  
3. 你的包有命名冲突
  
  npm ERR! code E403
  npm ERR! 403 403 Forbidden - PUT https://registry.npmjs.org/package-a - You do not have permission to publish "package-a". Are you logged in as the correct user?
  npm ERR! 403 In most cases, you or one of your dependencies are requesting
  npm ERR! 403 a package version that is forbidden by your security policy, or
  npm ERR! 403 on a server you do not have access to.
  
  **解决办法：给你的包改个名，或者加个 scope**
  
4. 你的包在 scope 中，得加钱，要加发带 scope 的公有包
  
  **解决办法：在子包 package.json 加入 publishConfig**
  
  ```json
  "publishConfig": {
      "access": "public"
    }
  ```
  

### **使用 changeset publish 进行发包**

```shell
npx changeset publish
```

执行后他会将 monorepo 中的子包中有更新的包全部 publish

```shell
npx changeset publish  
🦋  info npm info @yuqing521/package-a
🦋  info npm info @yuqing521/package-b
🦋  info npm info @yuqing521/package-c
🦋  warn Received 404 for npm info "@yuqing521/package-c"
🦋  warn Received 404 for npm info "@yuqing521/package-a"
🦋  warn Received 404 for npm info "@yuqing521/package-b"
🦋  info @yuqing521/package-a is being published because our local version (1.0.0) has not been published on npm
🦋  info @yuqing521/package-b is being published because our local version (1.0.0) has not been published on npm
🦋  info @yuqing521/package-c is being published because our local version (1.0.0) has not been published on npm
🦋  info Publishing "@yuqing521/package-a" at "1.0.0"
🦋  info Publishing "@yuqing521/package-b" at "1.0.0"
🦋  info Publishing "@yuqing521/package-c" at "1.0.0"
🦋  success packages published successfully:
🦋  @yuqing521/package-a@1.0.0
🦋  @yuqing521/package-b@1.0.0
🦋  @yuqing521/package-c@1.0.0
🦋  Creating git tags...
🦋  New tag:  @yuqing521/package-a@1.0.0
🦋  New tag:  @yuqing521/package-b@1.0.0
🦋  New tag:  @yuqing521/package-c@1.0.0
```

## 使用 Github Action 实现自动发包

changeset 官方提供了 [Github Action](https://github.com/changesets/action) 模板以及 [Changesets robot](https://github.com/apps/changeset-bot) 下面我们可以参考编写自己的 Github Action

先为根目录下 package.json 添加两个 script 方便后续使用

```json
"version": "changeset version",
"publish": "changeset publish"
```

编写 release.yml 就很简单了

```yaml
name: Release

on:
  push:
    branches:
      - master

jobs:
  release:
    name: Release
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Repo
        uses: actions/checkout@v2
        with:
          fetch-depth: 0

      - name: Setup Node.js 12.x
        uses: actions/setup-node@v2
        with:
          node-version: 12.x
          
      - name: Install pnpm
        uses: pnpm/action-setup@v2
        with:
          version: 6

      - name: Install Dependencies
        run: pnpm install
        
      - name: Build Package
        run: pnpm run build:all

      - name: Create Release Pull Request
        uses: changesets/action@v1
        with:
          publish: pnpm run publish
          version: pnpm run version
          commit: '[ci] release'
          title: '[ci] release'
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NPM_TOKEN: ${{ secrets.NPM_AUTH_TOKEN }}
```

流程基本就是本地 `changeset add` 后生成变更文件，提交 pr 会触发 ci 自动去将其发包 (安装依赖，打包，消费变更文件，发包)

当然还需要配置 `GITHUB_TOKEN `, `NPM_TOKEN`具体如何使用 Github Action 就需要诸君自己探索了。