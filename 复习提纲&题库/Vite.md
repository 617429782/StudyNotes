## 理念 https://cn.vitejs.dev/guide/why.html
1. 利用浏览器对 ESM 的支持，将部分打包工作交给浏览器，仅在浏览器请求 ESM 模块时，提供预编译后的源码，实现按需加载，这缓解了传统打包 bundle 过大导致启动缓慢的问题
2. 通过模块解析规则区分源码和依赖(是否 npm 引入)；源码模块协商缓存，依赖模块强缓存，减少不必要的资源请求
3. 动态热替换，todo
4. 生产环境采用 Rollup 进行 tree-shaking、chunk 分割 和 懒加载，以获取更好的加载性能

## 特性
1. 依赖预构建，默认支持 ESM 语法: 即便没有配置 type: module, vite 也会将 cjs 和 umd 文件预构建成 ESM 语法，同时将 import 的 npm 包名改写为相对资源地址
2. 智能类型提示: jsDoc 注解 or defineConfig 函数可以辅助开发者在配置 vite.config.ts 时，获得一些提示

## 配置项
  ```
    export default defineConfig({
      define: {
        'process.env.NODE_ENV': true, // 全局字符串替换
      },
      build: {
        lib: {
          entry: 'src/index.ts',
          name: 'SkyriverMessage', // 对外导出的包名，umd 格式下，对应在 window 挂载的命名空间
          formats: ['cjs', 'es', 'umd', 'iife'],
          fileName: (format) => `index.${format}.js`,
        },
        sourcemap: true,
        rollupOptions: { // 生产环境配置 https://cn.rollupjs.org/configuration-options/

        }
      },
    });
  ```

## vue-cli https://cn.vitejs.dev/guide/cli.html
  ```
    vite           // 启动开发服务器
    vite build     // 构建生产版本
    vite optimize  // 预构建依赖
    vite preview   // 本地预览构建产物
  ```

## 常用插件 https://cn.vitejs.dev/guide/using-plugins.html

## with monorepo https://cn.vitejs.dev/guide/dep-pre-bundling.html#monorepos-and-linked-dependencies

## 文件系统缓存 https://cn.vitejs.dev/guide/dep-pre-bundling.html#caching

## VS 其他打包工具 https://cn.vitejs.dev/guide/comparisons