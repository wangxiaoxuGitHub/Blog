


### 置顶

    npm uninstall hexo-generator-index --save

    npm install hexo-generator-index-pin-top --save

top: true


### 自动部署


GitHub Webhooks

![20200430115330](https://img.yjll.art/img/20200430115330.png)

``` python

from flask import Flask
from flask import request
import os


app = Flask(__name__)

@app.route('/',methods=['POST'])
def deployment():

    os.system('./deployment.sh')

    return 'Hello, World!'

if __name__ == '__main__':
    app.run(port=10080)

```

``` bash
#!/bin/bash
cd /root/Blog
git pull
rm -rf /var/www/blog/source/_posts/*
cp /root/Blog/public/* /var/www/blog/source/_posts/
cd /var/www/blog/
hexo clean
hexo g

```
