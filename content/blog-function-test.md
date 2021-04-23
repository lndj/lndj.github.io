+++
title = "博客功能测试"
date = 2021-03-10
category = "测试"
tags = [
  "placeholder posts",
  "prime-number posts",
]
+++
本文尽量测试博客的各项功能。

<!-- more -->

下面开始测试：

{% figure(link="https://github.com/lndj", src="https://camo.githubusercontent.com/400c4e52df43f6a0ab8a89b74b1a78d1a64da56a7848b9110c9d2991bb7c3105/68747470733a2f2f696d672e736869656c64732e696f2f62616467652f4c6963656e73652d47504c76332d626c75652e737667", alt="sample alt text") %}
This is some figure shortcode.
{% end %}


## Some Youtube video here

{{ youtube(id="linlz7-Pnvw") }}



```python
print('hello world')
```

下面的是 katex 支持测试：

{% katex(block=true) %}\KaTeX{% end %}
