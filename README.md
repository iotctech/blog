### 1.概述
个人静态博客页面，记录深度学习、Linux操作系统、虚拟化、嵌入式等相关技术。  

[**访问页面**](https://iotctech.github.io/blog)

### 2、代码使用
#### 2.1、克隆代码
```
git clone git@github.com:iotctech/blog.git
```

#### 2.2、安装hexo-cli
```
sudo apt-get install nodejs
sudo apt-get install npm

sudo npm install hexo-cli -g
```

#### 2.3、安装依赖
```
cd blog
npm install

# 搜索支持
npm install hexo-generator-searchdb --save
# git
npm install hexo-deployer-git --save

npm install hexo-renderer-pug --save
npm install hexo-renderer-sass --save

# 图片
npm install https://github.com/iotctech/hexo-asset-image --save
```

### 3、git部署
已在**_config.yml**中添加部署路径，执行下面脚本会自动部署到git page
```
hexo g -d
```

### 4、添加页面
```
hexo new post [文件名]
```