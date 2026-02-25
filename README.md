# WeChat-articles-spider



## 概述：

在一次更新团队主页时发现公众号推文已经长时间没有更新到官网上，然而手动copy文字+图片显然不太现实（三十多篇能累死） 因此有了这个小工具爬取微信公众号文章和图片



> 此工具仅用于交流学习,也欢迎各位贡献和star~



## 功能：

- 此工具可以爬取微信公众号文章并保存为`markdown`格式
- 爬取的图片会存放在`/images/文章标题/`目录下
- 可导入`urls.txt`文件以实现批量处理
- 能够避免微信图片防反爬处理



## 安装



```
git clone https://github.com/zer0ptr/wechat-official-accounts-article-spider.git

pip install -r requirment.txt
```



## 工具使用



首先将需要处理的文章URL批量复制到`urls.txt`内

然后直接

```
python ./main.py 
```
    



