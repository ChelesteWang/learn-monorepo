
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