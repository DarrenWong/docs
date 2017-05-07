title: Config 
---

框架提供了强大且可扩展的配置功能，可以自动合并应用、插件、框架的配置，按顺序覆盖，且可以根据环境维护不同的配置。合并后的配置可直接从 `app.config` 获取。
Egg provides powerful and extensible feature that automatically merge configurations of applications, plugins, framework, 
sequence overrides, and maintain configurations depend on the different environments. 
The merged configuration is available from `app.config`.

配置的管理有多种方案，以下列一些常见的方案
Several optionals for the config management as shown below,

1. 使用平台管理配置，应用构建时将当前环境的配置放入包内，启动时指定该配置。但应用就无法一次构建多次部署，而且本地开发环境想使用配置会变的很麻烦。Platform to config, config file will be specified at startup and pass through the current environment when building it. But it cannot once build multiple deployment, and it will become complex when using in local development environment.
1. 使用平台管理配置，在启动时将当前环境的配置通过环境变量传入，这是比较优雅的方式，但框架对运维的要求会比较高，需要部署平台支持，同时开发环境也有相同痛点。Platform to config，config file will pass throught as environment variables to the current environmentat when started. It is more elegant but the maintenance requirements will be higher and platform support will be needed. Using it in the local development environment also complex.
1. 使用代码管理配置，在代码中添加多个环境的配置，在启动时传入当前环境的参数即可。但无法全局配置，必须修改代码。Code to config，add multiple environment configs in the code and pass throught as environment parameters to the current environmentat when started. But it cannot config globally  unless change the code

我们选择了最后一种配置方案，**配置即代码**，配置的变更也应该经过 review 后才能发布。应用包本身是可以部署在多个环境的，只需要指定运行环境即可。
Last option is preferred,  also call it **Config as code**, changing configuration should be reviewed before release. And the package can be deployed in multiple environment with specified the operation parameter.
### 多环境配置 Multi-environment configuration

框架支持根据环境来加载配置，定义多个环境的配置文件，具体环境请查看[运行环境配置](./env.md)
Support loading and defining of multiple configurations according to the environments.For specific environment, refer to [operating environment configuration](./env.md)
```
config
|- config.default.js
|- config.test.js
|- config.prod.js
|- config.unittest.js
`- config.local.js
```

`config.default.js` 为默认的配置文件，所有环境都会加载这个配置文件，一般也会作为开发环境的默认配置文件。`Config.default.js` as the default configuration file, all environments will load it and also as the default development configuration file.

当指定 env 时会同时加载对应的配置文件，并覆盖默认配置文件的同名配置。如 `prod` 环境会加载 `config.prod.js` 和 `config.default.js` 文件，`config.prod.js` 会覆盖 `config.default.js` 的同名配置。When specify env, the corresponding configuration file is loaded and overrides the configureation with same name. Such as `prod` will load `config.prod.js` and `config.default.js`，`config.prod.js` will overrides `config.default.js` 

### 配置写法 Writing Config

配置文件返回的是一个 object 对象，可以覆盖框架的一些配置，应用也可以将自己业务的配置放到这里方便管理。
Config file returns and object that overrides configs of the framework. And make it easier to deal with the management of service configuration
```js
// 配置 logger 文件的目录，logger 默认配置由框架提供 Config the logger folder, default provided by framework
module.exports = {
  logger: {
    dir: '/home/admin/logs/demoapp',
  },
};
```

配置文件也可以返回一个 function，可以接受 appInfo 参数 Config file also can return a function then join the appInfo's parameters

```js
// 将 logger 目录放到代码目录下 logger file save to application folder
const path = require('path');
module.exports = appInfo => {
  return {
    logger: {
      dir: path.join(appInfo.baseDir, 'logs'),
    },
  };
};
```

内置的 appInfo 有 app Info

appInfo | 说明 |   Info
--- | ---  | ---
pkg | package.json |
name | 应用名，同  pkg.name   |   application name, same as pkg.name
baseDir | 应用代码的目录      |    baseDir of application
HOME | 用户目录，如 admin 账户为 /home/admin     |    user directory, such as admin user located in /home/admin
root | 应用根目录，只有在 local 和 unittest 环境下为 baseDir，其他都为 HOME。  |     application root directory, only local and unittest will be baseDir, others is HOME

`appInfo.root` 是一个优雅的适配，比如在服务器环境我们会使用 `/home/admin/logs` 作为日志目录，而本地开发时又不想污染用户目录，这样的适配就很好解决这个问题。
`appInfo.root` is a elegant configuration. For example it would be a better choice using appInfo.root if the developer want to use the `/home/admin/logs` as logger directory when using in server environment meanwhile don't want to contaminate the user directory when development locally.

### 配置加载顺序 Sequence Loading

应用、插件、框架都可以定义这些配置，而且目录结构都是一致的，但存在优先级（应用 > 框架 > 插件），相对于此运行环境的优先级会更高。
Applications, Frameworks can define configurations, and the directory structure remain consistent, priority (Applications > frameworks > Plugins).

比如在 prod 环境加载一个配置的加载顺序如下，后加载的会覆盖前面的同名配置。
For example, in the prod environment to load a configuration of the loading order is as follows, after the load will overwrite the previous configuration of the same name.

```
-> 插件 Plugins config.default.js
-> 框架 Framework config.default.js
-> 应用 Application config.default.js
-> 插件 Plugins config.prod.js
-> 框架 Framework config.prod.js
-> 应用 Application config.prod.js
```

**注意：插件之间也会有加载顺序，但大致顺序类似，具体逻辑可[查看加载器](../advanced/loader.md)。** 
Note: There will also be loading sequences between plugins, but similar to above, details can be viewed [loader] (../advanced /loader.md).

### 合并规则Merge rule

配置的合并使用 [extend2] 模块进行深度拷贝，[extend2] fork 自 [extend]，处理数组时会存在差异。
Configure the merge using the [extend2] module for deep copy, [extend2] fork from [extend], a little differences when processing the array.

```js
const a = {
  arr: [ 1, 2 ],
};
const b = {
  arr: [ 3 ],
};
extend(true, a, b);
// => { arr: [ 3 ] }
```

根据上面的例子，框架直接覆盖数组而不是进行合并。
As shown above, the framework covers the array directly rather than merge.

## 插件配置Plugin Configuration

在应用中可以通过 `config/plugin.js` 来控制插件的一些选项。
In the application can control some of the plug-in options through the `config / plugin.js` to


### 开启关闭 On/Off

框架有一些内置的插件，通过配置可以开启或关闭插件。插件关闭后，插件内所有的文件都不会被加载。
Some built-in plugins that can be configured to on or off. After the plugin is closed, all files in the plugin will not be loaded.

```js
// 关闭内置的 i18n 插件 Close buil-in plugin i18n
module.exports = {
  i18n: {
    enable: false,
  },
};
```

或可以简单的配置一个布尔值 Or simply assign a Boolean value

```js
module.exports = {
  i18n: false,
};
```

### 引入插件 Require plugins

框架默认内置了企业级应用常用的[一部分插件](https://github.com/eggjs/egg/blob/master/config/plugin.js)。
Defaults to the built-in some common enterprise application plugins (https://github.com/eggjs/egg/blob/master/config/plugin.js).


而应用开发者可以根据业务需求，引入其他插件，只需要指定 `package` 配置。
Specify the `package` configuration if developers require other plugins according to business needed.

```js
// 使用 mysql 插件 mysql plugin
module.exports = {
  mysql: {
    enable: true,
    package: 'egg-mysql',
  },
};
```

`package` 为一个 npm 模块，必须添加依赖到 `pkg.dependencies` 中。框架会在 node_modules 目录中找到这个模块作为插件入口。
`Package`  as an npm module, must be added as dependencies to` pkg.dependencies`. The module as a plugin entry in the node_modules directory so that framework can require them.

```json
{
  "dependencies": {
    "egg-mysql": "^1.0.0"
  }
}
```

**注意：配置的插件即使只在开发期使用，也必须是 dependencies 而不是 devDependencies，否则 `npm i --production` 后会找不到。**
** Note: Configured plugins must be dependencies rather than devDependencies even if they are used only using in the development, otherwise `npm i --production` will not be found. **

也可以指定 path 来替代 package。Also can specify path to replace the package.


```js
const path = require('path');
module.exports = {
  mysql: {
    enable: true,
    path: path.join(__dirname, '../app/plugin/egg-mysql'),
  },
};
```

path 为一个绝对路径，这样应用可以把自己写的插件直接放到应用目录中，如 `app/plugin` 目录。Path for an absolute path, so that the application can write their own plug-ins directly into the application directory, such as `app / plugin`.


## 配置结果Results Configuration

框架在启动时会把合并后的最终配置 dump 到 `run/application_config.json`（worker 进程）和 `run/agent_config.json`（agent 进程）中，可以用来分析问题。
The final configuration of the merge is dumped to the `run / application_config.json` (worker process) and` run / agent_config.json` (agent process) when framework starts. It can be used to analyse the problem.

配置文件中会隐藏一些字段，主要包括两类: Config file will hide some fields, including two categories:


- 如密码、密钥等安全字段，这里可以通过 `config.dump.ignore` 配置，必须是 [Set] 类型，查看[默认配置](https://github.com/eggjs/egg/blob/master/config/config.default.js)。
- such as passwords, keys and other security fields, which can be configured by `config.dump.ignore`, must be [Set] type, see [default configuration] (https://github.com/eggjs/egg/blob/master /config/config.default.js).
- 如函数、Buffer 等类型，`JSON.stringify` 后的内容特别大
- such as function, Buffer and other types, `JSON.stringify` will contains pretty large contents.

[Set]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Set
[extend]: https://github.com/justmoon/node-extend
[extend2]: https://github.com/eggjs/extend2
