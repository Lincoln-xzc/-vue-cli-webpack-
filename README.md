# wepack 打包vue项目速度优化
其实网上现在很多成熟的优化方案，本人这里也是做一个笔记，希望能给大家带来方便以及自己方便翻阅。
自己的项目是基于vue脚手架构建的一个项目，如果还不会构建vue项目，自己可以去网上查阅一下，很容易搞定。随着项目不断的增大，引入了很多第三方包，打包项目
的时间也越来越久，终于忍受不了决定优化一下。
可以看下现在打包所花费的时间，跟优化后的打包时间做下对比
优化的话从以下几个方面进行
> 1. 在webpack.base.conf.js中对vue-loader添加include,exclude.这样就能让vue-loader快速定位在那些目录文件需要处理。
  - include 表示哪些目录中的 .js 文件需要进行 babel-loader
  - exclude 表示哪些目录中的 .js 文件不要进行 babel-loader
```
  test: /\.vue$/,
  loader: 'vue-loader',
  options: vueLoaderConfig,
  include: [resolve('src'), resolve('node_modules')]
```
> 2.将一些很少数改动的包拆分出来提前打包，这里我们引入dllPlugin插件。首先我们需要安装DllPlugin,DllReferencePlugin（npm i DllPlugin DllReferencePlugin）,
    然后在build目录下新建一个webpack.dll.conf.js文件，加入以下代码：
```
const path = require('path');
const webpack = require('webpack');
const UglifyJsPlugin = require('uglifyjs-webpack-plugin')
function resolve(dir){
    return path.join(__dirname, '..', dir);
}

module.exports = {
    entry:{
        vendor: ['vue/dist/vue.esm.js', 'lodash', 'echarts', 'vuex', 'axios', 'vue-router', 'element-ui']
    },
    output:{
        path: path.join(__dirname, '../static/js'), //路径
        filename: '[name].dll.js', //打包文件名
        library: '[name]_library', //暴露在全局的对象名称
    },
    // resolve:{
    //     extensions: ['.js','.vue', '.json'],
    //     alias: {
    //         'vue$': 'vue/dist/vue.esm.js',
    //         '@': resolve('src')
    //     }
    // },
    plugins:[
        new webpack.DllPlugin({
            path: path.join(__dirname, '../dist', '[name]-manifest.json'), 
            name: '[name]_library'
        }),
        new webpack.optimize.UglifyJsPlugin({
            compress: {
              warnings: false
            }
          })
    ]
};
```
**重点：这里引入的Dllplugin插件，执行后将生成一个manifest.json文件和[name].dll.js文件。manifest.json文件供webpack.config.js中加入的DllReferencePlugin使用，使我们所编写的源文件能正确地访问到我们所需要的静态资源（运行时依赖包）。**
在package.json文件中加入"dll:webpack -p --progress --config build/webpack.dll.conf.js"，然后运行npm run dll生成相应文件.
在webpack.base.conf.js文件中通过DllReferencePlugin加入我们刚才生成的manifest.json文件
```
  new webpack.DllReferencePlugin({
    context:  path.resolve(__dirname, '..'),
    manifest: require('../dist/vendor-manifest.json')
  }),
```
现在已经将一些模块提前打包出来了，我们运行一下，发现速度提升了很多。
> 3.由于运行在 Node.js 之上的 Webpack 是单线程模型的，所以Webpack 需要处理的事情需要一件一件的做，不能多件事一起做。
我们需要Webpack 能同一时间处理多个任务，发挥多核 CPU 电脑的威力，HappyPack 就能让 Webpack 做到这点，它把任务分解给多个子进程去并发的执行，子进程处理完后再把结果发送给主进程。
**安装happypack**
  npm i happypack
在weabpck.base.conf.js中加入happypack
```
const HappyPack = require('happypack');
const happyThreadPool = HappyPack.ThreadPool({ size: os.cpus().length})
modules:{
  rules:[
    ...
    {
      test: /\.js$/,
      //loader: 'babel-loader?cacheDirectory=true',
      loader: 'happypack/loader?id=happyBabel',
      include: [resolve('src'), resolve('test'), resolve('node_modules/webpack-dev-server/client')],
      exclude: /node_modules(?!\/quill-image-drop-module| quill-image-resize-module)/
    }
  ]
}
plugins:[
  new HappyPack({
      id: 'happyBabel',
      threadPool: happyThreadPool,
      loaders:['babel-loader?cacheDirectory']
    })  
]
 
```
> 4.webpack默认是使用uglifyjsplugin去对js进行压缩的，但是uglifyjsplugin是单线程压缩代码，我们结合happypack，我们引入ParallelUglifyPlugin。ParalleUglifyPlugin能开启多个子进程
  去压缩代码。
  **安装ParallelUglifyPlugin**
  npm i -D webpack-parallel-uglify-plugin
  然后在webpack.prod.conf.js中加入如下代码:
```
  //去除uglifyjsPlugin
  new ParallelUglifyPlugin({
    cacheDir: '.cache/',
    uglifyJS:{
      output: {
        comments: false
      },
      compress: {
        warnings: false
      }
    }
  }),
```
通过以上几个方面的优化打包速度得到了很大的提示，打包之前我们的要花费120多秒，经过这些优化后我们打包只需要20多秒。
