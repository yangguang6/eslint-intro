# 漫谈 ESLint 共享配置

## 关于共享配置

**共享配置（ [shareable configuration](https://eslint.org/docs/developer-guide/shareable-configs) ）** 是一个导出配置对象的 npm 包，它使我们可以和其他项目共享 ESLint 配置。
如 `eslint-config-airbnb`、`eslint-config-standard` 。

在 `extends` 中使用：

```javascript
// .eslintrc.js

module.exports = {
    extends: ['standard']
}
```

## 关于插件

**插件（ [plugin](https://eslint.org/docs/developer-guide/working-with-plugins) ）** 是一个可以对 ESLint 进行各种扩展的 npm 包。它可以包括添加新的规则、导出一个共享配置等许多功能。
如 `eslint-plugin-vue`、`eslint-plugin-react` 。

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

通常，一个共享配置往往包含了一些自定义规则（也就是插件），但是 ESLint 在加载插件时是 [以项目根目录进行查找](https://eslint.org/docs/developer-guide/shareable-configs#publishing-a-shareable-config) ，
因此，官方建议将共享配置的相关插件指定为 `peerdependencies` ，也就是需要用户在项目中随同共享配置一同安装。

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

> 但是对于第三方 parser 或者 共享配置，ESLint 会相对于当前配置文件路径进行查找 😵 ，目前还没找到为什么这么做的原因。

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

而对用户来说，这样的安装方式是非常奇怪的，用户并不用关心配置内部到底使用了哪些插件。相比 `npm install --save-dev eslint-config-standard`
的安装方式用户体验差很多。

## 探讨

令人惊讶的是，这个问题其实在 2015 年就有人提出，也是许多开发者（如 standard 、 creat-react-app 的作者等等）的诉求，但是此
issue（ [#3458](https://github.com/eslint/eslint/issues/3458) ）的讨论持续至今仍未得到官方的正式支持。不过，通过阅读这个问题下的讨论，我也总结出了三个针对目前情况的变通方案。

### 方案一：依赖包管理器的依赖提升机制

在 npm 3 以后，所有的依赖安装都会被提升到项目更目录下。依赖这一特性，在共享配置中将插件指定为直接依赖（dependencies），那么插件的包会被提升至
项目根目录的 node_modules 下。此时 ESLint 就能找到依赖的插件。

不过，此方案最大的问题是包管理器的依赖提升并不是可靠的，如果包发生冲突，那么包就不会被提升。

### 方案二：开发 plugin

## 关于插件

## 基本配置

## 共享配置

- 与 plugin 的区别

## 共享配置的使用

## 如何写一个共享配置

## dependencies or peerDependencies ?

## overrides 覆盖机制


- extends 机制
- overrides 机制


# 漫谈 ESLint 共享配置

---

# 关于共享配置

**共享配置（[shareable configuration](https://eslint.org/docs/developer-guide/shareable-configs)）** 是一个导出配置对象的 npm 包，它使我们可以和其他项目共享 ESLint 配置。
常见的共享配置：`eslint-config-airbnb`、`eslint-config-standard` 。

##### 使用：

```javascript
// .eslintrc.js

module.exports = {
    extends: ['standard']
}
```

---

# 共享配置与插件的区别与联系

<br />

**插件（[plugin](https://eslint.org/docs/developer-guide/working-with-plugins)）** 是一个可以对 ESLint 进行各种扩展的 NPM 包

---
