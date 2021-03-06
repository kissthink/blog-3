# 基于 GitHub 搭建博客的新思路以及 PWA 模式初探
对于程序猿来说，大家大多希望自己的博客逼格高一些，很多现成的博客平台多半是看不上眼的。这也难怪，程序猿本来就爱折腾，尤其对于前端程序员来说，不仅博客要自己搭，还要看起来漂亮，可定制，可以用自己的前端技能给博客增添很多炫酷的效果。

针对这类人群，搭建博客也有不少解决方案，比如博客框架，如著名的 Wordpress，功能强大历史悠久。如果觉得 Wordpress 太重的话还有一些静态博客生成器，如 Jekyll 和 Hexo，它们通过解析写好的 Markdown 文件生成静态的 HTML 页面。页面的结构，样式和脚本插件都是可定制的，可以基于本身的框架进行二次开发，形成了繁荣的生态圈。

另外，还有一些人比较返璞归真，他们不爱折腾这些，只专注于内容本身，至于 UI 怎么样，有没有炫酷的功能都不重要。对于这样的人群，他们选择了直接在 GitHub 上开一个仓库，用仓库的 issues 功能写博客。这其实很方便，issue 里的第一条作为博客的内容，后面的回复都是评论，互相提醒就直接 @，issues 之间也可以相互引用。即使 issues 功能的初衷不是用来写博客，但真用来写博客也没什么违和感。我也是用 issues 写博客的一员，原因也是折腾烦了，懒得折腾。

不过之前在开发 [bowl](https://elemefe.github.io/bowl/) 这个库，给它写文档的时候用到了 GitHub Pages 服务，在用的过程中我冒出一个想法，可以用 GitHub Pages 服务来开发一个博客，以一种有些不一样的方式。

## 博客架构
用 GitHub Pages 作为博客的载体并不新鲜，Jekyll 和 Hexo 都是这么干的，不过它们的做法是将用户的 Markdown 文件转换成 html 文件，和生成的 CSS 及 JS 文件全部放在新生成的一个 public 目录中，然后将整个 public 目录上传到 GitHub 仓库的 gh-pages 分支上。GitHub 将以这个分支的整个目录作为一个静态服务器的根目录，访问提供的域名后相当于访问这个分支下的 index.html 文件。（这是我记忆中的方式，我很久没用这套东西了，可能和现在的方式有些出入，不过我想出入不会太大）

这种方式如果不加任何优化的话，在我看来有一些缺点。一个是站点纯静态，没有后台，如果要加入评论功能的话必须使用第三方插件，而评论功能是一个博客所必需的，而第三方评论插件也不太靠谱，国内常用的多说感觉很不稳定，国外的 Disqus 又容易被墙，而且注册要好像用谷歌脸书账号啥的，总之不太符合社会主义核心价值观。另一个缺点就是托管在 GitHub 仓库里的是生成的 html，而不是源 Markdown 文件，因此存放这些源文件又是一笔额外的维护成本。

但是实际上 GitHub Pages 除了运行在 gh-pages 分支上以外，还可以运行在 master 分支上的 docs 目录中，另外，源文件如果不想额外维护的话其实可以让 GitHub 帮我们维护。我注意到 GitHub 的 issues 已经包含了一些博客的基本功能，而 issues 也可以用 Markdown 语法来写，那么只要去调用 GitHub 的 Open API 就完全可以操纵 issues 的功能了，我们只需要编写一套静态页面，在页面中去调用 GitHub 的 API 即可。另外这个页面如果做成 SPA 的话，只需要一个 index 文件，用 hash 进行路由就可以实现多页面的跳转，这个可以有。

思路到这里已经比较清楚了，我们将单页应用 build 到 docs 目录中，通过域名访问到目录中的 index 文件，在 index 页面中执行我们的单页应用脚本，后台接口直接用 GitHub 的 API 操作本仓库的 issues。这样，用户访问页面看博客评论博客一系列的操作全部调用的是 issues 的接口，换句话说，GitHub 为我们做好了所有的后端开发，包括数据存储，我们只需要写前端就行了。

## API 权限
开发一个单页应用，这是前端的本职工作了，写起来自然得心应手。用什么框架都随意，纯手写也完全可以，我对 Vue 比较熟悉，就选择用 Vue 了。为了节省时间，安装好 vue-cli 后执行一句 `$ vue init webpack`，一个功能齐全的单页应用就搭好了，下面就是具体逻辑的实现了。

GitHub 的 API 功能非常完善，完善到你完全可以用它的 API 再造一个 GitHub 出来，唯一要注意的一点就是要做好权限的验证工作。GitHub 提供了多种[验证的方式](https://developer.github.com/guides/basics-of-authentication/)，如果不进行任何验证的话，虽然 API 也可以用，但是同一个 IP 每个小时只能调用 60 次，这显然是不行的。但又不能强制每个来看博客的用户用自己的账号先 OAuth 登录一下，这个对于用户也是不能接受的。因此，我选用了最简单的权限验证方式，在请求首部中进行 Basic Authentication 验证。

首先我们要做的是申请一个 token，申请到 token 之后只要和自己的用户名拼接成 `${username}:${token}` 这样的字符串，再将这个字符串进行 base64 编码后得到一个 authString，再在请求首部加上这样一个字段：
```
Authorization: Basic ${authString}
```
然后，API 请求频率的限制就会提升到每小时 5000 次，这个对于我这个小博客来说基本上就够用了:)

但是要注意的一点是，选择 token 权限的时候，下面的勾选项一个都不要选，这样这个 token 的权限和未登录匿名用户的权限就是一样的，只有你所有公开仓库的只读权限，即使别人通过抓请求拿到这个 token 也干不了什么事情。当然，由于这个 token 是要写在代码里并且上传到仓库中的，即使这个 token 权限过大，GitHub 也能从你的 commit 中检测到这个 token，给你发一封警告邮件并自动将 token 注销，所以实际上也不会有什么危险，不得不感叹 GitHub 在安全这方面做的还是很细致的。

API 的[文档](https://developer.github.com/v3/) 非常齐全，对照着文档来用就可以，同时 GitHub 也提供了渲染 Markdown 字符串的接口，这样也不用在页面中额外引入什么前端渲染 Markdown 的库了，很方便。

另外，考虑到大天朝有一座万里长城，GitHub API 的速度有时确实不太稳定，一旦某个接口响应特别慢的话页面刷不出来也会影响到体验，这个时候前端缓存就能派上用场了。我的博客更新频率不高，像博客列表和文章详情这些写好了也不常更新，那么这些数据就可以在第一次取到以后保存到 localStorage 中，下一次用户打开页面的时候首先把 localStorage 中的数据取出来展示，做到页面秒开，同时也去请求接口，数据返回后再更新一次页面，Vue 2.x 引入了 Virtual DOM，即使全页面更新性能应该也是不错的。

至于具体的实现就不多说了，这个各人实际情况和想法都不一样，而且我现在也没写完，想到啥写啥的状态 :P，不过在基本功能功能之外，还可以做一些更深度的优化。

## PWA 一些概念的引进
现在 Web 技术发展迅猛，最近两年由 Google 为首的一些公司在主推 Progressive Web Apps 的概念，说人话就是渐进式的 Web 应用。所谓渐进式，就是使用 PWA 模式开发的应用在基本的功能之外
