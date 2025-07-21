---
home: true
heroText: Yim's Blog
tagline: 
# heroImage: /hero.png
# heroImageStyle: {
#   maxWidth: '600px',
#   width: '100%',
#   display: block,
#   margin: '9rem auto 2rem',
#   background: '#fff',
#   borderRadius: '1rem',
# }
bgImageStyle: {
  height: '450px'
}
isShowTitleInHome: false
actionText: Guide
actionLink: /views/other/guide
features:
- title: Yesterday
  details: 开发一款看着开心、写着顺手的 vuepress 博客主题
- title: Today
  details: 希望帮助更多的人花更多的时间在内容创作上，而不是博客搭建上
- title: Tomorrow
  details: 希望更多的爱好者能够参与进来，帮助这个主题更好的成长
---

## 流程

1.在blogTest文件夹下，编写好所有的md文档之后执行以下脚本进行编译

```shell
npm run build
```

2.把public下的所有文件复制到blog文件夹

3.使用git提交到gitee

```shell
git add .
git commit -m "日志"
git pull
git push origin master
```



## 使用框架

vuepress