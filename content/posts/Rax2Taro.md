+++
title = 'Rax2Taro'
date = 2024-04-16T20:06:23+08:00
draft = false # true 草稿不展示 ｜ false 正式稿展示
+++

![截屏2024-03-07 17.14.17.png](https://cdn.nlark.com/yuque/0/2024/png/2019260/1709802874340-5bc3475e-a5db-4eb3-9897-a22f7c4119c5.png#averageHue=%23161a1f&clientId=uaa3214db-c1ef-4&from=ui&id=udcce31af&originHeight=956&originWidth=1716&originalType=binary&ratio=2&rotation=0&showTitle=false&size=761538&status=done&style=shadow&taskId=ue48ac1bd-b909-47f4-9d53-d7c3f224838&title=)
**「Rax2Taro」**：[点击查看 Github 地址](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2FTrade-Offf%2FRax2Taro%3Ftab%3Dreadme-ov-file)

# 一、前置背景

笔者日常使用 Rax 框架开发前端需求。但随着业务扩展，我面临一个头痛的需求：将现有的 Rax 组件适配为 Taro 组件，以实现一些特定商业场景的跨平台功能。
这一需求可以概括为：

- **新功能开发** - 在 Taro 框架中实现，确保多端兼容性。
- **旧功能复用** - 将现有 Rax 组件转换为 Taro，避免重写。

重写组件成本高昂，特别是对于缺乏文档和原开发者不在的旧组件。因此我们需要一种自动化工具，能够轻松地**「一键式」**将 Rax 组件转化为 Taro 组件，减少工作量，加快开发进程。
本篇博客将探讨构建一个 Rax 到 Taro 的编译器，从零开始实现组件级转换。

> 恐惧通常来自未知，你恐惧的不是写一个编译器，你恐惧的是你从来没有写过编译器。

作为前端同学，如果接到这种工作可能会汗流浃背。
但不要担心，只要我们把目标拆解到足够清晰、足够细化，一切困难都是纸老虎。

# 二、编译器

编译器是个宽泛的概念，最初是指将**高级语言**转换为计算机能识别的**汇编/机器语言**的工具。
**个人理解：**编译器本质是个从 A 转换的 B 的翻译工具（如有不妥，请评论区指出 🤝**）**
![](https://cdn.nlark.com/yuque/0/2024/jpeg/2019260/1709279932570-0790c102-b5b1-4db3-bc63-c2034d712f70.jpeg)
**以本文目标为例，编译器的作用如上图**
但是编译器的翻译过程不是简单的翻译，通常涉及到多个步骤（词法、语法、语义分析、中间代码生成等）。详细知识点不赘述了，感兴趣的朋友请翻阅[《编译原理》](https://book.douban.com/subject/3296317/)。

## 01 | Babel：JavaScript 编译器

我们以日常开发中，接触最多的 JavaScript 编译器 [Babel](https://link.juejin.cn/?target=https%3A%2F%2Fbabeljs.io%2F) 为例，了解编译器工作逻辑，这里只介绍 Babel 基本流程。更多详细内容可以看以下 Github 官方文档：[Babel 插件手册](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Fjamiebuilds%2Fbabel-handbook%2Fblob%2Fmaster%2Ftranslations%2Fzh-Hans%2Fplugin-handbook.md)

![](https://cdn.nlark.com/yuque/0/2024/jpeg/2019260/1709284639359-b739b068-4cd4-4383-a8a0-71ac674bf84c.jpeg)
**Babel 工作流程**

## 02 | 基本用法

这里以`const a = 1`转换成`var a = 1`为例，看下 Babel 是如何工作的：

### i. 解析（parse）成抽象语法树 AST

> 抽象语法树（Abstract Syntax Tree，AST），是源代码语法结构的一种抽象表示。它以树状的形式表现编程语言的语法结构，树上的每个节点都表示源代码中的一种结构。

**Babel 提供了 **[**@babel/parser**](https://babeljs.io/docs/babel-parser)** 将代码解析成 AST。**这一步主要做两件事：

- 词法分析：把代码转换为令牌流（tokens flow：解析的中间产物，不用管）
- 语法分析：再把每个 token 转换为 AST 结构

```javascript
const parse = require("@babel/parser").parse;

const ast = parse("const a = 1");
```

### ii. 转换（transform）AST

**Babel 提供了 **[**@babel/traverse**](https://www.oschina.net/action/GoToLink?url=https%3A%2F%2Fbabeljs.io%2Fdocs%2Fen%2Fnext%2Fbabel-traverse)** 对解析后的 AST 进行处理。**
转换接收 AST 并对其遍历，在此过程中对节点进行增删改查，是 Babel 编译器最核心的过程。
`traverse()`能够接收 ast 以及 visitor 两个参数：

- **ast** 是上一步解析得到的抽象语法树。
- **visitor** 提供访问不同节点的能力，当遍历到一个匹配的节点时，能够调用具体方法对节点进行处理。

```javascript
const traverse = require("@babel/traverse").default;
const t = require("@babel/types");

traverse(ast, {
  VariableDeclaration: function (path) {
    if (path.node.kind === "const") {
      path.replaceWith(
        t.variableDeclaration("var", path.node.declarations) //替换成var
      );
    }
    path.skip();
  },
});
```

**Babel 提供了**[**@babel/types**](https://www.oschina.net/action/GoToLink?url=https%3A%2F%2Fbabeljs.io%2Fdocs%2Fen%2Fnext%2Fbabel-types)** 用于定义 AST 节点，在 visitor 里做节点处理的时候用于替换等操作。**
在这个例子中，我们遍历上一步得到的 AST，在匹配到变量声明`VariableDeclaration`时，判断值是否为 const 操作时进行替换成 var。

### iii. 生成（generate）代码

**Babel 提供了 **[**@babel/generator**](https://www.oschina.net/action/GoToLink?url=https%3A%2F%2Fbabeljs.io%2Fdocs%2Fen%2Fnext%2Fbabel-generator)** 将 AST 再还原成代码。**
把转换后的最终 AST 还原为字符串形式的代码，同时创建源码映射（source maps）

```javascript
const generate = require("@babel/generator").default;

let code = generate(ast).code;
```

以上就是 Babel 在编译时的流程，这里涉及到了几个关键的包，总结如下图：
![](https://cdn.nlark.com/yuque/0/2024/jpeg/2019260/1710298160849-026739af-02ff-4709-b12d-597d3d84d0bd.jpeg)
我们接下来写的编译器，就是基于上述介绍的 Babel 包来实现。

# 三、Rax2Taro

除了转换工具，我们还需要了解被转换和生产的对象。
因此需要了解 Rax 和 Taro 框架，比较两者的差异和相似之处，注意转换过程中需要抹平的部分：
[「Rax」](https://rax.alibaba-inc.com/) 是阿里巴巴的的跨端解决方案，它的设计理念与 React 类似，提供了类似的组件化开发体验，能够运行在 Web、Node.js、阿里小程序、Weex 等多个平台。
[「Taro」](https://docs.taro.zone/docs/) 是京东的跨端跨框架解决方案，支持使用 React 语法开发一次，然后将代码编译成不同平台的小程序（微信/百度/支付宝/字节跳动/京东小程序等）和 H5 应用，甚至可以编译成 React Native 应用。

通过阅读对比二者官方文档，寻找到接下来开发的**关键破局点：Rax 和 Taro 均支持组件化开发**
由于 Rax 和 Taro 都受到 React 的影响，并且都使用 JSX 语法，它们的许多基础组件在概念上是类似的，这意味着有一些组件和属性是可以在两者之间直接映射的。

## 01 | 本期目标

从组件化下手，本期编译器能力计划将左侧 Rax 文档中（除 Link 外）的 7 个组件一键转换 Taro 组件

- 在 Taro 中优先寻找平替组件，若能找到则增加转换逻辑抹平差异，实现转换器。
- 如果找不到对应组件就标红，等后续对特例地统一处理。

![截屏2024-03-01 17.41.31.png](https://cdn.nlark.com/yuque/0/2024/png/2019260/1709286113155-c78a2296-e6dd-4543-9931-e649d9bdf20a.png#averageHue=%23fafafa&clientId=ua589d88f-7b83-4&from=ui&id=u10a0935f&originHeight=582&originWidth=772&originalType=binary&ratio=2&rotation=0&showTitle=false&size=333861&status=done&style=shadow&taskId=ued5734eb-be72-481a-aeac-d8c4e42c5b9&title=)

## 02 | 编译器结构设计

![](https://cdn.nlark.com/yuque/0/2024/jpeg/2019260/1709607531157-e185be69-8672-4245-b612-9926bcad5b23.jpeg)
**编译器结构设计**

```markdown
/Rax2Taro
|-- /node_modules # 项目依赖库安装文件夹
|-- /src
| |-- index.js # 主入口文件，协调整个转换过程
| |-- parser.js # 用于解析源代码生成 AST
| |-- generator.js # 用于从修改后的 AST 生成新的源代码
|-- /Transformers # 存放转换逻辑的模块文件夹
| |-- index.js # 整合各种转换规则的主要转换器
| |-- FunctionTransformer.js # 函数转换逻辑
| |-- /JSXElementsTransformer # 存放 JSX 元素的特定转换器
| |-- index.js # 整合 JSX 元素转换规则
| |-- ... # 其他 JSX 元素转换模块
| |-- ... # 其他转换逻辑模块
|-- /Input # 存放待转化的 Rax.js 的文件夹
|-- /Output # 存放转化后的 Taro.js 的文件夹
|-- package.json # 定义项目依赖和脚本
|-- README.md # 项目说明文档
```

**编译器代码结构设计**

## 03 | 转换 View 组件

> 「 写一个编译器可能很难，但是转换一个小组件很简单 」

我们的目标是转换 N 个基本组件。
一旦我们知道了如何转换 View 组件，我们只需重复相同的步骤六次即可完成目标。因此，我们将以 View 组件为案例，探讨如何制定单个组件的转换规则。

**「转换，本质就是找不同，并让不同变相同」**
![image.png](https://cdn.nlark.com/yuque/0/2024/png/2019260/1709605733033-93c9c6f4-a997-4ce0-bcf2-c73081f9128c.png#averageHue=%23444654&clientId=u9be0d808-8907-4&from=ui&id=VNbaZ&originHeight=772&originWidth=1338&originalType=binary&ratio=2&rotation=0&showTitle=false&size=150216&status=done&style=shadow&taskId=u3c121db2-7f09-4a08-9a0a-d623e6c1275&title=)
**Rax**
![image.png](https://cdn.nlark.com/yuque/0/2024/png/2019260/1709605733180-db0900c7-5ecb-48e6-8a24-aa6d5fe05d8f.png#averageHue=%23454855&clientId=u9be0d808-8907-4&from=ui&id=PBj7w&originHeight=724&originWidth=1322&originalType=binary&ratio=2&rotation=0&showTitle=false&size=139284&status=done&style=shadow&taskId=ub19fe133-0dfb-4270-b751-968870b7538&title=)
**Taro**
通过对比二者代码之间的差异，我们发现需要做这三件事：

1. Rax 需要引入 `createElement` 不然会报错，Taro 除了组件外没其他引入行为；
2. Import 引入写法不同，Rax 用单文件引入单组件，Taro 是在 @tarojs/components 集中引入；
3. 同样都是<View />组件，两个框架间的组件 API 属性可能不同，或者属性名相同功能不同。需要抹平差异，或者特异处理；

接下来让我们一步步实现。读取文件和转换 SourceCode 得到 AST 部分不讲，具体细节可以在 Github 项目里看，就两行代码。**下述一切操作均默认为转换器获取到 AST 之后**。

### i. 删除 createElement

```javascript
const traverse = require("@babel/traverse").default;
const importsTransformer = require("./ImportsTransformer");

function transform(ast) {
  traverse(ast, {
    ImportDeclaration(path) {
      importsTransformer.transformImportDeclaration(path);
    },
    // ... 添加其他节点类型的转换规则
  });
}

module.exports = {
  transform,
};
```

开始之前，首先了解一下转换器主入口结构。
引入了`@babel/traverse`，引入我们之后用来删除 createElement 的方法。
其中使用了`traverse()`函数，它用于遍历抽象语法树（AST），允许访问树中的每个节点，并对这些节点进行修改、添加或删除。

```javascript
function transformImportDeclaration(path) {
  const importSource = path.node.source.value;

  // 删除 "rax" 模块里的 createElement
  if (importSource === "rax") {
    // 过滤 createElement 引入
    path.node.specifiers = path.node.specifiers.filter(
      (specifier) =>
        !(
          t.isImportSpecifier(specifier) &&
          specifier.imported.name === "createElement"
        )
    );
    // 删除空引用
    if (path.node.specifiers.length === 0) {
      path.remove();
    }
  }
}
```

接下来看功能函数，因为会遍历节点，所以

1. 我们首先通过`path.node.source.value=== "rax"`定位，找到我们要增删改的目标节点；
2. 然后过滤这个语句中的所有导入说明符`specifiers` ，检查值是否为 createElement
   - `path.node.specifiers` 是 AST 中的一个部分，表示一个模块导入语句中的所有导入说明符。
   - `t.isImportSpecifier` 是 Babel 类型检查器的一部分。
3. 如果是，就被过滤掉。
4. 最后加一步空引用清除，删除`import { } from 'rax'`

### ii. 改变组件引入写法

引入写法修改类似上面处理方法：

1. 先定位找到 `import View from "rax-view"`，删掉这句引入
2. 声明一个对象，存 Taro 引入组件，把上面删掉的组件再以 `import { View } from "@tarojs/components"`的形式声明

但是考虑到可扩展性，之后还会遇到 Text、Image 等组件，所以这里我写了一张映射表，批量重复上面操作。

```javascript
const componentImportMap = {
  "rax-view": {
    source: "@tarojs/components",
    importName: "View",
  },
  "rax-text": {
    source: "@tarojs/components",
    importName: "Text",
  },
  // ... 添加更多组件及其转换规则
};

const taroComponentsToImport = new Set(); // 声明去重 Taro 对象

// 基础组件映射转换
function transformImportDeclaration(path) {
  const importSource = path.node.source.value;
  // ... createElement 删除逻辑

  const newImportInfo = componentImportMap[importSource];
  if (newImportInfo) {
    // 如果映射表里有，就存这个值到 Taro 对象中
    taroComponentsToImport.add(newImportInfo.importName);
    path.remove(); // 并移除原 rax-xxx 导入声明
  }
}
```

由于功能类似，所以这两个功能我都写在 `transformImportDeclaration` 函数里了。

### iii. 转换 View 组件

转换 View 组件听着挺复杂，其实拆解下，本质就是把两套属性差异抹平，用的也是上面对 AST 节点的增删改操作。
![截屏2024-03-05 13.39.21.png](https://cdn.nlark.com/yuque/0/2024/png/2019260/1709617208202-1c61e84f-d576-4ded-bf1b-726b8991fc0b.png#averageHue=%23fafafa&clientId=u9be0d808-8907-4&from=ui&height=1048&id=ub3fd1ad7&originHeight=1054&originWidth=1744&originalType=binary&ratio=2&rotation=0&showTitle=false&size=812228&status=done&style=shadow&taskId=u56f8cf83-31c4-4d11-a128-08f8260dd2a&title=&width=1734)
![截屏2024-03-05 13.39.37.png](https://cdn.nlark.com/yuque/0/2024/png/2019260/1709617214969-8aea45a7-9128-438b-b581-4a26771de03b.png#averageHue=%23fafafa&clientId=u9be0d808-8907-4&from=ui&height=1818&id=u378ab4af&originHeight=1828&originWidth=2168&originalType=binary&ratio=2&rotation=0&showTitle=false&size=1887461&status=done&style=shadow&taskId=ub954756d-606c-4219-bbb6-c33c6e78349&title=&width=2156)
从二者官方文档里展示的组件 API 可以明显看到 Rax 比 Taro 的 View 少了很多属性，但由于我们实现的是 **Rax -> Taro **的单向转换，所以**「一切编译行为以 Rax API 为转换基准」**。
至于 Taro 多出来的 API ，编译器不用管，如果后续开发需要用到 Taro 属性，则开发者根据 Taro 官方文档自行配置使用即可(反正 Rax 没有 🐒)

对比 View API 差异，我整理出下面的表格内容：

- 红色背景：API 不同，需编译转换。
- 黄色背景：API 相同，无需编译转换。**但功能有差异，使用过程中要注意**
- 灰色背景：无法兼容，编译过程中删除；
- 无背景：API 相同，无需编译转换。
  | **Rax 属性** | **Rax 描述** | **Taro 属性** | **Taro 描述** |
  | --- | --- | --- | --- |
  | className | 自定义样式名 | className | 自定义样式名 |
  | style | 自定义样式 | style | 自定义样式 |
  | **onClick** | 点击 | **onTap** | 点击。 |
  | **onLongpress** | 长按 | **onLongTap** | 长按 500ms 之后触发，触发了长按事件后进行移动将不会触发屏幕的滚动 |
  | **onAppear** | 当前元素可见时触发 | **onAppear** | 当前元素可见面积超过 50%时触发 |
  | **onDisappear** | 当前元素从可见变为不可见时触发 | **onDisappear** | 当前元素不可见面积超过 50%时触发 |
  | **onFirstAppear** | 当前元素首次可见时触发 | **onFirstAppear** | 当前元素首次可见面积达到 50%时触发 |
  | onTouchStart | 触摸动作开始 | onTouchStart | 触摸动作开始。 |
  | onTouchMove | 触摸后移动 | onTouchMove | 触摸后移动。 |
  | onTouchEnd | 触摸动作结束 | onTouchEnd | 触摸动作结束。 |
  | onTouchCancel | 触摸动作被打断，如来电提醒，弹窗 | onTouchCancel | 触摸动作被打断，如来电提醒，弹窗 |

可以看到有 5 条 API 属性有不同，其中 3 条是使用方法不同，2 条是需要编译器处理的属性。

- onClick -> onTap
- onLongpress -> onLongTap

View 转换器的逻辑如下：

```javascript
const t = require("@babel/types");

function transformViewElement(path) {
  // 确保我们只处理具有 name 属性的 JSXElement
  if (path.node.openingElement && path.node.openingElement.name) {
    const openingElementName = path.node.openingElement.name;

    if (
      t.isJSXIdentifier(openingElementName) &&
      openingElementName.name === "View"
    ) {
      path.node.openingElement.attributes.forEach((attribute) => {
        if (t.isJSXAttribute(attribute) && attribute.name) {
          const attributeName = attribute.name.name;

          switch (attributeName) {
            case "onClick":
              attribute.name.name = "onTap";
              break;
            case "onLongpress":
              attribute.name.name = "onLongTap";
              break;
          }
        }
      });
    }
  }
}

module.exports = {
  transformViewElement,
};
```

跟之前操作差不多，还是遍历找节点，找到要处理的 API 属性，将属性重命名。
至此，我们实现了 Rax -> Taro View 组件的编译转换。接下来需要对剩下的 6 个基础组件做同样操作，即可完成本期目标：
重复流程如下：

1. 列表格，整理 API 差异，标明处理方式
2. 遍历 AST 找对应节点、找到需要处理的 API 属性
3. 执行重命名 or 删除动作

机械性动作 \* n ...

为了保证转换器的拓展性，这里新增了一个主入口 index.js 文件用来批量管理各个独立组建的小转换器。
![截屏2024-03-05 13.53.57.png](https://cdn.nlark.com/yuque/0/2024/png/2019260/1709618204459-ba30c46b-a2e4-47be-8c6d-c433d60711a5.png#averageHue=%23282732&clientId=u9be0d808-8907-4&from=ui&id=u490fd858&originHeight=936&originWidth=2064&originalType=binary&ratio=2&rotation=0&showTitle=false&size=1072219&status=done&style=shadow&taskId=uc7cf71ab-a36d-4a84-9944-a89f95080ad&title=)

其他组件的对应表请在语雀文档查看：
[https://aliyuque.antfin.com/ljl353638/hy9yg3/xpgtgg3c8cv2wntt](https://aliyuque.antfin.com/ljl353638/hy9yg3/xpgtgg3c8cv2wntt)

# 四、自动化测试

由于在本地转换各个 API 需要反复调试，并且需要实时查看组件编译后的情况。为了开发提效，我本地还需要运行 Rax 和 Taro 两个用脚手架生成的项目，加一个小自动化测试脚本进行一键编译调试。

具体步骤如下，在`e2e.test.js` 中：

- 设定 Rax 项目路径：本项目从 Rax 应用的源文件夹`rax-test-demo/src/index.js`读取待转换组件代码。
- 设定 Taro 项目路径：将转换后的 Taro 组件代码写入 Taro 应用的目标文件夹 `TaroTestDemo/src/app.js`
- 转换代码：命令行执行：`npm run test:e2e` 转换器将源代码解析为 AST，并进行转换。
- 监测转换结果：在 Taro 测试环境中检查转换后的代码，确保没有报错且符合预期。

```javascript
const fs = require("fs");
const path = require("path");
const { transform } = require("../src/Transformers");
const parser = require("@babel/parser");
const generator = require("@babel/generator").default;

// 基于你 Rax 源文件和 Taro 输出的路径
const raxSourcePath = path.join(__dirname, "../../rax-test-demo/src/index.js");
const taroOutputPath = path.join(
  __dirname,
  "../../TaroTestDemo/src/pages/index/index.jsx"
);

describe("End-to-End Transformation", () => {
  it("从Rax组件中读取源码，转换为Taro组件", () => {
    const raxSourceCode = fs.readFileSync(raxSourcePath, "utf8");

    // 1.解析 Rax 源代码为 AST
    const raxAst = parser.parse(raxSourceCode, {
      sourceType: "module",
      plugins: ["jsx"],
    });
    // 2.转换 AST
    transform(raxAst);
    // 3.生成转换后的 Taro 源代码
    const taroOutput = generator(raxAst, {});

    fs.writeFileSync(taroOutputPath, taroOutput.code);
  });
});
```

## 01 | 准备 Rax 测试环境

```bash
npm install -g rax-cli # 安装 Rax 脚手架（如果尚未安装）

cd Desktop # 进入桌面
rax init RaxTestDemo # 初始化 Rax 项目

# 进入文件夹
cd rax-test-demo
# 安装依赖
npm install

# 运行
npm start
```

脚手架选项如下：
![截屏2024-03-05 14.06.44.png](https://cdn.nlark.com/yuque/0/2024/png/2019260/1709618822132-2be7baba-1ace-48f6-a01e-11c2b6edc111.png#averageHue=%23111111&clientId=u9be0d808-8907-4&from=ui&id=u4cb01bbd&originHeight=294&originWidth=1626&originalType=binary&ratio=2&rotation=0&showTitle=false&size=317538&status=done&style=shadow&taskId=ueead23ea-f8a0-4fa9-a515-6a904a277a9&title=)

## 02 | 准备 Taro 测试环境

```bash
npm install -g @tarojs/cli # 安装 Taro 脚手架（如果尚未安装）

cd Desktop # 进入桌面
taro init TaroTestDemo # 初始化 Taro 项目

# 进入文件夹
cd TaroTestDemo
# 安装依赖
npm install

# 运行
npm run dev:h5
```

脚手架选项如下：
![截屏2024-03-05 14.07.56.png](https://cdn.nlark.com/yuque/0/2024/png/2019260/1709618893196-d72cf860-2e63-4285-b139-08a0f225fb6f.png#averageHue=%230a0a0a&clientId=u9be0d808-8907-4&from=ui&id=u317a7a99&originHeight=684&originWidth=1588&originalType=binary&ratio=2&rotation=0&showTitle=false&size=591533&status=done&style=shadow&taskId=ub70b2783-1e89-4f5f-a2ca-02e55a04ac9&title=)
确保遵循上述步骤来准备 Rax 和 Taro 的测试环境。在双方都构建完成后，您可以执行 Jest 测试来验证转换过程。

## 03 | 执行自动化测试

![截屏2024-03-05 14.09.27.png](https://cdn.nlark.com/yuque/0/2024/png/2019260/1709618988413-80510d7b-ac96-4e29-879b-ff07bcb568e4.png#averageHue=%23272635&clientId=u9be0d808-8907-4&from=ui&height=286&id=u215c197d&originHeight=556&originWidth=972&originalType=binary&ratio=2&rotation=0&showTitle=false&size=276700&status=done&style=shadow&taskId=uf79f1139-e2b3-454f-955f-cd780cb7425&title=&width=500)

1. 安装 [Jest](https://jestjs.io/) 命令行执行：`npm install --save-dev jest`
2. package.json 配置：`"test:e2e": "jest tests/e2e.test.js"`
3. 命令行执行：`npm run test:e2e`

此时，你就可以在本地同时运行 Rax 与 Taro 项目，一边写 Rax 一边可实时通过此条命令进行编译转换 Taro。

# 五、总结

本篇文章从零开始构造了一个略具复杂度的 Rax 转 Taro 编译器。
初始目标挺吓人，但经过合理拆解发现大目标也不过只是走通 MVP（最小可行产品）后的重复累加。工作如此，生活亦如此。专注你的目标，不要被纷繁的信息流影响，脚踏实地一步步完成你的小 Step，一切总能完成的。

后续对这个编译器，我计划如下内容，欢迎持续关注：

- [ ] 新增自定义脚手架功能
- [ ] 抹平转换过程中的 CSS 样式差异
- [ ] 新增 README_EN 完善中英文使用文档

写作不易，如果觉得本文对你有启发有帮助的话，请在 [GitHub](https://github.com/Trade-Offf/Rax2Taro?tab=readme-ov-file) 帮我点个 Star ⭐
交个朋友，比心感谢 ❤ ~~~