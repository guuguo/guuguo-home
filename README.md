# 谷果之家  
- guuguo的个人博客

访问地址：https://my.guuguo.top
## 如何开发
```shell
#build public folder with watch file change 
hexo clean & hexo generate --watch


#deploy to tencent cos and build first
hexo deploy --generate
hexo generate --deploy
```
## 目录结构

```shell
# 样式修改相关

# 使用主题主目录
/themes/miracle/

# 网页的主布局文件（包含导航头，身体内容，和尾部）
/themes/miracle/layout/layout.ejs

# 网页的首页内容文件（文章列表）
/themes/miracle/layout/index.ejs

```