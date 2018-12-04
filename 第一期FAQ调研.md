---
title: 智能家居项目FAQ调研
date: 2018-11-23 —— 2018-11-30
categories:
- 智能家居项目
- FAQ问答系统
tags:
- 项目
Team: SCU-B418
---
* [目录](#0)
   * [智能家居FAQ系统架构思考](#1)
   * [中文聊天机器人资源](#2)
   * [项目数据集进展与规划](#3)
   * [Reference](#4)
   
<h1 id="1">一、智能家居FAQ系统架构思考</h1>  

## 什么是FAQ?  
**简介**：在智能客服的业务场景中，对于用户频繁会问到的**业务知识类问题**的自动解答（以下简称为FAQ）是一个非常关键的需求，可以说是智能客服最为核心的用户场景，基本上来说，就是用户使用智能客服系统，提问了一个业务知识的问题，系统需要在知识库里找到最合适的那一个答案，且一般来说，知识库都是人工事先编辑好的。    
**例子**：比如10086的在线智能客服，用户提问“如何查询话费”，那系统可以自动给出一个对应的知识“请您向10086号码发送‘HF’短信，即可查询当前话费”  
**特点**：  
1. 在这个场景中，知识的域是**垂直封闭**的，并不是开放的，只取决于开发这个智能客服的公司或者个人的需要支持的业务范围是多大。  
2. 由于是垂直领域，知识的变化并不会过于频繁，且知识库其实一般都是一个已经编辑好的库，一般由问题与答案这样的pair对组成，而不是那种很复杂的图结构或者关联表结构。如“如何查询话费”与“请您向10086号码发送 ‘HF’短信，即可查询当前话费”这样的一个文本pair对构成。一般来说pair对的数量都是几十到几百几千不等，不会特别大。
3. **回答相对简单**，不需要太多的推理和解析，只需要知道是知识库里的哪一个回答即可，某种程度上可以看做是一个**匹配问题**甚至是分类问题。
## 系统架构
参考百度AnyQ Framework  
![baidu]( https://github.com/WenRichard/Intelligent-Furniture-FAQ/raw/master/Image/百度AnyQ.png "百度AnyQ Framework")  
百度AnyQ Framework思路：  
召回 + 排序  
分为**字面检索模块（Term Retrieval）**和**语义检索模块（Semantic Retrieval）**
1. 提前把历史文本语料索引在Elastic search这样的搜索引擎中，或者可以把相似模型对于问题的建模稠密向量用Faiss或者annoy工具索引起来  
2. 当来了一个新问题的时候，就通过索引去搜索召回出历史语料中比较粗粒度的最相似TopK（这个时候TopK就可以降低到10或者20）问题即可  
3. 用精细化的耗时的复杂语义模型去进一步匹配
* ### **智能家居FAQ系统大体思路**
  1.目标：自动回答用户提出的家电领域问题，也可以和用户“简单闲聊”  
  2.类型：检索式QA-bot，偏向于智能能客服方向  
  3.准备：搜集家电领域的QA、闲聊语料  
  --家电领域QA: 搜集长虹家电等家电网站QA+拍脑袋制造QA+长虹集团提供QA等(**要求答案很简单，且比较清晰**)，**百度知道模块的百度类似问题推荐**     
  --闲聊语料： 参考之前收集的小黄鸡等中文对话语料，**百度知道模块的百度类似问题推荐**   
  4.模型：  
  --用传统NLP模型捕捉词形、浅层语义信息 -- 做召回，缩小搜索空间  
  --用深度模型捕捉句子结构、深层语义信息 -- 对召回的item做rank  
  **召回模型**  
  1. 分词、词性修正：对词典做定制，比如“xxx”等词在本场景中应该是一个专有名词，需要修改这些词的分词和词性  
  2. 去停用词：取常用停用词和语料中的一些高频词、保留“你、我”等一些在此场景中有实际意义、有区分度的代词等
  3. 相关性扩展：当搜索“XX漂亮”时应该也把“XX美丽”等词形上不同但语义相近的问题召回，可以人肉用近义词典做细粒度定制，或者用word2vec找距离相近的词，虽然严格上不是“近义词”、而是“相关词\关联词”，比如它认为“男人”和“女人”相近  
  4. 转为BOW：把原问题转为高维稀疏词频向量  
  5. --用tfidf weight代替词频做为词袋中各词对应的value（问题比较短的情况下）  
     --用lsa\lda等映射到低维空间（问题比较长，且不在乎信息损失）  
  6. 检索：当用户提问时，通过以上五个步骤把问题映射为一个高维稀疏向量，然后从问题库中召回与其cosin距离最近的n个问题  
     -- 问题： 若每次用户提问时都与知识库中几十万个高维稀疏向量中算cosin 是不现实的、时间复杂度不可接受  
     -- 解决方法： Locality Sensitive Hashing（LSH）这种hash可以把距离很近的数据以较高的概率映射成同一个hash值  
     -- 选择：   
       1. sklearn中的LSHForest，但它检索慢、recall 低  
       2. Facebook research的pysparnn，但它也有个缺点：建index慢，好在建index是一次性的、建好后用cPickle持久化，以后用时load就好了  
       
    上述效果：  
    --当用户提问时，**已经能召回n个比较相似的候选集了**，这里的”相似“是指topic级别的，比如”你 真 漂亮＝你 真 漂亮 ≈ 你 真 美丽“也就是能捕捉topic、词形、”近义词“（姑且叫近义词吧、其实更应该叫 相关词/关联词）信息，但是，还不能捕捉semantic级别的信息，以及句子结构方面的信息，比如”你认识孙萌吗”？与“孙萌是谁” 相似度只有0.573，但“智能化的目标是什么”与“智能家居是什么”的相似度却有0.733，缺点显而易见，于是引入语义模型  
  **语义模型**
    1. encoder： 把各问题encode成低维向量  
     模型： Siamese LSTM + attention， IARNN等  
    2. 相似度计算：dot，cosine等  
    
  **关于简单多轮对话**  
  1. 用NER识别人名、地名、机构名、部分专有名词 作为上下文的topic并记录到session  
  2. 当在用户的问题中找不到主语or主体词时，用session中记录的专用名词做补充，比如：用户只问“怎么用”，此时为了知道用户在问什么东西怎么用，就用session中记录的“优惠券”做补充，得到“优惠券怎么用”
     
## 提问处理模块  
* ## **Part1 Sentence pair matching**  
  **词语多义性问题**  
  导致问题：无法正确识别问题，导致答案召回准确率降低    
  场景：例如“中国银联”=“银联”，“中国农业银行”=“农行”      
  **解决方法**：构造词语对等列表或者词语相关性表    
  一、词义相同的两个词可以以较高的关联性进行识别，从而提高答案的准确性    
  二、词义较为相近的两个词关联，从而提高相近答案的输出（建议的形式输出，例如我们没有发现XXX的答案，建议查看YYY的答案），提高用户对于会话智能的认可     
  
  **未登录词问题，OOV问题**    
  **导致问题**：无法与库中的问题进行正确匹配  
  场景：暂无   
  **解决方法**：暂无   

* ## **Part2  Sentence Input**
  **不正常的用户输入问题**  
  导致问题：影响问题识别      
  场景：很多用户是因为存在问题或者发生故障来寻找客服服务的，本身带有消极的情绪，例如愤怒、着急、失望等。因此句子输入可能会带有某些包含情感的词语，比   如：他妈的等  
  **解决方法**（提升用户体验）： 
  一、感知用户的情绪并给予安慰； 设置安慰词表    
  二、建立“情感”词表，替换原询问句    
  三、定期数据统计，跟踪出现用户失望的会话，并从设计和数据方面进行改善   
  
* ## **Part3 Chatbot Response**   
  **如何准确捕捉问的类型**  
  导致问题：无法将用户意图正确分类，就无法将查询输入系统  
  场景：用户会问“为什么？”，“是什么？”，“怎么做？”以及这三个类型的引申词，如“如何？”等  
  **解决方法**：  
  一、先对用户意图进行大概的分类  
  二、”是什么？“之类的问题考虑转向KB-QA，”为什么“，”怎么做“转向FAQ  
  
## 检索模块  
待更新
  
## 答案抽取模块
待更新

<h1 id="2">二、聊天机器人资源（开源模型）</h1>  

|标题|时间|类型|
|-|-|-|
|[基于中文知识库的聊天机器人](https://github.com/DouYishun/KB-QA)|2018/11/29|技术|
|[一个汇总聊天机器人的网站](http://www.tensorflownews.com/category/chatbot/)|2018/11/29|技术|
|[自己做聊天机器人网站](http://www.shareditor.com/bloglistbytag/?tagname=%E8%87%AA%E5%B7%B1%E5%8A%A8%E6%89%8B%E5%81%9A%E8%81%8A%E5%A4%A9%E6%9C%BA%E5%99%A8%E4%BA%BA)|2018/11/29|技术|
|[AI 产品经理的成长之路（ChatBot 方向）](https://www.jianshu.com/p/b9b3d1b0f1e1)|2018/11/29|综述|
|[智能客服FAQ问答任务的技术选型探讨](https://zhuanlan.zhihu.com/p/50799128)|2018/12/2|综述|

|标题|时间|框架|类型|等级|
|-|-|-|-|-|
|[FAQ_Chatbot_Rasa](https://github.com/ajinkyaT/FAQ_Chatbot_Rasa)|2018/12/04|Rasa|FAQ & retrieval|重要|
|[FAQ_Chatbot](https://github.com/Dk20/FAQ_Chatbot)|2018/12/04|Null|FAQ & retrieval|一般|
|[nlp-faq-chatbot](https://github.com/vstiern/nlp-faq-chatbot)|2018/12/04|Null|FAQ & retrieval|一般|
|[FAQ-chatbot-for-energym](https://github.com/yafeunteun/FAQ-chatbot-for-energym)|2018/12/04|Rasa|FAQ & retrieval|重要|
|[NTU_FAQs_Chatbot](https://github.com/trangnm58/NTU_FAQs_Chatbot)|2018/12/04|Null|FAQ & retrieval|与项目相符|
|[retrieval-chatbot](https://github.com/simrat65/retrieval-chatbot)|2018/12/04|Null|FAQ & retrieval|一般|
|[FAQ-Chat-Bot](https://github.com/donowhy/FAQ-Chat-Bot)|2018/12/04|Null|FAQ & retrieval|一般|
|[Factoid-based-Question-Answer-Chatbot](https://github.com/vaibhawraj/Factoid-based-Question-Answer-Chatbot)|2018/12/04|Null|FAQ & retrieval|与项目相符|
|[retrieval_chatbot_new](https://github.com/ricosr/retrieval_chatbot)|2018/12/04|Null|FAQ & retrieval|与项目相符|
|[基于检索的简单问答系统](https://github.com/ShepherdX/retrieval_chatbot)|2018/12/04|Null|FAQ & retrieval|与项目相符|
|[retrieval_chatbot2](https://github.com/llamazing/retrival-chatbot)|2018/12/04|Null|FAQ & retrieval|与项目非常相符|
|[Chatbot-using-tensorflow](https://github.com/sujit0892/Chatbot-using-tensorflow)|2018/12/04|Null|FAQ & retrieval|一般|
|[campus-chatbot](https://github.com/fuzzzycoder/campus-chatbot)|2018/12/04|Null|FAQ & retrieval|一般|



<h1 id="3">三、项目数据集进展</h1>  

* #### 时间：2018/12/1 ##  
  进展：1.爬取了[长虹家电官网](http://cn.changhong.com/fw/cjwt/czzd/) 首页>服务>常见问题>操作指导>下的家电信息QA对，并对噪声数据进行删选  
  规划：1.爬取以下数据：首页>服务>常见问题>日常保养， 首页>服务>常见问题>故障排除， 首页>服务>自助排障，并对爬取数据进行去噪  
  
<h1 id="4">四、Reference</h1>  

* #### [浅谈聊天机器人 ChatBot 涉及到的技术点 以及词性标注和关键字提取](https://blog.csdn.net/smilejiasmile/article/details/80967630)
* #### [Chatbots 中对话式交互系统的分析与应用](https://blog.csdn.net/RA681t58CJxsgCkJ31/article/details/79738338)



