## 说明
本项目是用作发布hongker.github.io里的博客文章。

## 安装
- 首先安装nodejs
```
sudo apt-get install nodejs
```

- 安装npm
```
sudo apt-get install npm
```
使用cnpm对于国内用户安装会更快
```
sudo npm install -g cnpm
```

- 安装hexo
```
sudo cnpm install -g hexo-cli
```

## 初始化
- 初始化博客
```
hexo init myblog
hexo server
```

- 发布
```
hexo clean
hexo d
```