# 漫谈 ESLint 共享配置

## 关于共享配置

**共享配置（ [shareable configuration](https://eslint.org/docs/developer-guide/shareable-configs) ）** 是一个导出配置对象的 npm 包，它使我们可以和其他项目共享 ESLint 配置。如 `eslint-config-airbnb`、`eslint-config-standard` 。

在 `extends` 中使用：

```javascript
// .eslintrc.js

module.exports = {
    extends: ['standard']
}
```

## 关于插件

**插件（ [plugin](https://eslint.org/docs/developer-guide/working-with-plugins) ）** 是一个可以对 ESLint 进行各种扩展的 npm 包。它可以包括添加新的规则、导出一个共享配置等许多功能。如 `eslint-plugin-vue`、`eslint-plugin-react` 。

使用：

```javascript
// .eslintrc.js

module.exports = {
    extends: ['plugin:react/recommended']
}
```

手动配置规则：

```javascript
// .eslintrc.js

module.exports = {
    plugins: ['react'],
    rules: {
        'react/jsx-uses-react': 'error',
    }
}
```

## 当共享配置依赖插件

通常，一个共享配置往往包含了一些自定义规则（也就是插件），但是 ESLint 在加载插件时是 [以项目根目录进行查找](https://eslint.org/docs/developer-guide/shareable-configs#publishing-a-shareable-config) ，因此，官方建议将共享配置的相关插件指定为 `peerdependencies` ，也就是需要用户在项目中随同共享配置一同安装。

加载 plugin 部分的源码（删减）：

```javascript
class ConfigArrayFactory {
    
    _loadPlugin(name, ctx) {

        // const pluginBasePath =
        //     resolvePluginsRelativeTo ||
        //     (filePath && path.dirname(filePath)) ||
        //     cwd;
        const relativeTo = path.join(ctx.pluginBasePath, "__placeholder__.js");

        // const request = 'eslint-plugin-vue' // 这里假设插件为 'eslint-plugin-vue'
        
        let filePath;

        try {
            filePath = resolver.resolve(request, relativeTo);
        } catch (resolveError) {
            // ...
        }

        // ...
    }
}
```

> 但是对于第三方 parser 或者 共享配置，ESLint 会相对于当前配置文件路径进行查找。

加载 parser 部分的源码（删减）：

```javascript
class ConfigArrayFactory {

    _loadParser(nameOrPath, ctx) {

        // const filePath = providedFilePath
        //     ? path.resolve(cwd, providedFilePath)
        //     : "";
        const relativeTo = ctx.filePath || path.join(cwd, "__placeholder__.js");

        try {
            const filePath = resolver.resolve(nameOrPath, relativeTo);
            
            // ...
        } catch (resolveError) {
            // ...
        }

        // ...
    }
}
```

加载共享配置部分的源码（删减）：

```javascript
/**
 * Create a new context with default values.
 * @param {ConfigArrayFactoryInternalSlots} slots The internal slots.
 * @param {"config" | "ignore" | "implicit-processor" | undefined} providedType The type of the current configuration. Default is `"config"`.
 * @param {string | undefined} providedName The name of the current configuration. Default is the relative path from `cwd` to `filePath`.
 * @param {string | undefined} providedFilePath The path to the current configuration. Default is empty string.
 * @param {string | undefined} providedMatchBasePath The type of the current configuration. Default is the directory of `filePath` or `cwd`.
 * @returns {ConfigArrayFactoryLoadingContext} The created context.
 */
function createContext(
    { cwd, resolvePluginsRelativeTo },
    providedType,
    providedName,
    providedFilePath,
    providedMatchBasePath
) {
    const filePath = providedFilePath
        ? path.resolve(cwd, providedFilePath)
        : "";
    const matchBasePath =
        (providedMatchBasePath && path.resolve(cwd, providedMatchBasePath)) ||
        (filePath && path.dirname(filePath)) ||
        cwd;
    const name =
        providedName ||
        (filePath && path.relative(cwd, filePath)) ||
        "";
    const pluginBasePath =
        resolvePluginsRelativeTo ||
        (filePath && path.dirname(filePath)) ||
        cwd;
    const type = providedType || "config";

    return { filePath, matchBasePath, name, pluginBasePath, type };
}

class ConfigArrayFactory {

    _loadExtendedShareableConfig(extendName, ctx) {
        const { cwd, resolver } = internalSlotsMap.get(this);
        const relativeTo = ctx.filePath || path.join(cwd, "__placeholder__.js");  // 这里的 ctx.filePath 即当前的配置文件路径
        let request;
        
        // ...
        
        // request = 'eslint-config-standard' // 这里假设 request 的值为 'eslint-config-standard'

        let filePath;

        try {
            filePath = resolver.resolve(request, relativeTo);
        } catch (error) {
            // ...
        }

        // ...
        
        return this._loadConfigData({
            ...ctx,
            filePath,
            name: `${ctx.name} » ${request}`
        });
    }
}
```

由于这样的设计，对于像 `eslint-config-standard` 这样的共享配置的安装方式就成了这样：

```shell
npm install --save-dev eslint-config-standard eslint-plugin-promise eslint-plugin-import eslint-plugin-node
```

而对用户来说，这样的安装方式是非常奇怪的，用户并不用关心配置内部到底使用了哪些插件。相比 `npm install --save-dev eslint-config-standard`的安装方式用户体验差很多。

## 讨论

这个问题其实在 2015 年就有人提出，也是许多开发者（如 standard 、 creat-react-app 的作者等等）的诉求，但是此issue（ [#3458](https://github.com/eslint/eslint/issues/3458) ）的讨论持续至今仍未得到官方的正式支持（也是因为没有完美的解决办法）。

### ESLint 作者的观点

#### 历史

根据 ESLint 作者 [所述](https://github.com/eslint/eslint/issues/3458#issuecomment-252068708) ，ESLint 一开始首先支持了 plugin，一段时间后才支持了共享配置。当时官方鼓励用户使用 peer dependencies，以此表示共享配置和插件之间的对等关系。这种方式在当时工作得很好，因为 npm 会默认安装 peer dependencies 。可惜在 npm 3 以后，npm 不会默认安装 peer dependencies，而是交给用户去手动安装这些依赖。由于 npm 的改变，导致这种对等关系已不是最佳设计。

> 注：在 npm 7 以后，npm 又默认支持了安装 peer dependencies。

#### 与 npm 包的差异

共享配置与其他的 npm 模块工作方式不同 (that's a feature, not a bug)。

早期的假设是 **每次运行只能加载一个插件实例** 。

举个例子，假设有一个插件 `eslint-config-myplugin` ，其配置如下：

```yaml
plugins:
  - myplugin
rules:
  myplugin/rule: "error"
```

首先，我们基于 plugins 数组中的 `myplugin` 来加载插件，然后我们引用插件中的一条规则为 `myplugin/rule` 。为了使用这个约定，我们必须 100% 确定 `myplugin` 指的是什么，并且它必须在项目中的所有配置中引用同一个东西，以确保每个配置都可以覆盖其他配置的设置。这就意味着 **每个配置都不能像 npm 包查找那样搜索插件的目录结构** 。我们为插件保留一个缓存，并用它来查找。

#### 难点

基于以上所述，之所以这一问题没有进展是因为一个非常严重的边缘情况：如何处理一个项目中的两个不同的配置引用了同一个插件的两个不同版本的情况？

举个例子，假设：

- 我们有一个 `eslnt-config-foo` ，它依赖 `eslint-plugin-foo@2.0.0`，因为想要 “foo” 的规则。
- 同时用户有一个直接依赖 `eslint-plugin-foo@1.2.0` 。

在这种情况下，就会有几个问题：

1. 在配置文件中仅仅写了“foo“的情况下，如何确定应该加载 `eslint-plugin-foo` 的版本？
2. 如果我们默认选择最顶级（highest-up）的包（也就是 `eslint-plugin-foo@1.2.0`），如果 `eslint-config-foo` 引用了一个 2.0 版本才有的规则，那么 ESLint 会因为这条规则的丢失而报错。
3. 如果我们选择加载第一个最靠近指定它的配置的包，在另一方面也会有同样的问题：用户配置引用的规则可能被删除或选项格式可能已经改变，导致ESLint出错。
4. 如果我们选择两个包全部加载，则会产生规则和环境命名冲突的风险。

基于以上问题，可能会有一些解决方案：

**如果试图加载同一个包两次，那么抛出一个错误**

这似乎很难做到。首先，我们没有很好的方法知道加载的插件是否是不同的版本。其次，用户可能无法修复这个问题，因为可能使用的几个共享配置每个都依赖一个不同版本的插件。惩罚终端用户使用相互矛盾的插件包版本的配置似乎是一个错误的选择。

**找到可以加载同一个包的不同版本的方法**

首先，这意味着要找到一种方法来加载同一个包的两个不同版本，而 Node.js 在默认情况下并不是这样设计的，所以我们不得不在文件系统中”爬行“来做到这一点。由于 npm `node_modules` 扁平化，在文件系统中爬行变得更加困难，所以我们无法真正知道要查找的正确位置，这意味着要搜索大量的包。我们还需要找到一种方法来确保用户在任何时候都知道他们正在配置插件的哪个版本，这可能意味着我们需要一个不同的命名方案来配置共享配置和用户安装的插件。这也十分困难，因为共享配置和 ESLint 内部的任何其他配置其实一样，所以这也会有很多工作。

**创建一个新的可共享的东西，以我们想要的方式工作**

这里最主要的问题是，我们已经有了一个庞大的共享配置和插件的生态系统，我们真的想要创建一些新的东西来满足这种需求吗？

### 变通方案

不过，通过阅读这个问题下的讨论，我总结出了几个针对目前情况的变通方案。

#### 方案一：依赖包管理器的依赖提升机制

在 npm 3 以后，所有的依赖安装都会被提升到项目更目录下。依赖这一特性，在共享配置中将插件指定为直接依赖（dependencies），那么插件的包会被提升至项目根目录的 node_modules 下。此时 ESLint 就能找到依赖的插件。

不过，此方案最大的问题是包管理器的依赖提升并不是可靠的，如果包发生冲突，那么包就不会被提升。

#### 方案二：[使用 plugin 替代](https://github.com/eslint/eslint/issues/3458#issuecomment-257161846)

插件可以包含配置，也可以指定其它插件作为依赖。

这个方案的问题是规则前必须加上 plugin 的名字。假如我们的插件叫做 `eslint-plugin-airbnb` ，它依赖插件 `eslint-plugin-react` ，如果配置了一条 react 的规则 `react/jsx-uses-vars`，则会变成 `airbnb/jsx-uses-vars` ，如下所示：

```yaml
plugins:
  - airbnb
extends:
  - plugin:airbnb/es6
rules:
  airbnb/jsx-uses-vars: "error" 
```

如果想做一些区分，那么可以变成 `airbnb/react/jsx-uses-vars` 。但是如果层级过多，那么前缀就会变得很长。同时，规则命名的改变也会对项目中 ESLint disable 的注释语法产生影响。

#### 方案三：[install-peerdeps](https://www.npmjs.com/package/install-peerdeps)

通过 install-peerdeps 去自动安装 peer dependencies ，例如：

```shell
npx install-peerdeps --dev eslint-config-airbnb-base
```

#### 方案四：[@rushstack/eslint-patch](https://www.npmjs.com/package/@rushstack/eslint-patch)

`@rushstack/eslint-patch` 对 ESLint 解析插件路径的代码进行了覆盖，使其相对于当前配置文件路径查找。使用方式：

```javascript
require("@rushstack/eslint-patch/modern-module-resolution");

module.exports = {
  extends: ['@your-company/eslint-config']
}
```

## 新提案

由于目前的 ESLint 配置十分复杂且难以满足一些需求，一个新的提案 —— [Config File Simplification](https://github.com/eslint/rfcs/tree/main/designs/2019-config-simplification) 已经通过且正在实现中。

这个提案提供了一种通过新的配置文件格式来简化 ESLint 配置的方法。配置文件格式是用 JavaScript 编写的，并删除了几个现有的配置项 ，以便允许用户手动创建它们。

新的配置形式会更加灵活，可以支持共享配置和插件捆绑的需求。

### 新配置形式下的共享配置

在此设计下，共享配置可以将插件指定为直接依赖，并随共享配置一起安装，从而改善用户体验。

举个例子，假设有两个共享配置，`eslint-config-a` 和 `eslint-config-b` 都依赖插件 `eslint-plugin-example`，但是一个依赖 1.0.0 版本，另一个依赖 2.0.0 版本。那么 npm 会像这样安装这些插件：

```
your-project
├── eslint.config.js
└── node_modules
  ├── eslint
  ├── eslint-config-a
  | └── node_modules
  |   └── eslint-plugin-example@1.0.0
  └── eslint-config-b
    └── node_modules
      └── eslint-plugin-example@2.0.0
```

npm 会正确处理这种情况，将正确的插件版本安装在正确的地方。

此时也会有一个问题，那就是两个共享配置尝试使用默认的命名空间：

```javascript
const example = require("eslint-plugin-example");

module.exports = {
    plugins: {
        example
    }
};
```

如果两个可共享的配置都这样做，并且用户试图同时使用两个可共享的配置，那么当这些配置被规范化时将会抛出一个错误，因为插件命名空间 `example` 只能被分配一次。

为了解决这个问题，用户可以为同一个插件创建一个单独的命名空间，这样它就不会与共享配置中已有的插件命名空间冲突。例如，假设我们想要使用 `eslint-config-first` ，并且它定义了一个 `example` 插件命名空间。同时我们也想使用 `eslint-config-second` ，它也同样定义了一个 `example` 插件命名空间。由于一个插件命名空间不能被定义两次，所以 ESLint 会报错。但是通过 `eslint-config-second` 创建一个使用不同命名空间的新配置可以使我们仍然使用这两个共享配置。如下所示：

```javascript
// get the config you want to extend
const configToExtend = require("eslint-config-second");

// create a new copy (NOTE: probably best to do this with @eslint/config somehow)
const compatConfig = Object.create(configToExtend);
compatConfig.plugins["compat::example"] = require("eslint-plugin-example");
delete compatConfig.plugins.example;

// include in config
module.exports = [
    require("eslint-config-first"),
    compatConfig,
    {
        // overrides here
    }
];
```

### 进展

https://github.com/eslint/eslint/issues/13481
