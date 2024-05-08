# 安装

```shell
# 安装docsify
npm i docsify-cli -g

# 项目目录
cd yxyyyt.github.io

# 初始化，生成三个文件nojekyll、index.html、README.md
docsify init ./docs

# 启动服务，访问 http://localhost:3000/#/
docsify serve ./docs
```



# 问题

- docsify启动异常 enquirer unexpected token

  enquirer 2.4.0 cannot run in Node.js <= 16，需要升级 NodejS 版本
