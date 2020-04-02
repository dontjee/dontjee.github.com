---
layout: "post"
title: "Automatic system proxy configuration for Electron applications"
description: ''
category: null
tags: []
---

Electron applications integrate and run on a user's computer like a native application. Most things 'just work' but I struggled to get system proxy settings to be detected and used by Electron without requiring manual configuration. It is possible to use system proxy settings automatically and this post lays out some options for configuration.

#### First, a quick summary of Electron processes

First, you will need to know which process the request originates from because the options are different. More detailed information on the process types can be found in the Electron documentation or [this blog post](https://cameronnokes.com/blog/deep-dive-into-electron%27s-main-and-renderer-processes/).

The __main process__ is responsible for creating the browser windows for your application. It is a node.js application and therefore uses the node.js networking stack by default. The __renderer process__ is used to run the BrowserWindow.webContents portions of the application (the web UI). The renderer is an instance of Chromium and can access the native Chromium networking APIs (like `XmlHttpRequest` or `fetch`) in addition to the node.js stack. Access to the native Chromium networking APIs is important because Chromium handles [resolving the system networking settings like proxies](https://www.chromium.org/developers/design-documents/network-settings).

#### Renderer process requests

Http requests from the renderer process are relatively straightforward. Make sure to use one of the native Chromium APIs like `fetch`. The Chromium process will take care of detecting and using the system proxy settings for you. The `fetch` interface is easy to use and works the same as it does in any other browser.

#### Main process requests

To make an http request respecting system proxy settings from the main process, you have several options to consider.

1. The first is the [built-in 'net'](https://www.electronjs.org/docs/api/net) library. This option uses the native Chromium networking stack so it has the same access to the system proxy settings as the renderer process. This API interface is painful to work with and there are not any reliable open-source packages that wrap it in a more usable interface. I have not personally pursued this option due to the difficulty of working with the API but it may evolve to be more usable.

1. Another option is to use a javascript networking library like [axios](https://github.com/axios/axios) which uses the built-in `http` library from node.js. The library you choose can be configured manually to use a proxy. To do this automatically, you can retrieve the Chromium proxy settings and pass those along. Sample code for accessing the proxy information is below:

    ```js
   // retrieve the browser session
   const session = require('electron').remote.getCurrentWindow().webContents.session;

   // resolve the proxy for a known URL. This could be the URL you expect to use or a known good url like google.com
   session.resolveProxy('https://www.google.com', proxyUrl => {

      // DIRECT means no proxy is configured
      if (proxyUrl !== 'DIRECT') {
        // retrieve the parts of the proxy from the string returned
        // the url would look something like: 'PROXY http-proxy.mydomain.com:8080'
        const proxyUrlComponents = proxyUrl.split(':');

          const proxyHost = proxyUrlComponents[0].split(' ')[1];
          const proxyPort = parseInt(proxyParts[1], 10);

          // do something with proxy details
      }
   });
    ```

    This pattern has also been encapsulated into an NPM package [node-electron-proxy-agent](https://github.com/felicienfrancois/node-electron-proxy-agent).

1. Finally, the user can provide the proxy details and they can be configured in whatever http library being used. This is typically done via environment variables (e.g. `HTTP_PROXY`) or setting the `agent` of the request library to a proxy http agent like [tunnel](https://github.com/koichik/node-tunnel/). An example of setting the agent on an axios client is below:

```js
const agent = tunnel.httpsOverHttp({
    proxy: {
        host: 'http-proxy.mydomain.com',
        port: 8080,
    },
});
const axiosClient = axios.create({
    ...,
    httpsAgent: agent,
});
```

#### Wrap up

I prefer to keep the configuration as automatic as possible, so I prefer a combination of `fetch` for renderer processes and automatic configuration based on the Chromium session for the main process.