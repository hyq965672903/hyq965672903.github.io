---
title: Hello World
abbrlink: 4a17b156
---
Welcome to [Hexo](https://hexo.io/)! This is your very first post. Check [documentation](https://hexo.io/docs/) for more info. If you get any problems when using Hexo, you can find the answer in [troubleshooting](https://hexo.io/docs/troubleshooting.html) or you can ask me on [GitHub](https://github.com/hexojs/hexo/issues).

## Quick Start

### Create a new post

``` bash
$ hexo new "My New Post"
```

More info: [Writing](https://hexo.io/docs/writing.html)

### Run server

``` bash
$ hexo server
```

More info: [Server](https://hexo.io/docs/server.html)

### Generate static files

``` bash
$ hexo generate
```

More info: [Generating](https://hexo.io/docs/generating.html)

### Deploy to remote sites

``` bash
$ hexo deploy  test
```

More info: [Deployment](https://hexo.io/docs/one-command-deployment.html)


```java
        public static String numCheck(String fieldName, String field, Integer length, boolean allowNull, boolean allowEqualZero) {

        if (!allowNull && StringUtils.isBlank(field)) {
            return fieldName + "不能为空";
        }
        if (allowNull && StringUtils.isBlank(field)) {
            return null;
        }
        if (StringUtils.isNotBlank(field)) {
            if ((length + 1) < field.length()) {
                return fieldName + "长度不能超过 10 位";
            }
            if (!StringUtils.isNumeric(field)) {
                return fieldName + "请输入数字";
            }
            if (allowEqualZero) {
                if (Integer.parseInt(field) < 0) {
                    return fieldName + "请输入大于等于 0 数字";
                }
            } else {
                if (Integer.parseInt(field) <= 0) {
                    return fieldName + "请输入大于 0 数字";
                }
            }
        }
        return null;
    }
```



手工梗



<iframe height=700 width=1000 src="//player.bilibili.com/player.html?aid=247425240&bvid=BV19v41187fe&cid=319165979&page=1" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true"> </iframe>



{% note success %}
文字 或者 `markdown` 均可
{% endnote %}




{% label primary @text %}

{% label primary 123 %}





## 测试图片

![ne78w4](https://file.hyqup.cn/img/ne78w4.jpg)