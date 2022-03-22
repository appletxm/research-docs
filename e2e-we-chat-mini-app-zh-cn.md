
![about us](images/a.png)

团队开发小程序已经有一段时间了，随着开发的功能越来越多，我们的测试同学回归的任务也越发的重，所以我们决定用自动化测试来减轻一些回归测试的压力，同时也可以用来作为我们应用日常可访问性检查的一个工具，话不多说我们马上进入正题。

## 1. 方案确定
方案主要是围绕以下我们的几个需求
1. 覆盖小程序
2. 覆盖web-view
3. 测试框架
4. 原生页面与web-view页面的通讯
5. 输出测试结果（excel, 使用html在线展示）
6. 错误信息的收集

#### 1.1 覆盖小程序原生功能
能够在小程序上面使用的方案不多，团队内经过一翻讨论后，我们最终决定用官方提供的[miniprogram-automator](https://www.npmjs.com/package/miniprogram-automator)，使用可以查看[官方使用文档](https://developers.weixin.qq.com/miniprogram/dev/devtools/auto/quick-start.html)，这是一个基于nodejs的模块，所以对于我们js程序员来讲更加容易上手，这里要重点感谢一下测试同学的支持，她被迫也要使用js写自动化测试，再次感谢。

在查看了官方的使用文档后，发觉这个`miniprogram-automator`其实就是一个被阉割了的[Puppeteer](https://github.com/puppeteer/puppeteer)，使用过puppeteer的同学应该都不会陌生。

#### 1.2 覆盖web-view功能
对于web-view这一块的自动化，我们选用的是[puppeteer](https://github.com/puppeteer/puppeteer)（[你也可以查看中文api文档](https://zhaoqize.github.io/puppeteer-api-zh_CN/#/)），这是一个基于nodejs的模块，它所提供的API及功能都可以满足我们需求，同时微软推出的[playwright](https://github.com/microsoft/playwright)也是一个不错的选择，不过鉴于太新需要重新了解api及尝试，所以暂时没有采用，以后希望有机会可以尝试一下。

具体来讲我们使用的是`puppeteer-core`这个nodejs模块，因为我们不太希望在安装项目依赖时候要额外安装一个浏览器，这个过程太花时间了，而且还有安装不成功的风险。所以我们直接安装`puppeteer-core`，然后使用配置使用已经安装好的浏览器即可，具体与`puppeteer`的差异，我们可以查看[puppeteer-vs-puppeteer-core](https://github.com/puppeteer/puppeteer/blob/main/docs/api.md#puppeteer-vs-puppeteer-core)。

#### 1.3 测试框架
测试框架可供选择的还是挺多的[jest](https://jestjs.io/), [mocka](https://mochajs.bootcss.com/), [jasmine](https://jasmine.github.io/)等都是不错的，鉴于我们对框架的熟悉程度，我们选择了`jest`，相对简单并且文档也还不错。

#### 1.4 输出测试结果
我们期望测试结果的输出方式是excel及在线的html预览，所以我们需要使`jest`的自定义`reporter`配置项来注册hook事件，在测试完成后可以接收到测试的结果。具体配置内容可以查看官网[jest自定义reporter](https://jestjs.io/docs/configuration#reporters-arraymodulename--modulename-options)

#### 1.5 错误信息的收集
我们在收集到测试结果的同时，我们其实还是需要关注那些测试用例出错，并且出错的具体信息。这部分的测试出错信息Jest是有返回的，不过返回的格式是`ansi`，于展示也不友好，于是我们需要引入另外的模块 `ansi-to-html`，可以方便地将ansi转换成html，这就很完美了。

#### 1.6 原生与web-view之间的通讯
鉴于我们的web-view的访问地址是需要原生的页面产生的，所以为了能够保证web-view可以直接使用这些地址，我们使用了一个临时的文本文件来存储这些地址信息，在puppeteer在加载这些页面的时候，我们会在临时的文本文件里边查找。

#### 1.7 方案总结

经过上述的多个步骤，最终我们的自动化测试项目架构如下：

![solution diagram](https://image-static.segmentfault.com/323/484/3234840757-623820c981394_fix732)


## 2. 方案实施

#### 2.1 配置小程序开发工具

在使用小程序官方提供的`miniprogram-automator`进行自动化测试，我们需要完成以下几个配置
1. 配置好小程序开发工具的cli目录，此目录一般是在小程开发工具的安装目录下，并且路径地址分格符需要使用 `‘/’`无论window还是mac，如‘path/to/cli’
2. 配置好小程序原码的地址

windows下配置例如下：
```javascript
{
  projectPath: 'D:/project/code/product/mini-app/dist',
  cliPath: 'F:/ssd programs/微信web开发者工具/cli.bat'
}
```
mac下配置例如下
```javascript
{
 projectPath: "/Users/nb666/Desktop/me/project/product/code/mini-app/dist",
 cliPath: "/Applications/wechatwebdevtools.app/Contents/MacOS/cli"
}
```

3. 打开小程序服务端口

![config dev-tools](https://image-static.segmentfault.com/217/118/2171188632-623814b95d72a_fix732)
打开端口配置

#### 2.2 配置puppeteer
因为我们用的是puppeteer-core，所以我们需要额外指定一个浏览工具给它，因为我本机安装的是chrome，所以我配置的是chrome浏览器的exe地址，不过就官方说明，只要有dev-tools的浏览器都可以用作它的运行浏览器，包括firefxo跟edge。

以下是我们这边项目的puppeteer配置内容，供参考。
```javascript
puppeteerCfg: {
    browserConfig: {
      executablePath: 'C:\\Program Files (x86)\\Google\\Chrome\\Application\\chrome.exe',
      headless: false,
      ignoreHTTPSErrors: true,
      devtools: true,
      defaultViewport: {
        width: 1440,
        height: 900,
      },
      args: [
        '--no-sandbox',
        '--disable-setuid-sandbox'
      ]
    },
    pageConfig: {
      waitUntil: 'networkidle0',
      timeout: 0
    },
    mockDevice: 'iPhone 6'
  }
```

#### 2.3 配置Jest

对于Jest的配置，除了常用的之外，我们还需要配置上述`1.4 输出测试结果`的自定测试结果的收集脚本。

![jest report](https://segmentfault.com/img/bVcYC06)
配置jes.config.js文件的reporters项

![report.sj](https://segmentfault.com/img/bVcYC1p)
jest-repoerter.js文件

在上述的配置后，我们在每次完成测试后是可以拿到测试结果的，不过我们还要需要对结果内容进行修饰。对于要输出excel文件的需求，我们使用的是[exceljs](https://www.npmjs.com/package/exceljs)，而对于要输出在线查看的需求，我们将测试结果直接通过`web-scoke`t或者`sse(server-sent events)`将结果下发给到对应的浏览器窗口。

下面附上我们项目中使用到的Jest配置文件jest.config.js，供大家参考。
```javascript
module.exports = {
  moduleFileExtensions: [
    'js',
    'html',
    'json'
  ],
  transform: {
    '^.+\\.js$': 'babel-jest'
  },
  moduleDirectories: [
    'node_modules'
  ],
  moduleFileExtensions: ['js', 'json', 'html', 'scss', 'css'],
  moduleNameMapper: {
    '\\.(jpg|jpeg|png|gif|eot|otf|webp|svg|ttf|woff|woff2|mp4|webm|wav|mp3|m4a|aac|oga|less|css)$': '<rootDir>/test/mocks/mock-file.js'
  },
  testMatch: ['<rootDir>/test/**/*.spec.js'],
  globals: {
    '__DEV__': true,
    '__ENV__': 'TEST'
  },
  globalSetup: '<rootDir>/scripts/jest-global-setup.js',
  globalTeardown: '<rootDir>/scripts/jest-global-teardown.js',
  reporters: ["default", "<rootDir>/scripts/jest-report.js"]
}
```

#### 2.4 编写测试代码
对于测试代码的编写我们使用es6，所以需要在上述的Jest配置中设置`transform`配置项，接着我们分别列举一下基于`puppeteer-core`及`miniprogram-automator`如何实现自动化代码的编写，相信大家就会明白了。

1. 使用`miniprogram-automator`的参考实例

这个是用来测试首页的`登录`按钮点击后能跳转到登录页面
```javascript
const automator = require('miniprogram-automator')
const waitTime = require('../scripts/util-wait-time')
const { wsEndpoint } = require('../config')
const { loadedLoginPage } = require('./test-login-models')

jest.setTimeout(120 * 1000) // 设置jest完成当前文件下面所有的test case所需要用的时间

describe('User Login', () => {
  let miniProgram = null
  let page = null

  beforeAll(async () => {
    miniProgram = await automator.connect({
      wsEndpoint
    })

    page = await miniProgram.currentPage() // 默认会是小程序的第一个页面

    await page.waitFor(async () => {
      const locaNode = await page.$$('.search-name')
      return locaNode.length > 0
    })
  })

  it('should open login page after click the entry ad. in home page', async () => {
    const enteryNode = await page.$('.ad-banner-image')
    await enteryNode.tap()

    await waitTime(5000) //让小程序充分完成页面的渲染，这里有点尴尬，需要给足够的时间，不然拿不到跳转后的页面

    const { loginButton } = await loadedLoginPage(miniProgram)
    expect(loginButton.length).toBeGreaterThan(0)
  })

  afterAll(async () => {
    await mockLocation.doRestore({ miniProgram })

    await miniProgram.close()
    page = null
    miniProgram = null
    await waitTime(3000)
  })
})
```

2. 使用`puppeteer-core`的参考实例

配置文件config.js

```javascript
{
puppeteerCfg: {
    browserConfig: {
      executablePath: 'C:\\Program Files (x86)\\Google\\Chrome\\Application\\chrome.exe',
      headless: false,
      ignoreHTTPSErrors: true,
      devtools: true,
      defaultViewport: {
        width: 1440,
        height: 900,
      },
      args: [
        '--no-sandbox',
        '--disable-setuid-sandbox'
      ]
    },
    pageConfig: {
      // waitUntil: 'networkidle2',
      waitUntil: 'networkidle0',
      // waitUntil: 'load',
      timeout: 0
    },
    mockDevice: 'iPhone 6'
  }
}
```

对puppeteer操作的封装 puppeteer-opts.js

```javascript
const puppeteer = require('puppeteer-core')
const { puppeteerCfg } = require('../config')
const { browserConfig, pageConfig, mockDevice } = puppeteerCfg
const allDevices = puppeteer['devices']

let wsEnd = null
let browser = null

async function createBrowser() {
  if (!wsEnd) {
    browser = await puppeteer.launch(browserConfig)
    wsEnd = browser.wsEndpoint()
  } else {
    browser = await puppeteer.connect({
      browserWSEndpoint: wsEnd
    })
  }

  // global.browser = browser

  return browser
}

async function createPage(passedBrowser, linkUrl, customize) {
  const phone = allDevices[mockDevice || 'iPhone 6']
  const pages = await passedBrowser.pages()
  const page = pages[0]
  // const page = await passedBrowser.newPage()

  await page.emulate(phone)

  if (customize && typeof customize === 'function') {
    await page.goto(linkUrl)
    await customize(page)
  } else {
    await page.goto(linkUrl, customize || pageConfig)
  }

  return page
}

async function closeBrowser() {
  if (browser) {
    await browser.close()
    browser = null
  }
  if (wsEnd) {
    wsEnd = null
  }
}

module.exports = {
  createBrowser,
  createPage,
  closeBrowser
}
```

单元测试文件 test.js

```javascirpt
const { createBrowser, createPage } = require('../scripts/puppeteer-opts')
const { getStringFromTmpFile } = require('../scripts/tmp-file-opts')
const waitTime = require('../scripts/util-wait-time')
const { customInfo } = require('../config')
const { waitSSOcall } = require('./test-aa-models')

let linkUrl = null
let browser = null
let page = null

const inputInfo = customInfo

jest.setTimeout(300 * 1000)

describe('Puppeteer Sample', () => {
  beforeAll(async () => {
    const otaDeepLink = getStringFromTmpFile('otaDeepLink')
    linkUrl = otaDeepLink

    console.log('**link from cache**', linkUrl)

    browser = await createBrowser()
    page = await createPage(browser, linkUrl, {
      waitUntil: 'domcontentloaded',
      timeout: 5 * 60 * 1000
    })

    await waitSSOcall(page, '/um/v2/users/')
  })

  it('should open search result page success', async () => {
    const listNodes = await page.$$('a[class^=Button__ButtonContainer]')
    console.log('**search list leng**', listNodes.length)
    expect(listNodes.length).toBeGreaterThanOrEqual(1)
  })

  afterAll(async () => {
    await browser.close()
    page = null
    browser = null
  })
})
```

#### 2.5 执行测试用例
我们通过设置Jest配置文件`testMatch: ['<rootDir>/test/**/*.spec.js']`，让Jest去特定的目录下面收集以`.spec`为后缀名的js文件所包含的所有测试用例。这本质上是不需要人为去干预这个收集跟执行的顺序的，但是如果我们有执行顺序上面的要求，我们就需要使用额外的文件来控制执行的顺序，如下截图`index.spec.js`就实现了让两个测试用例文件按顺序执行的控制。

![e2e test](https://segmentfault.com/img/bVcYEJa) 


## 3. 结果呈现
excel的结果展示如下
![excel report](https://segmentfault.com/img/bVcYD2E)

html的结果展示如下
![html report](https://segmentfault.com/img/bVcYD2D)

错误信息的展示如下
![error show](https://segmentfault.com/img/bVcYENP)



## 4. 注意事项
1. 所有的授权需要手动触发预先完成
2. 元素selector只能支持元素跟样式，并且不能有多级
3. 选择自定义组件下面的元素需要在样式前端加上组件名作为前缀
4. 不能覆盖web-view
5. 需要扫码登录后才能使用dev-tool作为自动化测试终端





