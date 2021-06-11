
# Webpack性能优化

### 对于webpack，人们通常一年只接触两次，剩下的时间就“只管用”了。

## webpack是什么？
    webpack是一个用于现代JavaScript应用程序的静态模块打包工具。当 webpack处理应用程序时，它会在内部构建一个依赖图(dependency graph)，此依赖图对应映射到项目所需的每个模块，并生成一个或多个bundle。

![image](https://gimg2.baidu.com/image_search/src=http%3A%2F%2Fwww.jqhtml.com%2Fwp-content%2Fuploads%2F2017%2F06%2Fwebpack613-1.png&refer=http%3A%2F%2Fwww.jqhtml.com&app=2002&size=f9999,10000&q=a80&n=0&g=0n&fmt=jpeg?sec=1624496570&t=e0692d9c377cf7e0f794e4159376f843)

## 开发服务器，devServer
**问题**：每次改动代码之后，都需要重新打包，因为运行项目加载的包里面并没有当前改动的代码，这样每次改动，每次打包，依次循环，会重复打包。

**作用**：用于自动化-自动编译、自动打包、自动刷新浏览器等。

**特点**：只会在内存中编译打包，不会有任何输出。

**启动指令**：npm webpack-dev-server
会监视src下面源代码的变化，自动进行编译，因此我们修改配置文件里面代码的时候，需要终止正在运行的项目，重新编译配置：
```javaScript
devServer: {
  contentBase: resolve(__dirname, 'build'); // 运行项目的目录，即构建后的目录
  compress: true, // 启动gzip压缩
  port: 3000,
  open: true,
}
```

## webpack性能优化
* 开发环境性能优化
* 生产环境性能优化

### 开发环境性能优化
* 优化打包构建速度
  * HMR
* 优化代码调试
  * source-map

### 生产环境性能优化
* 优化打包构建速度
  * oneOf
  * babel缓存
  * 多进程打包
  * externals
  * dll
* 优化代码运行的性能
  * 缓存(hash-chunkhash-contenthash)
  * tree shaking
  * code split
  * 懒加载/预加载
  * pwa

## 开发环境优化打包构建速度---HMR（Hot Module Replacement）热模块更新
**问题**：当我们修改css文件样式的时候，js文件也会重新打包。若有100个模块，100个样式文件，只要修改一个文件，另外的所有文件都会重新打包，速度将非常慢。如果一个模块修改，只打包该模块？

**作用**：一个模块发生变化，只会重新打包该模块极其涉及的模块，而不是所有模块。 =》 极大提升构建速度。

**使用**：
  * html文件默认没有HMR功能，但是他恰好不需要，因为只有一个html文件。
  * js文件默认没有HMR功能，需要手动配置。注：HMR功能对js的处理，只能处理非入口js文件；更新main.js文件会刷新整个网页。
  * 样式模块可以使用是因为sty-loader内部实现了，因此在开发环境中使用style-loader，打包速度更快；
```javaScript
devServer: {
  hot: true,
}

// main.js文件中这样配置
if (module.hot) {
  module.hot.accept('./test.js', function() {
    a();
  })
}
```

## 生产环境优化打包构建速度---oneOf
**问题**：在一个配置文件中，rules里面非常多的loader规则，会有处理图片的url-loader，处理css、less的style-loader/less-loader等，正常来讲，每个不同类型的文件在loader转换时，都要将module里面rules中的所有loader遍历一遍，如果符合，就被对应loader处理，不符合则直接过。

**作用**：使用oneOf根据文件类型加载对应的loader，只要能匹配一个即可退出。

=>提升构建速度，避免每个文件都被所有loader过一遍，因为任何一个文件，构建过程中，在遇到第一个与之对应的loader后，不会再往下进行。

**注意事项**：对于同一类型文件，比如处理js，如果需要多个loader，可以单独抽离js处理，确保oneOf里面一个文件类型对应一个loader。可以配置 enforce: 'pre'，指定优先执行。使用OneOf的时候，因为一个文件只能被一个loader处理。那么当一个文件要被多个loader处理，一定要指定loader执行的先后顺序：如下述代码中先执行eslint 在执行babel。

**代码配置**：
```javascript
module: {
  rules: [
    {
      test: /\.js$/,
      exclude: /node_modules/,
      enforce: 'pre',
      loader: 'eslint-loader',
      options: {
        fix: true
      }
    },
    {
      oneOf: [
        {
          test: /\.js$/,
          exclude: /node_modules/,
          loader: 'babel-loader',
          options: {
            presets: [
              [
                '@babel/preset-env',
                {
                  useBuiltIns: 'usage',
                  corejs: {version: 3},
                  targets: {
                    chrome: '60',
                    firefox: '50'
                  }
                }
              ]
            ]
          }
        },
        {
          test: /\.(jpg|png|gif)/,
          loader: 'url-loader',
          options: {
            limit: 8 * 1024,
            name: '[hash:10].[ext]',
            outputPath: 'imgs',
            esModule: false
          }
        },
        {
          test: /\.html$/,
          loader: 'html-loader'
        },
      ]
    }
  ]
},
```

## 生产环境优化打包构建速度---babel缓存
**问题**：在编译过程中，比如有100个js模块，当有一个改动的时候，另外99个模块应该保持不变，不再重新编译。与HMR有点类似，但是呢，在生产环境下不能使用HMR，因为HMR是基于devServer的，而生产环境不需要devServer。

**作用**：babel先将之前的100个文件，编译后的文件进行缓存，如果文件没有变化的话，直接使用缓存，而不会重新编译。

=>让第二次打包构建速度更快。

**代码配置**：
```javascript
use: {
  // 不处理node_modules中的js文件，从而优化处理加快速度
  exclude: /node_modules/,
  loader: 'babel-loader',
  options: {
    cacheDirectory: true
  }
}
```

## 生产环境优化代码运行的性能---文件资源缓存
**作用**: 让代码上线运行缓存更好使用。

**hash**: 每次wepack构建时会生成一个唯一的hash值。

问题: 因为js和css同时使用一个hash值。如果重新打包，会导致所有缓存失效。

**chunkhash**：根据chunk生成的hash值。如果打包来源于同一个chunk，那么hash值就一样。

问题: js和css的hash值还是一样的，因为css是在js中被引入的，所以同属于一个chunk。

**contenthash**: 根据文件的内容生成hash值。不同文件hash值一定不一样。   

**代码配置**:
```javascript
  output: {
    filename: 'js/[name].[hash/chunkhash/contenthash:10].js'
  }
```

## webpack 5当中添加了用于长期缓存的新算法。在生产模式下默认启用这些功能。
Webpack 5 针对 moduleId 和 chunkId 的计算方式进行了优化，增加确定性的 moduleId 和 chunkId 的生成策略。moduleId 根据上下文模块路径，chunkId 根据 chunk 内容计算，最后为 moduleId 和 chunkId 生成 3 - 4 位的数字id，实现长期缓存，生产环境下默认开启。
```javascript
chunkIds: "deterministic", moduleIds: "deterministic"
```


## 生产环境优化代码运行的性能---tree shaking
**前提**：
  1. 必须使用ES6模块化？因为tree-shaking是针对静态结构进行分析，只有import和export是静态的导入和导出。而commonjs有动态导入和导出的功能，无法进行静态分析。
  2. 开启production环境？Webpack 只有在压缩代码的时候会 tree-shaking，而这只会发生在生产模式中。

**作用**:去除无用代码，减少代码体积

**example**：
```javascript
// index.js
import { cube } from './test.js'
console.log(cube(5)); // 输出125

// test.js
export function square(x) {
  console.log('square');
  return x.a;
}

export function cube(x) {
  console.log('cube'); // 输出cube
  return x * x * x;
}
```

**问题**：比如index.js为入口文件引入了test.js，而test文件里面有两个函数cube, square，如果在index.js文件中只使用了cube函数，square函数没有使用，这个时候，webpack 4 还会将square函数编译。

**webpack 5 tree shaking作用：**：打包体积更小

  1.webpack 5 能够处理嵌套模块的tree shaking

**代码实现：**:
```javascript
// test.js
function a () {
  console.log('a')
}

function b () {
  console.log('b')
}

export default {
  a, b
}
// index.js
import a from './test.js'
// 使用a变量
console.log(a.a())
console.log('hello world');
```

在上述代码中，把test文件里面没有使用到的b函数删除，正确的保留了a函数。webpack4是做不到这一点的，只有webpack5才有这个功能。webpack 4 没有分析模块的导出和引用之间的依赖关系。webpack 5 有一个新的选项optimization.innerGraph，在生产模式下是默认启用的，它可以对模块中的标志进行分析，找出导出和引用之间的依赖关系。

2.webpack 5 能够处理多个模块之间的关系
```javascript
import { something } from './something';
function usingSomething() {
  return something;
}
export function test() {
  return usingSomething();
}
```
上述代码中，当设置了"sideEffetcs: false"时，内部依赖图算法会找出 something 只有在使用 test 导出时才会使用。一旦发现test方法没有使用，不但删除test，还会删除./something。
sideEffects 是什么呢？让webpack去除tree shaking带来副作用的代码。false为了告诉webpack我这个npm包里的所有文件代码都是没有副作用的。

```javascript
// a.js
export function a() {}

// b.js
export function b(){}

// package/index.js
import a from './a'
import b from './b'
export { a, b }

// app.js
import {a} from 'package'
console.log(a)

当我们以app.js为 entry 时，经过摇树后的代码会变成这样：
// a.js
export function a() {}

// b.js 不再导出 function b(){}
function b() {}

// package/index.js 不再导出 b 模块
import a from './a'
import b from './b'
export { a }

// app.js
import {a} from 'package'
console.log(a)

但是如果 b 模块中添加了一些副作用，比如一个简单的 log：
// b.js
export function b(v) { reutrn v }
console.log(b(1))

打包之后会发现 b 模块内容变成了：
// b.js
console.log(function (v){return v}(1))
虽然 b 模块的导出是被忽略了，但是副作用代码被保留下来了。由于目前 transformer 转换后可能引入的各种奇怪操作引发的副作用，很多时候我们会发现就算有了 tree shaking 我们的 bundle size 还是没有明显的减小。而通常我们期望的是 b 模块既然不被使用了，其中所有的代码应该不被引入才对。
这个时候 sideEffects 的作用就显现出来了：如果我们引入的包/模块 被标记为 sideEffects: false 了，那么不管它是否真的有副作用，只要它没有被引用到，整个模块/包都会被完整的移除。
```

3.webpack 5 还能处理对CommonJs的tree shaking


## 代码分割

**方式一**: 假如项目中有两个文件，想让每一个文件都单独打包（默认两个文件打包进一个bundle），这个时候采用多入口文件的方式。
```javascript
// 单入口
entry: './src/index.js',

entry: {
  // 多入口：有一个入口，最终的输出就有一个bundle
  index: './src/index.js',
  test: './src/test.js'
},
```
**方式二**:

**问题**：比如在入口js文件中引入了lodash库，没做处理的话，在打包的时候，会打包到一个文件中，特别大，lodash占用了很大内存。另外，如果在test文件中也引入lodash，b文件中同样引入了lodash，那么会打包两次。

**作用**：将打包生成的一个文件，分割成多个文件，分割成多个文件之后，各个文件代码体积小，同时并行加载，加载速度更快，同时实现按需加载。

**代码配置**：
```javaScript
// 1. 可以将node_modules中代码单独打包一个chunk最终输出；
// 2. 自动分析多入口chunk中，有没有公共的文件（这个文件不能太小）。如果有会打包成单独一个chunk；不会重复打包多次。
optimization: {
  splitChunks: { 
    chunks: 'all'
  }
},
```

**方式三**：
```javaScript
// 通过js代码，让某个文件被单独打包成一个chunk
// import动态导入语法：能将某个文件单独打包（这种写法是ES10写法）
// 修改了打包的名字，若不修改，则是通过id来添加name
import(/* webpackChunkName: 'test' */'./test')
  .then(({ mul, count }) => {
    // 文件加载成功
    // eslint-disable-next-line
    console.log(mul(2, 5));
  })
  .catch(() => {
    console.log('文件加载失败');
  });
```
### 在上述代码中，若不添加webpackChunkName，那么打包之后的文件名称就是生成的id，而在webpack 5当中，在开发环境当中，可以不使用import(/* webpackChunkName: 'test' */'./test')来为chunk命名了，生产环境仍然有必要。webpack内部有chunk命名规则，不再是以id（0,1,2）来命名了。


## 生产环境优化代码运行的性能---lazy loading
**问题**：有两个文件，每个文件中都涉及相应的函数，我们想要其中一个文件中的函数在进行操作之后再执行，但是当打包后发现事件还没有触发，就执行了，这个时候需要添加懒加载。不会重复加载，当第一次加载完之后，第二次会读取缓存，不会重新加载。

**前提**：先进行代码分割，将代码分割成单独的js文件，再对这个单独的文件进行懒加载。

**特点**：所谓的懒加载就是利用代码分割，将代码分割的import语法放入到一个异步的回调函数中，这个异步的回调函数作为懒加载代码的触发条件。

**代码配置**：
```javaScript
document.getElementById('btn').onclick = function() {
  // 懒加载：当文件需要使用时才加载。
  // 预加载 prefetch：会在使用之前，提前加载js文件。
  // 正常加载可以认为是并行加载（同一时间加载多个文件）  
  // 预加载 prefetch：等其他资源加载完毕，浏览器空闲了，再偷偷加载资源，好处就是如果遇到大文件，会提前加载好，省得当用户触发的时候需要花费更多的时间等待加载该文件。不会阻塞其他文件加载，会等其他文件加载完毕之后加载。缺点是兼容性很差，只能在PC端的高版本浏览器中使用，移动端、IE浏览器有很大的兼容性问题。
  import(/* webpackChunkName: 'test', webpackPrefetch: true */'./test').then(({ mul }) => {
    console.log(mul(4, 5));
  }).catch((err) => {
      console.log('加载失败');
  });;
};

console.log('base.js被加载');

export function mul(a, b) {
  return a - b;
}
```

## 新一代构建工具
esbuild、Snowpack、vite、webpack(特指webpack 5)等。

webpack5里面甩掉了很多历史包袱，让代码更加严谨、规范化才能正常运行。

webpack的痛点：
* Webpack 的热更新会以当前修改的文件为入口重新 build 打包，所有涉及到的依赖也都会被重新加载一次；
* webpack每次启动项目，都需要预打包，对所有的文件进行打包导致所需要的时间较长。

<font color=#00ffff>目前，Vite已经和vue解耦，逐渐成为新型框架首选的工程化工具。</font>

<font color=#00ffff>**下一代项目构建工具Vite**</font>
