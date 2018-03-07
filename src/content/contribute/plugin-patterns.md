---
title: 插件模式
sort: 5
---

在 webpack 构建系统中，能够通过插件进行定制，这赋予了无限的可能性。这使你可以创建自定义资源类型(asset type)，执行唯一的构建修改(build modification)，甚至可以使用中间件来增强 webpack 运行时。下面是在编写插件时非常有用一些 webpack 的功能。

## 检索遍历资源(asset)、chunk、模块和依赖

在执行完成编译的封存阶段(seal)之后，编译(compilation)的所有结构都可以遍历。

```javascript
function MyPlugin() {}

MyPlugin.prototype.apply = function(compiler) {
  compiler.plugin('emit', function(compilation, callback) {

    // 检索每个（构建输出的）chunk：
    compilation.chunks.forEach(function(chunk) {
      // 检索 chunk 中（内置输入的）的每个模块：
      chunk.modules.forEach(function(module) {
        // 检索模块中包含的每个源文件路径：
        module.fileDependencies.forEach(function(filepath) {
          // 我们现在已经对源结构有不少了解……
        });
      });

      // 检索由 chunk 生成的每个资源(asset)文件名：
      chunk.files.forEach(function(filename) {
        // Get the asset source for each file generated by the chunk:
        var source = compilation.assets[filename].source();
      });
    });

    callback();
  });
};

module.exports = MyPlugin;
```

- `compilation.modules`：编译后的（内置输入的）模块数组。每个模块管理控制来源代码库(source library)中的原始文件(raw file)的构建。
- `module.fileDependencies`：模块中引入的源文件路径构成的数组。这包括源 JavaScript 文件本身（例如：index.js）以及它所需的所有依赖资源文件（样式表、图像等）。审查依赖，可以用于查看一个模块有哪些从属的源文件。
- `compilation.chunks`：编译后的（构建输出的）chunk 数组。每个 chunk 所管理控制的最终渲染资源的组合。
- `chunk.modules`：chunk 中引入的模块构成的数组。通过扩展(extension)可以审查每个模块的依赖，来查看哪些原始源文件被注入到 chunk 中。
- `chunk.files`：chunk 生成的输出文件名构成的数组。你可以从 `compilation.assets` 表中访问这些资源来源。

### 监听观察图(watch graph)的修改

运行 webpack 中间件时，每个编译包括一个 `fileDependencies` 数组（正在观察哪些文件）和一个 `fileTimestamps` 哈希，它将被观察的文件路径映射到时间戳。这可以用于检测编译中哪些文件已经修改：

```javascript
function MyPlugin() {
  this.startTime = Date.now();
  this.prevTimestamps = {};
}

MyPlugin.prototype.apply = function(compiler) {
  compiler.plugin('emit', function(compilation, callback) {

    var changedFiles = Object.keys(compilation.fileTimestamps).filter(function(watchfile) {
      return (this.prevTimestamps[watchfile] || this.startTime) < (compilation.fileTimestamps[watchfile] || Infinity);
    }.bind(this));

    this.prevTimestamps = compilation.fileTimestamps;
    callback();
  }.bind(this));
};

module.exports = MyPlugin;
```

还可以将新文件路径添加到观察图(watch graph)中，以在这些文件修改时，接收消息和重新触发编译。只要将有效的文件路径推送到 `compilation.fileDependencies` 数组中，就可以将其添加到观察图中。注意：`fileDependencies` 数组在每次编译时都会重新构建，因此你的插件必须将自己要观察的依赖都推送到每次编译中，以使它们都处于观察中。

## 监听 chunk 的修改

类似于观察图(watch graph)，监听 chunk（或者模块，就当前情况而言）的修改也很简单，通过在编译时跟踪它们的 hash 来实现。

```javascript
function MyPlugin() {
  this.chunkVersions = {};
}

MyPlugin.prototype.apply = function(compiler) {
  compiler.plugin('emit', function(compilation, callback) {

    var changedChunks = compilation.chunks.filter(function(chunk) {
      var oldVersion = this.chunkVersions[chunk.name];
      this.chunkVersions[chunk.name] = chunk.hash;
      return chunk.hash !== oldVersion;
    }.bind(this));

    callback();
  }.bind(this));
};

module.exports = MyPlugin;
```

***

> 原文：https://webpack.js.org/contribute/plugin-patterns/