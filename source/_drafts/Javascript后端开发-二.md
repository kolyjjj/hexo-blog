title: Javascript后端开发(二)
categories:
- Tech
tags: 
- NodeJS
---

本文接着上文[《Javascript后端开发学习》](http://koly.me/2016/01/26/Javascript%E5%90%8E%E7%AB%AF%E5%BC%80%E5%8F%91%E5%AD%A6%E4%B9%A0/)。将继续记录学习NodeJS后端开发的“流水”。这篇主要是记录comments的CRUD。<!-- more -->
## 编码前的思考
主要有两个方面：
* 数据库的表设计，即post和comment的关系
* api的设计，即url的设计

先说第一个，表设计。详细的表设计流程可以参考[《数据库设计基础》](http://koly.me/2015/11/03/%E3%80%90%E8%AF%91%E3%80%91%E6%95%B0%E6%8D%AE%E5%BA%93%E8%AE%BE%E8%AE%A1%E5%9F%BA%E7%A1%80/)。这里明显有两个实体，一个是post，另一个是comment。实体之间的关系一般有一对一，一对多，多对多。由于一个post可以对应多个comment，而同一个comment不能同时属于两个不同的post，所以post和comment的关系是一对多关系，即一个post对应多个comment。如果在关系数据库中，则需要建立一张post表，一张comment表，然后在comment表中使用一列来存储post的id以表示该条记录所属于的post。在mongodb中对一对多的实现由两种方式:
* 嵌入式文档([embedded document](https://docs.mongodb.org/manual/tutorial/model-embedded-one-to-many-relationships-between-documents/))
* 文档引用([document references](https://docs.mongodb.org/manual/tutorial/model-referenced-one-to-many-relationships-between-documents/))。

嵌入式文档的意思就是说在表示post的一个json里面有一个数组，然后这个数组的每个元素就是一个comment。文档引用就是，post是一个文档，而comment是另一个文档，post文档有一个`_id`，在comment文档里面增加一个属性，然后使用post的那个`_id`来表示该comment属于哪个文档。第一种方式在读取post的时候就可以读取该post对应的comments，对于一个post，只需要一次查询就可以得到该post的内容和对应的所有comments。第二种方式需要两次查询才能拿到post及对应的comments。这里考虑到从用户的角度来说post相对comment来说更为重要一些，有的时候甚至没有comment也没有太大的影响。同时如果comments过多的话可能会影响响应时间，而用户可能希望首先看到post的内容。所以从用户的行为习惯出发，更倾向于将post和comment分开，使用分开的查询，这样可以尽可能保证post的相应速度，优先显示出post。技术上来说，使用一个单独的请求来获取comment，在comment变多的时候可以考虑分页处理，而不会影响post，甚至于将comment作为单独的一个模块独立出去，比如[DISQUS](https://disqus.com/home/explore/)。所以最后决定采取第二种方式。

api的设计，就遵循Restful API的设计风格，考虑到两个资源的关系，comment的主api是`/api/post/<post_id>/comments/`，对应的CRUD就是POST, GET, PUT, DELETE。

那么想到这里就可以试着开始开发了。
