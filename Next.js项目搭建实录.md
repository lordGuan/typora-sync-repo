## Next.js项目搭建实录

本文档随手记录基于Next.js项目的搭建过程，融合各个功能的过程中遇到的问题，评估是否可以满足开发需要，同时考虑便利性和稳定性。记录的顺序部分前后。

### 使用styled-components

Next.js支持css-in-js，但是写法别扭所以暂不考虑，使用React比较流行的styled-components，把HTML标签装饰成组件的形式，复用度高，自带代码分割和前缀补全，对于“主题”支持更好，维护起来也很方便。编程式的样式书写，也有一定的SCSS或less的优势。webstorm安装一下styled-components支持，就可以方便的书写样式表。

> npm install --save styled-components

为了支持Next.js的SSR，还要手动创建（如果没有的话）.babelrc文件，开启styled-components的ssr支持：

```json
{
  "presets": [
    "next/babel"
  ],
  "plugins": [
    [
      "styled-components",
      {
        "ssr": true
      }
    ]
  ]
}
```

扩展Next.js的App植入`<ThemeProvider>`，在这里使用createGlobalStyle创建全局样式：

```tsx
import App from 'next/app'
import React from 'react'
import {ThemeProvider, createGlobalStyle} from 'styled-components'

const theme = {
  colors: {
    primary: '#0070f3'
  }
};

const GlobalStyle = createGlobalStyle`
  body {
    padding: 0;
    margin: 0;
  }
`;

export default class MyApp extends App {
  render() {
    const {Component, pageProps} = this.props;
    return (
      <ThemeProvider theme={theme}>
        <React.Fragment>
          <GlobalStyle/>
          <Component {...pageProps} />
        </React.Fragment>
      </ThemeProvider>
    )
  }
}
```

扩展Next.js的Document注入用于加载样式表的head（讲道理这个并没有看懂，但是styled-components的官方文档让我们直接copy这些逻辑，那我就不管了）：

```tsx
import Document from 'next/document'
import {ServerStyleSheet} from 'styled-components'

export default class MyDocument extends Document {
  static async getInitialProps(ctx) {
    const sheet = new ServerStyleSheet();
    const originalRenderPage = ctx.renderPage;

    try {
      ctx.renderPage = () =>
        originalRenderPage({
          enhanceApp: App => props => sheet.collectStyles(<App {...props} />)
        });

      const initialProps = await Document.getInitialProps(ctx);
      return {
        ...initialProps,
        styles: (
          <>
            {initialProps.styles}
            {sheet.getStyleElement()}
          </>
        )
      }
    } finally {
      sheet.seal()
    }
  }
}
```



### 使用iconfont

由于iconfont用到了svg和ttf等特殊文件，所以需要一些特殊的帮助。通过查询，我们需要`next-font`来扩展webpack配置。

```js
const withSass = require('@zeit/next-sass');
const withCSS = require('@zeit/next-css');
const withFonts = require('next-fonts');

module.exports = withFonts(withCSS(withSass({
  enableSvg: true,
  webpack(config, options) {
    return config;
  }
})));
```

注意处理的顺序，现将sass处理成css，在处理css的时候需要引用字体等，所以顺序是withFonts>withCSS>withSass。并且配置要注意webpack属性，将config返回出来。处理好loader，就可以在_app.tsx中进行全局引用了。

### 使用路由

Next.js的路由系统基于`/pages`目录下的文件组织，和请求进行对应，参数路由以[params].js对应。

> 测试发现：在/pages目录下所有的子目录及组件文件，都会被映射成路由。比如/pages/post/components/Button.js会响应/post/components/Button请求。

`/pages`目录下所有的非“_”开头的文件，都将按其相对路径被映射成路由，所以页面中包含的组件，需要单独出去进行维护。

### 使用react-intl国际化

[官方范例](https://github.com/zeit/next.js/tree/canary/examples/with-react-intl)

#### 基本用法

按照react-intl的要求准备语言文件，要注意的是react-intl使用的messages是一个普通的平铺的对象，但是键可以使用路径形式，即‘common.confirm.ok’。

```js
module.exports = {
  title: 'Next-Demo 标题',
  greeting: '欢迎!',
  'common.confirm.ok': '确认'
};
```

因为下面的例子多语言文件是从服务端加载的，所以使用CommonJS形式导出。为了更好的维护多语言文件，可以引用`flat`库将其平铺。

next-with-react-intl这个例子，主要是在服务端结合了国际化的部分。先来看server.js的部分代码：

```js
server.get('*', (req, res) => {
    const accept = accepts(req);
    const locale = (accept.language(accept.languages(supportedLanguages)) || 'zh-CN').split("-")[0];
    req.locale = locale;
    req.localeDataScript = getLocaleDataScript(locale);
    req.messages = flat(getMessages(locale));
    return handle(req, res);
  });


const localeDataCache = new Map();
const getLocaleDataScript = locale => {
  const lang = locale.split('-')[0];
  if (!localeDataCache.has(lang)) {
    const localeDataFile = require.resolve(`@formatjs/intl-relativetimeformat/dist/locale-data/${lang}`);
    const localeDataScript = readFileSync(localeDataFile, 'utf8');
    localeDataCache.set(lang, localeDataScript)
  }
  return localeDataCache.get(lang)
};
```

理论上服务端跟react-intl没关系，而是跟`intl`国际化有关系。这里负责解析所有的请求，并通过`accepts`库，根据请求信息来决定应当使用的语种，然后通过`getLocaleDataScript`方法，将`formatjs`中对应语种的通用多语言内容提取出来，再将对应语言的多语言文件内容读取出来，统统塞到`req`中，这些信息就会被带到浏览器端。下面的工作交给_app.tsx和\_document.tsx。

```tsx
// _app.tsx
export default class MyApp extends App<Props> {
  static getInitialProps = async function ({Component, router, ctx}) {
    let pageProps = {};

    if (Component.getInitialProps) {
      pageProps = await Component.getInitialProps(ctx);
    }

    const {req} = ctx;
    const {locale, messages} = req || (window as any).__NEXT_DATA__.props;
    return {pageProps, locale, messages};
  };


  render() {
    const {Component, pageProps, locale, messages} = this.props;
    return (
      <ThemeProvider theme={theme}>
        <IntlProvider locale={locale} messages={messages}>
          <Component {...pageProps} />
        </IntlProvider>
      </ThemeProvider>
    )
  }
}
```

扩展App组件，相当于全局处理，所以无论你访问哪个页面，都会先处理这里的逻辑。getInitialProps方法中，从上下文`ctx`中拿到req，从req中获得我们在后端插入的locale和messages属性，将这两个属性设置给`<IntlProvider/>`组件。使用多语言如下：

```tsx
// index.tsx
export default class Index extends React.Component {
  render() {
    return (
        <div>
          <FormattedMessage id="greeting" defaultMessage="拉拉拉"/>
        </div>
    )
  }
}
```

使用`FormattedMessage`组件根据id属性来加载多语言内容，defaultMessage用于在找不到对应id的时候默认展示。

#### 切换多语言



### 使用Redux

[官方范例](https://github.com/zeit/next.js/tree/canary/examples/with-redux)

