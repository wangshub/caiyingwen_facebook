# **Python**蔡英文facebook主页分析(by神奇的战士)

- 博客地址：[https://wangshub.github.io/](https://wangshub.github.io/)
- 公众号：神奇的战士
- 拒绝转载

用[Facebook Graph API](https://developers.facebook.com/tools/explorer/?method=GET&path=tsaiingwen%2Fposts&version=v2.11)和情绪分析API对蔡英文Facebook主页进行统计分析。

## 1. 说明

[蔡英文](https://baike.baidu.com/item/%E8%94%A1%E8%8B%B1%E6%96%87/353?fr=aladdin)2016年5月20日，蔡英文正式就任台湾地区领导人，成为台湾地区首位女性领导人。

最近台湾省地区新闻主要有：
> 新闻来源： 人民网
> [坚决惩治电信诈骗犯罪 切实维护两岸同胞利益](http://tw.people.com.cn/n1/2017/1222/c14657-29722289.html)
> [两岸学者评新党人士被调查事件：民进党当局逆流而动终将自掘坟墓](http://tw.people.com.cn/n1/2017/1221/c14657-29720989.html)
> [台民众高呼“醒来”，蔡英文不能继续装睡](http://tw.people.com.cn/n1/2017/1126/c14657-29668078.html)
> ...

**但是真实的台湾同胞们是如何看待她的执政表现呢？**

## 2. 实现工具

如果是直接爬取脸书的主页，需要进行模拟登陆，反爬虫，代理，验证等等一系列的操作。幸好脸书开放出了图API，可以在一定的请求限制下对脸书上的数据进行访问。注意在多线程请求API的时候，不应该请求的太快，否则会被系统封禁一段时间(不要问我为什么-_-)。

**目前为止使用了如下这些工具：**

- python 2.7
- [Facebook Graph API](https://developers.facebook.com/tools/explorer/?method=GET&path=tsaiingwen%2Fposts&version=v2.11)
- 情感分析API
- python 词云
- [python 中文jieba分词](https://github.com/fxsjy/jieba)
- python Pandas
- python 多线程

## 3. 数据处理

### 3.1 posts

首先测试脸书[Facebook Graph API](https://developers.facebook.com/tools/explorer/?method=GET&path=tsaiingwen%2Fposts&version=v2.11)，对蔡小姐的post进行访问，

**curl测试脚本**

```shell
curl -i -X GET \
 "https://graph.facebook.com/v2.11/tsaiingwen/posts?access_token=xxxxxxxxxxxxxxxxx"
```

**返回示例**

```json
"data": [
    {
      "created_time": "2017-12-24T11:50:06+0000",
      "message": "蔡想想🐱祝福大家聖誕快樂🎅
        #MerryChristmas",
      "id": "46251501064_10154820163381065"
    },
    ...
    ...
],
"paging": {
    "cursors": {
      "before": "xxxxxx",
      "after": "xxxxx"
    },
    "next": "xxxxxxxxxxxxxxxxxxxxxx"
  }

```
可以观察到，脸书的每一个post都对应了一个唯一的**id**，由于post的数量是在太多，所以一次请求无法完整获取。根据**next**可以得到下一页的post，直到**next**为空时，表示所有的post获取完毕。

根据以上原理，我获取了蔡小姐从开通脸书第一天起到今天，发的每一条post。

![](https://ws1.sinaimg.cn/large/c3a916a7gy1fmsa0xjo5fj21py0ht0ul.jpg)

- 横坐标：时间
- 纵坐标：每天发文数量

自 *2008-10-22T13:55:20+0000*蔡小姐发了第一条post以来，一共发了**4120**篇状态，基本上在脸书上还是非常活跃的，在2012年最多一天发送了24条状态，成功刷屏。


### 3.2 comments

与 **3.1**节类似，每一个post下都会有网友进行评论，那么如何获取所有评论?参考图谱API文档，利用测试脚本

**curl测试脚本**

```shell
curl -i -X GET \
 "https://graph.facebook.com/v2.11/46251501064_10154729068451065/comments?access_token=xxxxxxxxxxxx"
```

**返回示例**

```json
{
  "data": [
    {
      "created_time": "2017-11-13T07:15:25+0000",
      "message": "XXXXXXXX",
      "id": "10154729068451065_10154729097936065"
    },
    ...
    ...
  "paging": {
  "cursors": {
      "before": "MTQyNQZDZD",
      "after": "MTM5MQZDZD"
    },
    "next": "https://graph.facebook.com/v2.11/46251501064_10154729068451065/comments?access_token=xxxxxxxx&pretty=0&limit=25&after=MTM5MQZDZD"
  }

```
每一条评论都对应着唯一的**id**，**next**字段是下一页的评论内容。可以通过设置，选择一夜最多显示100条评论。以此逐级获取所有的评论。

![](https://ws1.sinaimg.cn/large/c3a916a7gy1fmsadx87t0j20m80goacm.jpg)

- 横坐标：时间
- 纵坐标：每条状态对应的评论数量

一共爬取了**1830322**条网友评论，最多评论数是**23630**条。其中几次出现了较大值，原因应该是前几次大陆网友自发组织的Facebook远征军去进行**友好访问**了。具体内容可以接下来对这几次的峰值进行详细分析。

> **相关新闻**：
> [帝吧“远征”facebook｜一场表情包大战的爱国交流](http://news.163.com/16/0122/19/BDV5H0O200014SEH.html)
> [如何评价李毅吧 2016 年 1 月 20 日「出征」Facebook？](https://www.zhihu.com/question/39663757)

## 4. 数据分析

### 4.1. 蔡英文主页分析

一共获取了蔡小姐的**4120**状态，对json的message字段进行提取，将所有的状态的文字保存进行词云分析，看哪些词汇出现的频率最高。

1. 首先利用Pandas对状态的结构数据进行保存；
2. 读取Pandas表格，获取所有的状态文字；
3. 利用jieba中文分词库，对所有的文字进行分割；
4. 显示，保存图片；

![蔡小姐词云](https://ws1.sinaimg.cn/large/c3a916a7gy1fmsaval30qj21041taaom.jpg)


### 4.2. 蔡英文评论分析 

从蔡小姐的post的所有评论当中，我找出了一条评论最多的状态，共有**23630**条评论，对应id为`46251501064_10154244975341065`，读取对应数据文件，利用词云分析可得

![](https://ws1.sinaimg.cn/large/c3a916a7gy1fmt0ape4h1j20b405kt9b.jpg)

看来台湾网友也十分注意**安全开车**，其实这条post的评论区被台湾网友刷屏了，看来怨气挺重呢，哈哈哈哈，霸屏具体内容是
```
 1.政府請正視目前台灣改裝汽機車問題！
 排氣管及改裝品可以合法製造 合法販賣 合法進口但裝載車上就不合法 這是什麼邏輯 政府要課稅又要開罰單又是什麼想法？
 排氣管或車上零件是原廠被惡意檢舉驗車那是否能跟監理單位或環保署拿今日上班請假損失？
 2.環保局 監理站 警察執法單位 專業度嚴重不足 原廠排氣管也開單 叫民眾到監理單位驗車 當做民眾都很有時間？
 3.請提供可比照國外變更車體，如重機行李箱、遮陽板、避震、制動煞車系統在不影響行車安全的部份合乎法規
 4.如民眾遭受到檢舉達人惡意檢舉，因此需要請假驗車，若屬於惡意檢舉，政府需要支付民眾請假之當天工資
 蔡??...您不是希望台灣能跟世界接軌，那請您重視汽機車改裝合法性與可變更性，在不影響行車安全與噪音的> 情況下，請把檢舉改裝還於司法單位執行，才不構成擾民。
 
```
既然这条被刷屏了，那就换成最新的一篇post，看看网友又关心啥问题。。。
截止爬取脸书时，最新一条博客是：

```
你有吃過越南生春捲、香蘭娘惹糕或是薑黃飯嗎？它們是來自東南亞各國的美食，現在也是台灣的美食。
今天是國際移民日，前幾天，我邀請了幾位新移民的好朋友來到總統府，一起準備午餐。在這場午餐的約會中，他們和我分享來到台灣生活的點點滴滴，也給我很多建議。
謝謝你們來到台灣，讓我們的社會更多元、更茁壯。祝大家國際移民日快樂！
#留言告訴我你最喜歡的新南向美食
#晚餐文",
```

蔡小姐问网友喜欢吃啥美食，我们来看看网友是如何回复的

![](https://ws1.sinaimg.cn/large/c3a916a7gy1fmt0p0kbpbj218g0m8drx.jpg)

结合最近的新党王炳忠事件，评论中出现了较多**绿色恐怖、王炳忠、白色恐怖**等高频词汇

## 5. TODO

可以分析的数据还有很多，就先分析这么多了，接下来，可以对评论进行情感分析，看下网友对蔡小姐的评论是积极还是消极的多一些。

