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