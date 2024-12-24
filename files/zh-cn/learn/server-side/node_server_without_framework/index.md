---
title: 不用框架的 Node.js 服务器
slug: Learn/Server-side/Node_server_without_framework
l10n:
  sourceCommit: 8184fa218341dbb193ce6adaa1240c89fae045ec
---

{{LearnSidebar}}

本文提供了一个使用纯 [Node.js](https://nodejs.org/zh-cn/cn/) 构建的简单静态文件服务器，无需使用框架。
在目前的 Node.js 中，内置的 API 和几行代码几乎就能提供我们所需的一切。

## 示例

使用 Node.js 构建的简单静态文件服务器：

```js
import * as fs from "node:fs";
import * as http from "node:http";
import * as path from "node:path";

const PORT = 8000;

const MIME_TYPES = {
  default: "application/octet-stream",
  html: "text/html; charset=UTF-8",
  js: "application/javascript",
  css: "text/css",
  png: "image/png",
  jpg: "image/jpg",
  gif: "image/gif",
  ico: "image/x-icon",
  svg: "image/svg+xml",
};

const STATIC_PATH = path.join(process.cwd(), "./static");

const toBool = [() => true, () => false];

const prepareFile = async (url) => {
  const paths = [STATIC_PATH, url];
  if (url.endsWith("/")) paths.push("index.html");
  const filePath = path.join(...paths);
  const pathTraversal = !filePath.startsWith(STATIC_PATH);
  const exists = await fs.promises.access(filePath).then(...toBool);
  const found = !pathTraversal && exists;
  const streamPath = found ? filePath : STATIC_PATH + "/404.html";
  const ext = path.extname(streamPath).substring(1).toLowerCase();
  const stream = fs.createReadStream(streamPath);
  return { found, ext, stream };
};

http
  .createServer(async (req, res) => {
    const file = await prepareFile(req.url);
    const statusCode = file.found ? 200 : 404;
    const mimeType = MIME_TYPES[file.ext] || MIME_TYPES.default;
    res.writeHead(statusCode, { "Content-Type": mimeType });
    file.stream.pipe(res);
    console.log(`${req.method} ${req.url} ${statusCode}`);
  })
  .listen(PORT);

console.log(`服务器正运行在 http://127.0.0.1:${PORT}/`);
```

### 细节

下面几行导入了 Node.js 的内部模块。

```js
import * as fs from "node:fs";
import * as http from "node:http";
import * as path from "node:path";
```

接下来是创建服务器的函数。`https.createServer` 会返回一个 `Server` 对象，我们可以通过对 `PORT` 的监听来启动它。

```js
http
  .createServer((req, res) => {
    /* 处理 http 请求 */
  })
  .listen(PORT);

console.log(`服务器正运行在 http://127.0.0.1:${PORT}/`);
```

异步函数 `prepareFile` 返回这样的结构：`{ found: boolean, ext: string, stream: ReadableStream }`。如果文件是可服务的（服务器进程可以访问且未发现路径遍历漏洞），我们将返回 HTTP 状态 `200` 作为表示成功的 `statusCode` （否则我们将返回 `HTTP 404`）。请注意，其他状态代码可以在 `http.STATUS_CODES` 中找到。对于 `404` 状态，我们将返回 `'/404.html'` 文件的内容。

所请求文件的扩展名将被解析并转为小写。然后，我们在 `MIME_TYPES` 集合中搜索正确的 [MIME 类型](/zh-CN/docs/Web/HTTP/MIME_types)。如果没有找到匹配的类型，我们将使用 `application/octet-stream` 作为默认类型。

最后，如果没有异常，我们将发送所请求的文件。`file.stream` 将包含一个 `Readable` 流，并被导入到 `res` 流中（`Writable` 流的实例）。

```js
res.writeHead(statusCode, { "Content-Type": mimeType });
file.stream.pipe(res);
```
