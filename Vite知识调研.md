## Vite知识调研

1. Vite和WebPack的区别，各有哪些优点和不足
2. Vite的适用场景，实际使用中存在哪些问题
3. Vite打包为什么可以那么“快”
4. 将现有的项目迁移使用Vite存在哪些问题
5. Vite的落地前景
6. Vite常见的配置项和其他API

---



**1. Vite和WebPack的区别，各有哪些优点和不足**

  现在前端主流的打包工具主要以 Webpack 为代表，但随着项目规模的发展，构建方面的痛点越来越突出，最大的感受就是**太慢了**，一方面项目冷启动时必须递归打包整个项目的依赖树，另一方面 JavaScript 语言本身(解释执行、单线程执行)的限制，导致构建性能遇到瓶颈。

  这时候，基于ES module的 Vite应运而生，由尤雨溪(以下简称尤师)带队开发。

**区别**

  `webpack`构建项目会先**打包**，之后启动本地开发服务器，采用**全部加载**的方案，请求模块时加载模块相应的打包结果。

   `vite`启动项目则选择的不打包的方案，在浏览器请求某个模块时，根据模块进行编译，实现**按需动态编译**。

从而可以跳过打包过程中的分析模块依赖、编译等操作。当然，不是随便就可以跳过打包的，后文会提到一些关于vite跳过打包要处理的问题。

​	**webpack**作为老牌霸主，把工作放在了服务器上，全部编译打包，经过多年的优化，已经十分稳定。缺陷主要是提到的速度慢。

  **vite**是一颗备受瞩目的新星，你可能并没有开始使用或研究它，但你一定耳闻过它。最大的优势是速度快。项目复杂度越大，vite的优势就越明显。它甚至可以比webpack快十倍百倍...  它目前的缺点有：

1. 浏览器兼容性。只能使用在现代浏览器上（支持es2015+）
2. 打包兼容性不稳定。对于CommonJs的模块不完全兼容
3. 开发服务器和产品构建之间的最佳输出和行为存在不一致的情况
4. 生态不及webpack，插件等不够丰富
5. 生产环境下，ESbuild构建对于css的代码分割不够友好

|         |                         **打包过程**                         |                             原理                             |
| ------- | :----------------------------------------------------------: | :----------------------------------------------------------: |
| webpack | 识别入口->逐层识别依赖->分析/转换/编译/输出代码->打包后的代码 | 逐级递归识别依赖，构建依赖图谱->转化AST语法树->处理代码->转换为浏览器可识别的代码 |
| vite    |                              -                               | 基于浏览器原生支持的 ES module，利用浏览器解析 imports，服务器端按需编译返回 |

参考

https://blog.51cto.com/xuedingmaojun/2967713

https://juejin.cn/post/7005731645911203877

**3. Vite打包为什么可以那么“快”**

Vite引以为傲的是开发环境不打包，尤师利用了[浏览器的原生 ES Module 支持](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules)，直接在 html 文件里写诸如这样的代码：

```html
// index.html
<div id="app"></div>
<script type="module">
  import { createApp } from 'vue'
  import Main from './Main.vue'
  createApp(Main).mount('#app')
</script>
复制代码
```

 Vite 会在本地帮你启动一个服务器，当浏览器读取到这个 html 文件之后，会在执行到 import 的时候才去向服务端发送模块的请求，解析成浏览器可以执行的 js 文件返回到浏览器端。也就是说只有在真正使用到这个模块的时，浏览器才会请求并且解析这个模块，最大程度的做到了**按需加载**。

引用Vite 官网上的图，传统的 bundle 模式是这样的：

![传统 bundle](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e1c187722cd9405687c6c0ff40b54b9b~tplv-k3u1fbpfcp-watermark.awebp)

而基于 ESM 的构建模式则是这样的：

![基于 ESM](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/af2907c55cdb4fedadf8e604907ddc57~tplv-k3u1fbpfcp-watermark.awebp)

灰色部分是暂时没有用到的路由，甚至完全不会参与构建过程，即使项目中的路由增加，构建速度也不会变慢。

## 依赖预编译

[依赖预编译原因](https://vitejs.cn/guide/dep-pre-bundling.html)，其实是 Vite 2.0 在为用户启动开发服务器之前，先用 `esbuild` 把检测到的依赖预先构建了一遍。依赖预编译原因   

也许你会疑惑，不是一直说好的 no-bundle 吗，怎么还是走启动时编译这条路线了？尤师这么做当然是有理由的，我们先以导入 `lodash-es` 这个包为例。

当你用 `import { debounce } from 'lodash'` 导入一个命名函数的时候，可能你理想中的场景就是浏览器去下载只包含这个函数的文件。但其实没那么理想，`debounce` 函数的模块内部又依赖了很多其他函数，形成了一个依赖图。

当浏览器请求 `debounce` 的模块时，又会发现内部有 2 个 `import`，再这样延伸下去，这个函数内部竟然带来了 600 次请求，耗时会在 1s 左右。

![lodash 请求依赖链路](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d9273fbf819c430ea0a44677c789cf6b~tplv-k3u1fbpfcp-watermark.awebp)

这当然是不可接受的，于是尤师想了个折中的办法，正好利用 [Esbuild](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fevanw%2Fesbuild) 接近无敌的构建速度，让你在没有感知的情况下在启动的时候预先帮你把 `debounce` 所用到的所有内部模块全部打包成一个传统的 `js bundle`。

`Esbuild` 使用 Go 编写，并且比以 JavaScript 编写的打包器预构建依赖快 10-100 倍。

在 `httpServer.listen` 启动开发服务器之前，会先把这个函数劫持改写，放入依赖预构建的前置步骤，[Vite 启动服务器源码](https://github.com/vitejs/vite/blob/main/packages/vite/src/node/server/index.ts)。

```ts
// server/index.ts
const listen = httpServer.listen.bind(httpServer)
httpServer.listen = (async (port: number, ...args: any[]) => {
  try {
    await container.buildStart({})
    // 这里会进行依赖的预构建
    await runOptimize()
  } catch (e) {
    httpServer.emit('error', e)
    return
  }
  return listen(port, ...args)
}) as any
复制代码
```

而 `runOptimize` 相关的代码则在 [Github optimizer](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fvitejs%2Fvite%2Fblob%2Fmain%2Fpackages%2Fvite%2Fsrc%2Fnode%2Foptimizer%2Findex.ts) 中。

首先会根据本次运行的入口，来扫描其中的依赖：

```ts
let deps: Record<string, string>, missing: Record<string, string>
if (!newDeps) {
  ;({ deps, missing } = await scanImports(config))
}
复制代码
```

`scanImports` 其实就是利用 `Esbuild` 构建时提供的钩子去扫描文件中的依赖，收集到 `deps` 变量里，在扫描到入口文件（比如 `index.html`）中依赖的模块后，形成类似这样的依赖路径数据结构：

```js
{
  "lodash-es": "node_modules/lodash-es"
}
复制代码
```

之后再根据分析出来的依赖，使用 `Esbuild` 把它们提前打包成单文件的 bundle。

```ts
const esbuildService = await ensureService()
await esbuildService.build({
  entryPoints: Object.keys(flatIdDeps),
  bundle: true,
  format: 'esm',
  external: config.optimizeDeps?.exclude,
  logLevel: 'error',
  splitting: true,
  sourcemap: true,
  outdir: cacheDir,
  treeShaking: 'ignore-annotations',
  metafile: esbuildMetaPath,
  define,
  plugins: [esbuildDepPlugin(flatIdDeps, flatIdToExports, config)]
})
复制代码
```

在浏览器请求相关模块时，返回这个预构建好的模块。这样，当浏览器请求 `lodash-es` 中的 `debounce` 模块的时候，就可以保证只发生一次接口请求了。

你可以理解为，这一步和 `Webpack` 所做的构建一样，只不过速度快了几十倍。

在预构建这个步骤中，还会对 `CommonJS` 模块进行分析，方便后面需要统一处理成浏览器可以执行的 `ES Module`。

## 插件机制

很多同学提到 Vite，第一反应就是生态不够成熟，其他构建工具有那么多的第三方插件，提供了各种各样开箱即用的便捷功能，Vite 需要多久才能赶上呢？

Vite 从 preact 的 WMR 中得到了启发，把插件机制做成**兼容 Rollup** 的格式。

于是便有了这个**相亲相爱**的 LOGO：

![Vite Rollup Plugins](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ee1321a80d7841e2b4632e20526ff38c~tplv-k3u1fbpfcp-watermark.awebp)

目前和 vite 兼容或者内置的插件，可以查看[vite-rollup-plugins](https://link.juejin.cn?target=https%3A%2F%2Fvite-rollup-plugins.patak.dev%2F)。

简单的介绍一下 Rollup 插件，其实插件这个东西，就是 Rollup 对外提供一些时机的钩子，还有一些工具方法，让用户去写一些配置代码，以此介入 Rollup 运行的各个时机之中。

比如在打包之前注入某些东西，或者改变某些产物结构，仅此而已。

而 Vite 需要做的就是基于 Rollup 设计的接口进行扩展，在保证 Rollup 插件兼容的可能性的同时，再加入一些 Vite 特有的钩子和属性来扩展。

举个简单的例子，[@rollup/plugin-image](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Frollup%2Fplugins%2Fblob%2Fmaster%2Fpackages%2Fimage%2Fsrc%2Findex.js) 可以把图片模块解析成 base64 格式，它的源码其实很简单：

```ts
export default function image(opts = {}) {
  const options = Object.assign({}, defaults, opts)
  const filter = createFilter(options.include, options.exclude)

  return {
    name: 'image',

    load(id) {
      if (!filter(id)) {
        return null
      }

      const mime = mimeTypes[extname(id)]
      if (!mime) {
        // not an image
        return null
      }

      const isSvg = mime === mimeTypes['.svg']
      const format = isSvg ? 'utf-8' : 'base64'
      const source = readFileSync(id, format).replace(/[\r\n]+/gm, '')
      const dataUri = getDataUri({ format, isSvg, mime, source })
      const code = options.dom
        ? domTemplate({ dataUri })
        : constTemplate({ dataUri })

      return code.trim()
    }
  }
}
复制代码
```

其实就是在 `load` 这个钩子，读取模块时，把图片转换成相应格式的 `data-uri`，所以 Vite 只需要在读取模块的时候，也去兼容执行相关的钩子即可。

虽然 Vite 很多行为和 Rollup 构建不同，但他们内部有很多相似的行为和时机，只要确保 Rollup 插件只使用了这些共有的钩子，就很容易做到插件的通用。

可以参考 [Vite 官网文档 —— 插件部分](https://link.juejin.cn?target=https%3A%2F%2Fcn.vitejs.dev%2Fguide%2Fapi-plugin.html%23rollup-%E6%8F%92%E4%BB%B6%E5%85%BC%E5%AE%B9%E6%80%A7)

> 一般来说，只要一个 Rollup 插件符合以下标准，那么它应该只是作为一个 Vite 插件:
>
> - 没有使用 moduleParsed 钩子。
> - 它在打包钩子和输出钩子之间没有很强的耦合。
> - 如果一个 Rollup 插件只在构建阶段有意义，则在 build.rollupOptions.plugins 下指定即可。

Vite 后面的目标应该也是尽可能和 Rollup 相关的插件生态打通，社区也会一起贡献力量，希望 Vite 的生态越来越好。

## 比较

和 Vite 同时期出现的现代化构建工具还有：

- [Snowpack - The faster frontend build tool](https://link.juejin.cn?target=https%3A%2F%2Fwww.snowpack.dev%2F)
- [preactjs/wmr: 👩‍🚀 The tiny all-in-one development tool for modern web apps.](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fpreactjs%2Fwmr)
- [Web Dev Server: Modern Web](https://link.juejin.cn?target=https%3A%2F%2Fmodern-web.dev%2Fdocs%2Fdev-server%2Foverview%2F)

### Snowpack

Snowpack 和 Vite 比较相似，也是基于 ESM 来实现开发环境模块加载，但是它的构建时却是交给用户自己选择，整体的打包体验显得有点支离破碎。

而 Vite 直接整合了 Rollup，为用户提供了完善、开箱即用的解决方案，并且由于这些集成，也方便扩展更多的高级功能。

### WMR

WMR 则是为 Preact 而生的，如果你在使用 Preact，可以优先考虑使用这个工具。

### @web/dev-server

这个工具并未提供开箱即用的框架支持，也需要手动设置 Rollup 构建配置，不过这个项目里包含的很多工具也可以让 Vite 用户受益。

更具体的比较可以参考[Vite 文档 —— 比较](https://link.juejin.cn?target=https%3A%2F%2Fcn.vitejs.dev%2Fguide%2Fcomparisons.html)

## 总结

Vite 是一个充满魔力的现代化构建工具，尤老师也在各个平台放下狠话，说要替代 Webpack。其实 Webpack 在上个世代也是一个贡献很大的构建工具，只是由于新特性的出现，有了可以解决它的诟病的解决方案。

目前我个人觉得，一些轻型的项目（不需要一些特别奇怪的依赖构建）完全可以开始尝试 Vite，比如：

- 各种框架、库中的展示 demo 项目。
- 轻量级的一些企业项目。

也衷心祝福 Vite 的生态越来越好，共同迎接这个构建的新世代。

不过到那个时候，我可能还会挺怀念从前 Webpack 怀念构建的时候，那几分钟一本正经的摸鱼时刻 😆。

作者：ssh_晨曦时梦见兮
链接：https://juejin.cn/post/6932367804108800007
来源：稀土掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

**4. 将现有的项目迁移使用Vite存在哪些问题**

+ *Svg组件报错*

​	Vite 暂时没有对 svg 组件写法的支持，在默认情况下，下面的写法会导致浏览器报错:

```js
import Up from 'common/imgs/up.svg';

function Home() {
  return {
    <>
      // 省略其他子组件
      <Up className="admin-header-user-up" />
    </>
  }
}

```

解决方案 ： 使用`vite-plugin-react-svg`插件, 将svg添加到Vite的plugins数组中，实现了以组件方式引用 SVG 资源的能力

`import Up from 'common/imgs/up.svg?component';`

+ *大量第三方包报错*

