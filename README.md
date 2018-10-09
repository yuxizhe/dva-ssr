## dva-ssr

### react服务端渲染demo (基于Dva)

#### 使用

- npm install
- npm run buildClient
- npm run buildServer
- npm run ssr
- view localhost:3000

#### 功能

- 支持Dva（即基于 react, react-router, redux, redux-saga 的状态管理）
- 支持Code Splitting （不再使用Dva自带的 dva/dynamic加载组件）
- 支持 CSS Modules

#### SSR实现逻辑

##### 概览

![ssr overview](doc/dva_ssr.png)

上图是SSR的运行时流程图（暂时不考虑构建的问题）

收到客户端的SSR请求后，SSR Server将依次执行如下五部操作：

1. 对请求的路径，进行路由匹配；并 "获取/加载"(获取对应同步组件，加载对应异步组件) 所涉及的组件

```javascript
  
  // 初始化
  const history = createMemoryHistory();
  history.push(req.path);
  const initialState = {};

  const app = dva({history, initialState});
  app.router(router);
  const App = app.start();
  let routes = getRoutes(app);

  // 匹配路由，获取需要加载的Route组件（包含Loadable组件）
  const matchedComponents = matchRoutes(routes, req.path).map(({route}) => {
    if (!route.component.preload) {
      // 同步组件
      return route.component;
    } else {
      // 异步组件
      return route.component.preload().then(res => res.default)
    }
  });
  const loadedComponents = await Promise.all(matchedComponents);
```

2. 对1中组件进行初始化（如需），进行接口请求，并等待请求返回。

> 注： 需要进行数据初始化的组件，需要定义 static fetching 方法

```javascript
const actionList = loadedComponents.map(component => {
    if (component.fetching) {
      return component.fetching({
        ...app._store,
        ...component.props,
        path: req.path
      });
    } else {
      return null;
    }
  });
await Promise.all(actionList);
```

3. 调用 ReactDOMServer.renderString 渲染数据

```javascript

//  Render Dva App。同时使用Loadable.Capture 捕捉本次渲染包含的Loadable组件集合Array<String>。
const modules = [];
const markup = renderToString(
    <Loadable.Capture report={module => modules.push(module)}>
      <App location={req.path} context={{}}/>
    </Loadable.Capture>
);

//  构造需要render的 script标签。其中利用了react-loadable的webpack插件在构建过程中生成的module字典
let bundles = getBundles(moduleDict, modules);
let scripts = bundles.filter(bundle => bundle.file.endsWith('.js'));
let scriptMarkups = scripts.map(bundle => {
  return `<script src="/public/${bundle.file}"></script>`
}).join('\n');
```

> Loadable 的相关概念和用法，请参考 github: [react-loadable](https://github.com/jamiebuilds/react-loadable)

##### Code Splitting


4. 获取preloadedState

```javascript
const preloadedState = app._store.getState();
```

5. 拼装Html，并返回
```javascript
res.send(`
<!DOCTYPE html>
<html>
  <head>
    <title>React Server Side Demo With Dva</title>
      <link href="/public/style.css" rel="stylesheet">
  </head>
  <body>
    <div id="app">${markup}</div>
    <script>window.__PRELOADED_STATE__ = ${JSON.stringify(preloadedState).replace(/</g, '\\\u003c')}</script>
    <script src="/public/main.js"></script>
    ${scriptMarkups}
  </body>
</html>
`);
```

