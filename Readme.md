# Guide
## Init
`git clone https://github.com/SakaZhang/dbgroup.git`

用本地logseq打开这个仓库，设置git autocommit
## Push
在设置git autocommit后，logseq会自动进行`git add && git commit -m "Logseq auto commit"`

进入仓库目录进行`git push`即可触发Github Action, 整个CI大概需要7-10分钟，[dbgroup.club](https://dbgroup.club)会自动完成更新

**CI只构建了master分支，为避免冲突，建议每次推送前先拉取**

## 图床
Logseq会把pages中插入的图片保存在`assets`目录下,但是由于是基于git做的同步和部署，推荐使用[图床](https://www.imgurl.org/)，然后将图片以上传图片后md链接形式贴在Logseq中

## Logseq语法
Logseq支持[标准md语法](https://zhuanlan.zhihu.com/p/450086994), [块语法](https://zhuanlan.zhihu.com/p/450069864)
