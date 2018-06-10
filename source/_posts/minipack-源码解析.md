---
title: minipack 源码解析
date: 2018-06-09 17:30:33
tags: 源码解析
---

# minipack 是什么 

>  A simplified example of a modern module bundler written in JavaScript

正如 [github](https://github.com/ronami/minipack) 介绍，这是一个用 JavaScript 编写的现代模块构建工具的简化示例。

作为一名 FE, 平时可能会使用 Webpack/Browserify/Rollup/Parcel 等构建工具。了解这些构建工具的工作原理可以帮助我们更好地决定编写代码的方式，所以了解他们的工作原理还是很有必要的。

本文将围绕 minipack 的源码来分析如何实现一个 module bundlers.

<!-- more -->

## module bundlers

module bundlers 将小块代码编译成更大和更复杂的代码，让其运行在 Web 浏览器中。 它拥有入口文件的概念，我们让 module bundlers 知道哪个文件是我们应用程序的入口文件，然后让 module bundlers 从该文件开始，并去尝试理解它依赖哪些文件，然后它会尝试了解这些文件的依赖关系，直到它发现应用程序中的每个模块，以及它们如何相互依赖，最后构成 dependenciesGraph(依赖图).

## 源码解析

代码主要有三个重要方法

### createAsset

```javascript
// 该方法接受文件路径, 读取内容并提取它的依赖关系
function createAsset(filename) {
  // 以字符串形式读取文件的内容
  const content = fs.readFileSync(filename, 'utf-8');

  // 使用 babylon 转换我们的原始代码，转换之后生成抽象语法树(AST)
  // babylon 的介绍可以参考 https://github.com/jamiebuilds/babel-handbook/blob/master/translations/zh-Hans/plugin-handbook.md#toc-babylon
  // 如果对 AST 不太了解，可以通过 https://astexplorer.net/ 查看生成的 AST 代码是怎样的
  const ast = babylon.parse(content, {
    sourceType: 'module',
  });

  // 当我们解析完以后，我们就可以提取当前文件中的 dependencies
  // dependencies 翻译为依赖，也就是我们文件中所有的 `import xxxx from xxxx`
  // 我们将这些依赖都放在 dependencies 的数组里面，这个数组将保存这个模块依赖的模块的相对路径
  const dependencies = [];

  // 使用 babel-traverse 遍历 AST，babel-traverse 的介绍可以参考 https://github.com/jamiebuilds/babel-handbook/blob/master/translations/zh-Hans/plugin-handbook.md#toc-babel-traverse
  traverse(ast, {
    // 当遍历到类型为 `ImportDeclaration` 的 AST 节点，其实就是我们的 `import xxx from xxx.js`
    ImportDeclaration: ({node}) => {
      // 将匹配到的模块路径 push 到 dependencies
      dependencies.push(node.source.value);
    },
  });

  // 递增简单计数器为此模块分配唯一标识符
  const id = ID++;

  // es6 module 以及 es6 语法无法兼容所有浏览器
  // 为了确保 bundle 能在所有的浏览器种运行，使用 babel 进行编译，转换成 CommonJS 代码
  // 使用 babel-preset-env 确定需要支持的浏览器
  const {code} = transformFromAst(ast, null, {
    presets: ['env'],
  });

  // 最后将模块导出
  return {
    id,
    filename,
    dependencies,
    code,
  };
}
```

### createGraph

```javascript
// 该方法接收入口文件的文件路径，获取整个应用程序的每个模块的依赖关系，可以把它抽象理解为依赖图
function createGraph(entry) {
  // 首先解析入口文件，获取入口模块的信息
  const mainAsset = createAsset(entry);

  const queue = [mainAsset];

  // 使用一个`for ... of`循环遍历 queue
  for (const asset of queue) {
	// 子模块的依赖关系
    asset.mapping = {};

    const dirname = path.dirname(asset.filename);

    // 遍历依赖的模块
    asset.dependencies.forEach(relativePath => {
      // 转换成绝对路径
      const absolutePath = path.join(dirname, relativePath);

      // 解析依赖
      const child = createAsset(absolutePath);

      // 供后期 require ${id} 使用
      asset.mapping[relativePath] = child.id;

      // 将解析的依赖 push 到 queue 中
      queue.push(child);
    });
  }

  // 最后将 queue 返回，包含了目标应用中每个 module 的信息
  return queue;
}
```

### bundle

```javascript
// 该方法将 graph 加工成浏览器中可执行代码
function bundle(graph) {
  let modules = '';

  // 在我们到达该函数的主体之前,我们将构建一个作为该函数的参数的对象
  // 请注意, 我们构建的这个字符串被两个花括号 ({}) 包裹, 因此对于每个模块,
  // 我们添加一个这种格式的字符串: `key: value,`
  graph.forEach(mod => {
    //  图表中的每个模块在这个对象中都有一个`entry`. 我们使用`模块的id`作为`key`和一个数组作为`value` (用数组因为我们在每个模块中有2个值)
    // 第一个值是用函数包装的每个模块的代码. 这是因为模块应该被 限定范围: 在一个模块中定义变量不会影响 其他模块 或 全局范围
    // 我们的模块在我们将它们`转换{被 babel 转译}`后, 使用`commonjs`模块系统: 他们期望一个`require`, 一个`module`和`exports`对象可用. 那些在浏览器中通常不可用,所以我们将它们实现并将它们注入到函数包装中
    // 对于第二个值,我们用`stringify`解析模块及其依赖之间的关系(也就是上文的asset.mapping). 解析后的对象看起来像这样: `{'./relative/path': 1}`
    // 这是因为我们模块的被转换后会通过相对路径来调用`require()`. 当调用这个函数时,我们应该能够知道依赖图中的哪个模块对应于该模块的相对路径
    modules += `${mod.id}: [
      function (require, module, exports) { ${mod.code} },
      ${JSON.stringify(mod.mapping)},
    ],`;
  });

  // 这一段代码实际上才是模块引入的核心逻辑
  // 我们制造一个顶层的 require 函数，这个函数接收一个 id 作为值，并且返回一个全新的 module 对象
  // 我们倒入我们刚刚制作好的模块，给他加上 {}，使其成为 {1:[...],2:[...]} 这样一个完整的形式
  // 然后塞入我们的立即执行函数中(function(modules) {...})()
  // 在 (function(modules) {...})() 中，我们先调用 require(0)
  // 理由很简单，因为我们的主模块永远是排在第一位的
  // 紧接着，在我们的 require 函数中，我们拿到外部传进来的 modules，利用我们一直在说的全局数字 id 获取我们的模块
  // 每个模块获取出来的就是一个二维元组
  // 然后，我们要制造一个 `子require`
  // 这么做的原因是我们在文件中使用 require 时，我们一般 require 的是地址，而顶层的 require 函数参数时 id
  // 不要担心，我们之前的 idMapping 在这里就用上了，通过用户 require 进来的地址，在 idMapping 中找到 id
  // 然后递归调用 require(id)，就能够实现模块的自动倒入了
  // 接下来制造一个 const newModule = {exports: {}};
  // 运行我们的函数 fn(childRequire, newModule, newModule.exports);，将应该丢进去的丢进去
  // 最后 return newModule.exports 这个模块的 exports 对象
  const result = `
    (function(modules) {
      function require(id) {
        const [fn, mapping] = modules[id];

        function localRequire(name) {
          return require(mapping[name]);
        }

        const module = { exports : {} };

        fn(localRequire, module, module.exports);

        return module.exports;
      }

      require(0);
    })({${modules}})
  `;

  return result;
}
```

## 总结

回过头来看，思路还是比较清晰的，主要分为两步：

1. 通过 babel 以及 babel 插件，分析、收集依赖，得到整个程序的依赖图
2. 实现一个 loader, 用于 require module

这是一个简化的构建工具设计流程，例子中还有一些问题尚未处理，比如只支持 es6 模块收集，无法兼容 CommonJS/CMD/AMD 模块; 模块引用路径必须写后缀名等问题。需要优化的地方还有很多，感兴趣的同学可以 fork 该项目完善。

## 参考链接

[minipack-explain](https://github.com/chinanf-boy/minipack-explain/blob/master/src/minipack.js)