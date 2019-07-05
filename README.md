# blog-hexo
My blog hexo configuration. liusheng.github.io

## Re-deploy when use new computor

### Install hexo
```
npm install hexo-cli -g
```

### Clone blog-hexo repo
```
git clone https://github.com/liusheng/blog-hexo
```

### Install blog
```
cd blog-hexo
npm install
npm install hexo-deploy-git --save
hexo clean
hexo d -g
```

