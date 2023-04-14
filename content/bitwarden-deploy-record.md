+++
title = "记录一下折腾 Bitwarden 密码管理器"
date = 2023-04-14
+++

近来萌生想要替换 1Password 的想法，于是有想起来之前曾经尝试使用一段时间自部署的 Bitwarden 服务，由于当时购买的腾讯云机器后续没有继续续费以及国内域名需要备案等问题，遂放弃。

本着“降本增效”的原则，看上了 Google cloud compute 的免费额度。 此处记录一下部署的信息，以免日后遗忘。

部署直接使用了开源项目： https://github.com/dadatuputi/bitwarden_gcloud

包含如下能力：

  1. 部署了 Rust 写的 server 服务；
  2. 使用 Caddy 来为域名自动 Https;
  3. fail2ban;
  4. 自动升级镜像；
  5. 阻止来自特定国家的流量。

上述第 5 点比较特殊，因为 Google cloud 的免费套餐里面，不包含来自 CN / AU 的流量，为了避免意外的费用，所以阻止了这两个地方的访问。

为了解决不能直接访问的问题，又再 Cloudflare workers 服务上使用代理的访问来访问 Bitwarden 服务，以此来绕过这个问题。

目前访问方式如下：

  原本 Bitwarden 服务域名： https://p.xxx.xxxx , 无代理不能直接访问，Cloudflare workers 域名： https://pp.xxx.xxxx

顺便记录一下 workers 的代码：

```javascript
addEventListener('fetch', event => {
    event.respondWith(fetchAndApply(event.request));
})

async function fetchAndApply(request) {
    let response = null;
    let method = request.method;

    let url = new URL(request.url);
    let url_hostname = url.hostname;
    url.protocol = 'https:';
    url.host = 'p.xxxx.xxxx';

    let request_headers = request.headers;
    let new_request_headers = new Headers(request_headers);
    new_request_headers.set('Host', url.host);
    new_request_headers.set('Referer', url.protocol + '//' + url_hostname);

    let original_response = await fetch(url.href, {
        method: method,
        headers: new_request_headers,
        body: request.body
    })
  
    let original_response_clone = original_response.clone();
    let original_text = null;
    let response_headers = original_response.headers;
    let new_response_headers = new Headers(response_headers);
    let status = original_response.status;

    new_response_headers.set('Cache-Control', 'no-store');
    new_response_headers.set('access-control-allow-origin', '*');
    new_response_headers.set('access-control-allow-credentials', true);
    new_response_headers.delete('content-security-policy');
    new_response_headers.delete('content-security-policy-report-only');
    new_response_headers.delete('clear-site-data');

    original_text = original_response_clone.body
    response = new Response(original_text, {
        status,
        headers: new_response_headers
    })

    return response
}

```
