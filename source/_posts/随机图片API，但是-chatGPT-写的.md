---
title: 随机图片API，但是chatGPT写的
date: 2023-06-18 03:12:48
tags: [chatGPT,API,随机图片]
categories: coding
---

## 前言

@yltx 这一阵在升级他的博客，需要实现一个从 GitHub 仓库中返回随机图片的 API ，从而实现博客背景的随机更换。在拿着 GitHub 官方的 `octokit` 手撸了一会 cloudflare worker 之后，我懒癌发作，于是开始尝试让 chatGPT 帮我完成这个简单的API。

*篇幅所限，省略了一部分不重要的过程。*

## 编写

第一次用chatGPT写程序，为了逐步引导它，我是这么开头的：

> 编写一个 cloudflare worker 脚本，让我可以获取 GitHub指定 repo 下的目录结构

chatGPT 是这么回答我的：

> 当然！下面是一个示例的 Cloudflare Worker 脚本，可以用于获取 GitHub 指定仓库的目录结构：
> 
> ```javascript
> addEventListener('fetch', event => {
>   event.respondWith(handleRequest(event.request))
> })
> 
> async function handleRequest(request) {
>   const repoOwner = 'OWNER'; // 替换为你的仓库所有者
>   const repoName = 'REPO'; // 替换为你的仓库名称
> 
>   // 发起 API 请求获取仓库内容
>   const response = await fetch(`https://api.github.com/repos/${repoOwner}/${repoName}/contents`);
> 
>   // 检查响应是否成功
>   if (!response.ok) {
>     return new Response(`Failed to fetch repository contents. Status: ${response.status}`, { status: response.status });
>   }
> 
>   // 解析 API 响应的 JSON 数据
>   const data = await response.json();
> 
>   // 提取目录结构信息
>   const directories = data.filter(item => item.type === 'file').map(item => item.path);
> 
>   // 返回目录结构
>   return new Response(JSON.stringify(directories, null, 2), {
>     headers: {
>       'Content-Type': 'application/json',
>     },
>   });
> }
> ```
> 
> 请将其中的 `OWNER` 替换为 GitHub 仓库的所有者，`REPO` 替换为仓库名称。完成后，将此脚本部署为 Cloudflare Worker，并将请求指向部署的 Worker 地址，即可获取指定仓库的目录结构。
> 
> 请注意，这段脚本使用 GitHub 的 REST API v3 进行请求，因此需要考虑 API 请求频率限制和身份验证等问题。如果需要更复杂的功能，例如递归获取所有子目录和文件，可能需要根据实际需求进行进一步开发。

问题出现了，@yltx 的博客部署在 GitHub 的私有 repo 中，这样直接调用 API 显然是没有权限的，于是我提示它：

> API 始终返回403

结果 chatGPT 以为我的 IP 被 block 了，开始教我如何使用代理服务（

这显然不是问题的关键，于是我又提示：

> 不是代理的问题，是不是我没有提供我的 personal token？

这次chatGPT理解了，在代码中加入了如下内容：

```javascript
const personalToken = 'PERSONAL_TOKEN'; // 替换为你的个人访问令牌

  // 构建请求的头部
  const headers = {
    'Authorization': `Bearer ${personalToken}`,
    'User-Agent': 'Cloudflare Worker',
  };
```

很好！满分！~~甚至 `User-Agent` 都是对的~~

现在，把 chatGPT 编写的 javascript 脚本用 `wrangler` 运行测试一下，效果是这样的：

![](https://blog.wpcwzy.top/uploads/20230618022642.png)

返回的 json 比较复杂，实际只需要随机一个图片并返回 url 就行了。

继续引导它：

> 好的，现在我需要你随机返回其中的一个 url 即可，注意，需要返回 raw.githubusercontent.com 域名的，以便我直接将其设置为图片的 src 属性

于是乎，代码被 chatGPT 改成了这个样子：

```javascript
  // 提取文件列表信息的直接下载链接
  const downloadLinks = data.filter(item => item.type === 'file').map(item => item.download_url);

  // 随机选择一个下载链接
  const randomDownloadLink = downloadLinks[Math.floor(Math.random() * downloadLinks.length)];

  // 返回随机选择的下载链接
  return new Response(randomDownloadLink, {
    headers: {
      'Content-Type': 'text/plain',
    },
  });
```

妙哉！很好用！

但这里又出现了一个小插曲：

@yltx 是通过 CSS 设置背景图的，而 CSS 不能实现先调用 API 再操作 DOM 的功能，所以这个 API 的需求又从返回 url 变成了直接返回图片。

小case！直接把新的需求甩给 chatGPT，一次就输出了正确的代码。

## 最终效果

代码完全体如下：

```javascript
var worker_default = {
  async fetch(request, env, ctx) {
    const repoOwner = "wpcwzy";
    const repoName = "***";
    const path = "resources/img";
    const personalToken = "***";
    const headers = {
      "Authorization": `Bearer ${personalToken}`,
      "User-Agent": "Cloudflare Worker"
    };
    const response = await fetch(`https://api.github.com/repos/${repoOwner}/${repoName}/contents/${path}`, { headers });
    if (!response.ok) {
      return new Response(`Failed to fetch repository contents. Status: ${response.status}`, { status: response.status });
    }
    const data = await response.json();
    const rawFileLinks = data.filter((item) => item.type === "file").map((item) => item.download_url.replace("https://github.com", "https://raw.githubusercontent.com"));
    const randomRawFileLink = rawFileLinks[Math.floor(Math.random() * rawFileLinks.length)];
    const imageResponse = await fetch(randomRawFileLink);
    if (!imageResponse.ok) {
      return new Response(`Failed to fetch image. Status: ${imageResponse.status}`, { status: imageResponse.status });
    }
    const imageContent = await imageResponse.arrayBuffer();
    return new Response(imageContent, {
      headers: {
        "Content-Type": "image/png"
      }
    });
  }
};
export {
  worker_default as default
};
```

浏览器直接打开的效果：

![](https://blog.wpcwzy.top/uploads/20230618023543.png)

博客中的效果大家可以直接点进 [yltx的博客](https://yltx.cf) 瞅一瞅

## 感想总结

1. chatGPT 的确拥有一定的编程能力，能起到不小的辅助作用。我平常较少使用 javascript 语言，但在 chatGPT 的帮助下，可以快速实现一些简单的功能。从速度上来说，简单的 API 让 chatGPT 直接实现也要比人类查阅文档、查阅样例代码再自己编写要快很多。

2. chatGPT 对功能的实现方式很有意思，我在自己编写时，使用了 GitHub 的 `octokit` 库，但是在云函数中使用 npm 库明显是要经过一定配置的，在运维方面稍稍麻烦一些。chatGPT 直接通过 `fetch` GitHub 的 API 来实现，较好配置，而且代码整体都实现的非常优雅，注释也很全面。

3. 这次编写随机图片 API ，没有直接给 chatGPT 说明需求而是有意引导了一下，上来没有让他直接输出图片而是先获取repo文件目录。从整个编写的体验来看，chatGPT 或许拥有不用人工引导，输入简单需求就可直接输出代码的能力，下次可以尝试一下。

最后，感谢 [pengzhile/pandora (github.com)](https://github.com/pengzhile/pandora) 实现的 chatGPT 反代，要不咱会被 openai 的网站整崩溃的（
