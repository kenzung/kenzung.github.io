---
title: 部署create-react-app应用到gitlab
date: 2018-07-25 22:17:59
tags: 
- gitlab CI
- create-react-app
---

# 配置gitlab CI
最近想把自己写的react程序（使用了eject后的create-reat-app）部署到网上，给自己的GitHub Project添加一个live demo。以前就听说过gitlab CI，这次就想在gitlab CI上折腾一下（主要免费，哈哈）。

首先，需要在项目的根目录上创建一个`.gitlab-ci.yml` 。gitlab CI的配置文件的配置项还是挺多的。具体可以看看[GitLab Docs](https://docs.gitlab.com/ee/ci/)。

这是我构建任务的yaml文件
```yml
image: node:8.9.4 #docker镜像，node版本根据自己需要更改
before_script:
  - npm install
pages:
  stage: deploy
  cache:
    paths:
      - node_modules # 缓存 node_modules
  script:
    - npm run build
    - rm -rf public 
    - mv build public
  artifacts:
    paths:
      - public/
  only:
    - master
```

我在这里遇到的坑总结一下：
1. 这一点估计和该配置文件无关的。我的项目使用了eslint，执行eslint校验时提示warning，在gitlab ci上eslint的warning就会导致build fail。其实是gitlabCI把`process.env.CI`变量被设置为`true`，然后warning就会被当做error。
2. 部署静态页面，需要使用`pages`这个特定的任务名称，自己命名的jobs是不生效的。pages相当于gitlab提供了一个webserver来运行我们构建好的静态页面。静态内容必须放置`public/`目录，`artifacts:paths:`的值必须为`public/`。
3. 需要在`pages`任务下指定stage。

# package.json和react router配置
但是如果按照上述配置，构建虽然会成功。估计很大概率会得到下面这个`404`页面
![404](https://ws1.sinaimg.cn/large/6ec33160gy1ftmk8yxwbej20mr0i1aam.jpg)

## package.json配置
### 首先需要设置一个homepage
为了兼容本地开发和CI编译，把`package.json`的`homepage`设置为`.`。设置这个可以让url路径变为相对路径。若没有设置，访问`kenzung.gitlab.io/dbp7`就会白屏，加载静态资源出错。这个好像争议挺多，具体可以看[StackOverflow issue1](https://github.com/facebook/create-react-app/issues/1487)和[StackOverflow issue2](https://github.com/facebook/create-react-app/pull/1489)。我暂时没有找到更好的解决方案。

![error](https://ws1.sinaimg.cn/large/6ec33160gy1ftmjtntjeoj20fi05sgm5.jpg)

### 设置构建脚本
`package.json`文件
```diff
  "scripts": {
    "start": "node scripts/start.js",
    "build": "node scripts/build.js",
    "test": "node scripts/test.js --env=jsdom",
+    "gitlab:build": "cross-env BASE_PATH=dbp7/ node scripts/build.js",
    "storybook": "start-storybook -p 9001 -c .storybook"
  }
```
这里使用了cross-env包。注入一个变量`BASE_PATH`，`BASE_PATH`的值是gitlab的项目名称。

`config/env.js`文件
```diff
function getClientEnvironment(publicUrl) {
  const raw = Object.keys(process.env)
    .filter(key => REACT_APP.test(key))
    .reduce(
      (env, key) => {
        env[key] = process.env[key];
        return env;
      },
      {
        // Useful for determining whether we’re running in production mode.
        // Most importantly, it switches React into the correct mode.
        NODE_ENV: process.env.NODE_ENV || 'development',
        // Useful for resolving the correct path to static assets in `public`.
        // For example, <img src={process.env.PUBLIC_URL + '/img/logo.png'} />.
        // This should only be used as an escape hatch. Normally you would put
        // images into the `src` and `import` them in code to get their paths.
        PUBLIC_URL: publicUrl,
+        BASE_PATH: process.env.BASE_PATH
      }
    );
    // Stringify all values so we can feed into Webpack DefinePlugin
  const stringified = {
    'process.env': Object.keys(raw).reduce((env, key) => {
      env[key] = JSON.stringify(raw[key]);
      return env;
    }, {}),
  };

  return { raw, stringified };
}
```

在`env.js`中增加的这行代码是必须的，否则reduce函数的initvalue就少了`BASE_PATH`，导致`env`没有没有BASE_PATH。

## react路由更改
经过测试，gitlab CI上使用BrowserRouter是可以，但是刷新页面会白屏。这个应该和[create-react-app官方文档](https://github.com/facebook/create-react-app/blob/master/packages/react-scripts/template/README.md#github-pages)说的可能一致，应该是不支持`history API`导致的。使用`HashRouter`即可解决上述问题。
```JSX
let Router = process.env.BASE_PATH ? HashRouter : BrowserRouter;
return (
    <Router>
        <React.Fragment>
            <Switch>
                <Route exact path="/" component={Home} />
                <Route path="/movie/celebrity/:id" component={ActorDetail}/>
                <Route path="/movie/subject/:id" component={MovieDetail}/>
                <Route path="/movie/:type" component={MoreMovie}/>
                <Route path="/movie" component={Movie} />
                <Route path="/book" component={Book} />
                <Route path="/event/:id" component={EventDetail}/>
                <Route path="/search" component={Search} />
                <Route component={NotFound}/>
            </Switch>
        </React.Fragment>
    </Router>
)
```

最后，由于我的程序含有`HTTP`和`HTTPS`的请求，gitlab是`HTTPS`的，这个mix请求会触发chrome浏览器报错

    This request has been blocked; the content must be served over HTTPS.

因此可以在html头部加上以下代码，意思是让http升级为不安全的https。
```html
<meta http-equiv="Content-Security-Policy" content="upgrade-insecure-requests">
```

最后附上[live demo地址](https://kenzung.gitlab.io/dbp7/)
![](https://ws1.sinaimg.cn/large/6ec33160gy1ftn6nyjj4eg212q0rckaw.jpg)


# 参考
* [So you want to host your Single Page React App on GitHub Pages?](https://itnext.io/so-you-want-to-host-your-single-age-react-app-on-github-pages-a826ab01e48)
* [将 create-react-app 项目部署到 gitlab-pages 上](https://gitivon.gitlab.io/blog/deploy-react-gitlab.html#gitlab-ci-yml-%E9%85%8D%E7%BD%AE)
* [create-reate-app文档](https://github.com/facebook/create-react-app/blob/master/packages/react-scripts/template/README.md#github-pages)
