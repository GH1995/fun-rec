##  推荐系统的技术栈

推荐系统是一个非常大的框架，有非常多的模块在里面，完整的一套推荐系统体系里，不仅会涉及到推荐算法工程师、后台开发工程师、数据挖掘/分析工程师、NLP/CV工程师还有前端、客户端甚至产品、运营等支持。我们作为算法工程师，需要掌握的技术栈主要就是在算法和工程两个区域了，所以这篇文章将会分别从算法和工程两个角度出发，结合两者分析当前主流的一些推荐算法技术栈。

## 算法

首先我们从推荐系统架构出发，一种分法是将整个推荐系统架构分为召回、粗排、精排、重排、混排等模块。它的分解方法是从一份数据如何从生产出来，到线上服务完整顺序的一个流程。因为在不同环节，我们一般会考虑不同的算法，所以这种角度出发我们来研究推荐系统主流的算法技术栈。

<div align=center>
<img src="http://ryluo.oss-cn-chengdu.aliyuncs.com/图片image-20220409211354342.png" alt="在这里插入图片描述" style="zoom:90%;" /> 
</div>

为了帮助新手在后文方便理解，首先简单介绍这些模块的功能主要是：
- 召回：从推荐池中选取几千上万的item，送给后续的排序模块。由于召回面对的候选集十分大，且一般需要在线输出，故召回模块必须轻量快速低延迟。由于后续还有排序模块作为保障，召回不需要十分准确，但不可遗漏（特别是搜索系统中的召回模块）。目前基本上采用多路召回解决范式，分为非个性化召回和个性化召回。个性化召回又有content-based、behavior-based、feature-based等多种方式。
- 粗排：粗拍的原因是有时候召回的结果还是太多，精排层速度还是跟不上，所以加入粗排。粗排可以理解为精排前的一轮过滤机制，减轻精排模块的压力。粗排介于召回和精排之间，要同时兼顾精准性和低延迟。一般模型也不能过于复杂
- 精排：获取粗排模块的结果，对候选集进行打分和排序。精排需要在最大时延允许的情况下，保证打分的精准性，是整个系统中至关重要的一个模块，也是最复杂，研究最多的一个模块。精排系统构建一般需要涉及样本、特征、模型三部分。
- 重排：获取精排的排序结果，基于运营策略、多样性、context上下文等，重新进行一个微调。比如三八节对美妆类目商品提权，类目打散、同图打散、同卖家打散等保证用户体验措施。重排中规则比较多，但目前也有不少基于模型来提升重排效果的方案。
- 混排：多个业务线都想在Feeds流中获取曝光，则需要对它们的结果进行混排。比如推荐流中插入广告、视频流中插入图文和banner等。可以基于规则策略（如广告定坑）和强化学习来实现。

### 画像层

首先是推荐系统的物料库，这部分内容里，算法主要体现在如何绘制一个用户画像和商品画像。这个环节是推荐系统架构的基础设施，一般可能新用户/商品进来，或者每周定期会重新一次整个物料库，计算其中信息，为用户打上标签，计算统计信息，为商品做内容理解等内容。其中用户画像是大家比较容易理解的，比如用户年龄、爱好通常APP会通过注册界面收集这些信息。而商品画像形式就非常多了，比如淘宝主要推荐商品，抖音主要是短视频，所以大家的物料形式比较多，内容、质量差异也比较大，所以内容画像各家的做法也不同，当前比较主流的都会涉及到一个多模态信息内容理解。下面我贴了一个微信看一看的内容画像框架，然后我们来介绍下在这一块主要使用的算法技术。

<div align=center>
<img src="http://ryluo.oss-cn-chengdu.aliyuncs.com/图片image-20220410143333692.png" alt="在这里插入图片描述" style="zoom:90%;" /> 
</div>

一般推荐系统会加入多模态的一个内容理解。我们用短视频形式举个例子，假设用户拍摄了一条短视频，上传到了平台，从推荐角度看，首先我们有的信息是这条短视频的作者、长度、作者为它选择的标签、时间戳这些信息。但是这对于推荐来说是远远不够的，首先作者打上的标签不一定准确反映作品，原因可能是我们模型的语义空间可能和作者/现实世界不一致。其次我们需要更多维度的特征，比如有些用户喜欢看小姐姐跳舞，那我希望能够判断一条视频中是否有小姐姐，这就涉及到封面图的基于CV的内容抽取或者整个视频的抽取；再比如作品的标题一般能够反映主题信息，除了很多平台常用的用“#”加上一个标签以外，我们也希望能够通过标题抽取出基于NLP的信息。还有更多的维度可以考虑：封面图多维度的多媒体特征体系，包括人脸识别，人脸embedding，标签，一二级分类，视频embedding表示，水印，OCR识别，清晰度，低俗色情，敏感信息等多种维度。

这里面涉及的任务主要是CV的目标检测、语义分割等任务，NLP中的情感分析、摘要抽取、自然语言理解等任务。但是这部分算法一般团队都会有专门负责的组，不需要推荐算法工程师来负责，他们会有多模态的语意标签输出，主要形式是各种粒度的Embedding。我们只需要在我们的推荐模型中引入这些预训练的Embedding。

#### 文本理解

这应该是用的最多的模态信息，包括item的标题、正文、OCR、评论等数据。这里面也可以产生不同粒度的信息，比如文本分类，把整个item做一个粗粒度的分类。

这里的典型算法有：RNN、TextCNN、FastText、Bert等；

#### 关键词标签

相比文本分类，关键词是更细粒度的信息，往往是一个mutil-hot的形式，它会对item在我们的标签库的选取最合适的关键词或者标签。

这里典型的算法有：TF-IDF、Bert、LSTM-CRF等。

#### 内容理解

在很多场景下，推荐的主题都是视频或者图片，远远多于仅推荐文本的情况，这里视频/图片item中的内容中除了文本的内容以外，更多的信息其实来源于视频/图片内容本身, 因此需要尝试从多种模态中抽取更丰富的信息。主要包括分类信息、封面图OCR的信息、视频标签信息等

这里典型的算法有：TSN、RetinaFace、PSENet等。

#### 知识图谱

知识图谱作为知识承载系统，用于对接内外部关键词信息与词关系信息；内容画像会将原关系信息整合，并构建可业务应用的关系知识体系，其次，依赖业务中积累用户行为产生的实体关系数据，本身用户需求的标签信息，一并用于构建业务知识的兴趣图谱，基于同构网络与异构网络表示学习等核心模型，输出知识表示与表达，抽象后的图谱用于文本识别，推荐语义理解，兴趣拓展推理等场景，直接用于兴趣推理的冷启场景已经验证有很不错的收益。

这方面的算法有：KGAT、RippleNet等。

### 召回/粗排

推荐系统的召回阶段可以理解为根据用户的历史行为数据，为用户在海量的信息中粗选一批待推荐的内容，挑选出一个小的候选集的过程。粗排用到的很多技术与召回重合，所以放在一起讲，粗排也不是必需的环节，它的功能对召回的结果进行个粗略的排序，在保证一定精准的前提下，进一步减少往后传送的物品数量，这就是粗排的作用。

<div align=center>
<img src="http://ryluo.oss-cn-chengdu.aliyuncs.com/图片image-20220410000221817.png" alt="在这里插入图片描述" style="zoom:90%;" /> 
</div>

召回模块面对几百上千万的推荐池物料规模，候选集十分庞大。由于后续有排序模块作为保障，故不需要十分准确，但必须保证不要遗漏和低延迟。目前主要通过多路召回来实现，一方面各路可以并行计算，另一方面取长补短。可以看到各类同类竞品的系统虽然细节上多少存在差异，但不约而同的采取了多路召回的架构，这类设计考虑如下几点问题：

1. **考虑用户层面**：用户兴趣的多元化，用户需求与场景的多元化：例如：新闻需求，重大要闻，相关内容沉浸阅读等等

2. **考虑系统层面**：增强系统的鲁棒性；部分召回失效，其余召回队列兜底不会导致整个召回层失效；排序层失效，召回队列兜底不会导致整个推荐系统失效

3. **系统多样性内容分发**：图文、视频、小视频；精准、试探、时效一定比例；召回目标的多元化，例如：相关性，沉浸时长，时效性，特色内容等等

4. **可解释性推荐一部分召回是有明确推荐理由的**：很好的解决产品性数据的引入；

介绍了召回任务的目的和场景后，接下来分析召回层面主要的技术栈，因为召回一般都是多路召回，从模型角度分析有很多召回算法，这种一般是在召回层占大部分比例点召回，除此之外，还会有探索类召回、策略运营类召回、社交类召回等。接下来我们着重介绍模型类召回。

#### 经典模型召回

随着技术发展，在Embedding基础上的模型化召回是一个技术发展潮流方向。这种召回的范式是通过某种算法，对user和item分别打上Embedding，然后user与item在线进行KNN计算实时查询最近领结果作为召回结果，快速找出匹配的物品。需要注意的是如果召回采用模型召回方法，优化目标最好和排序的优化目标一致，否则可能被过滤掉。

在这方面典型的算法有：FM、双塔DSSM、Multi-View DNN等。

#### 序列模型召回

推荐系统主要解决的是基于用户的隐式阅读行为来做个性化推荐的问题，序列模型一些基于神经网络模型学习得到Word2Vec模型，再后面的基于RNN的语言模型，最先用的最多的Bert，这些方法都可以应用到召回的学习中。

用户在使用 APP 或者网站的时候，一般会产生一些针对物品的行为，比如点击一些感兴趣的物品，收藏或者互动行为，或者是购买商品等。而一般用户之所以会对物品发生行为，往往意味着这些物品是符合用户兴趣的，而不同类型的行为，可能代表了不同程度的兴趣。比如购买就是比点击更能表征用户兴趣的行为。在召回阶段，如何根据用户行为序列打 embedding，可以采取有监督的模型，比如 Next Item Prediction 的预测方式即可；也可以采用无监督的方式，比如物品只要能打出 embedding，就能无监督集成用户行为序列内容，例如 Sum Pooling。

这方面典型的算法有：CBOW、Skip-Gram、GRU、Bert等。

#### 用户序列拆分

上文讲了利用用户行为物品序列，打出用户兴趣 Embedding 的做法。但是，另外一个现实是：用户往往是多兴趣的，比如可能同时对娱乐、体育、收藏感兴趣。这些不同的兴趣也能从用户行为序列的物品构成上看出来，比如行为序列中大部分是娱乐类，一部分体育类，少部分收藏类等。那么能否把用户行为序列物品中，这种不同类型的用户兴趣细分，而不是都笼统地打到一个用户兴趣 Embedding 里呢？用户多兴趣拆分就是解决这类更细致刻画用户兴趣的方向。

本质上，把用户行为序列打到多个 embedding 上，实际它是个类似聚类的过程，就是把不同的 Item，聚类到不同的兴趣类别里去。目前常用的拆分用户兴趣 embedding 的方法，主要是胶囊网络和 Memory Network，但是理论上，很多类似聚类的方法应该都是有效的，所以完全可以在这块替换成你自己的能产生聚类效果的方法来做。

这方面典型的算法有：Multi-Interest Network with Dynamic Routing for Recommendation at Tmall等。

#### 知识图谱

知识图谱有一个独有的优势和价值，那就是对于推荐结果的可解释性；比如推荐给用户某个物品，可以在知识图谱里通过物品的关键关联路径给出合理解释，这对于推荐结果的解释性来说是很好的，因为知识图谱说到底是人编码出来让自己容易理解的一套知识体系，所以人非常容易理解其间的关系。知识图谱的可解释性往往是和图路径方法关联在一起的，而 Path 类方法，很多实验证明了，在排序角度来看，是效果最差的一类方法，但是它在可解释性方面有很好的效果，所以往往可以利用知识图谱构建一条可解释性的召回通路。

这方面的算法有：KGAT、RippleNet等。

#### 图模型

推荐系统中User和Item相关的行为、需求、属性和社交信息具有天然的图结构，可以使用一张复杂的异构图来表示整个推荐系统。图神经网络模型推荐就是基于这个想法，把异构网络中包含的结构和语义信息编码到结点Embedding表示中，并使用得到向量进行个性化推荐。知识图谱其实是图神经网络的一个比较特殊的具体实例，但是，知识图谱因为编码的是静态知识，而不是用户比较直接的行为数据，和具体应用距离比较远，这可能是导致两者在推荐领域表现有差异的主要原因。

这方面典型的算法有：GraphSAGE、PinSage等。

### 精排

排序模型是推荐系统中涵盖的研究方向最多，有非常多的子领域值得研究探索，这也是推荐系统中技术含量最高的部分，毕竟它是直接面对用户，产生的结果对用户影响最大的一层。目前精排层深度学习已经一统天下了，这是王喆老师《深度学习推荐算法》书中的精排层模型演化线路。具体来看分为DNN、Wide&Deep两大块，实际深入还有序列建模，以及没有提到的多任务建模都是工业界非常常用的，所以我们接下来具体谈论其中每一块的技术栈。

<div align=center>
<img src="http://ryluo.oss-cn-chengdu.aliyuncs.com/图片image-20220410234144149.png" alt="在这里插入图片描述" style="zoom:90%;" /> 
</div>

#### 特征交叉模型

在深度学习推荐算法发展早期，很多论文聚焦于如何提升模型的特征组合和交叉的能力，这其中既包含隐式特征交叉Deep Crossing也有采用显式特征交叉的探究。本质上是希望模型能够摆脱人工先验的特征工程，实现端到端的一套模型。

在早期的推荐系统中，基本是由人工进行特征交叉的，往往凭借对业务的理解和经验，但是费时费力。于是有了很多的这方面的研究，从FM到GBDT+LR都是引入模型进行自动化的特征交叉。再往后就是深度模型，深度模型虽然有万能近似定理，但是真正想要发挥模型的潜力，显式的特征交叉还是必不可少的。

这方面的经典研究工作有：DCN、DeepFM、xDeepFM等；

#### 序列模型

在推荐系统中，历史行为序列是非常重要的特征。在序列建模中，主要任务目标是得到用户此刻的兴趣向量（user interest vector）。如何刻画用户兴趣的广泛性，是推荐系统比较大的一个难点，用户历史行为序列建模的研究经历了从Pooling、RNN到attention、capsule再到transformer的顺序。在序列模型中，又有很多细分的方向，比如根据用户行为长度有研究用户终身行为序列的，也有聚焦当下兴趣的，还有研究如何抽取序列特征的抽取器，比如研究attention还是胶囊网络。

这方面典型的研究工作有：DIN、DSIN、DIEN、SIM等；

#### 多模态信息融合

在上文我们提到算法团队往往会利用内容画像信息，既有基于CV也有基于NLP抽取出来的信息。这是非常合理的，我们在逛抖音、淘宝的时候关注的不仅仅item的价格、品牌，同样会关注封面小姐姐好不好看、标题够不够震惊等信息。除此之外，在冷启动场景下，我们能够利用等信息不够多，如果能够使用多模态信息，能很大程度上解决数据稀疏的问题。

传统做法在多模态信息融合就是希望把不同模态信息利用起来，通过Embedding技术融合进模型。在推荐领域，主流的做法还是一套非端到端的体系，由其他模型抽取出多模态信息，推荐只需要融合入这些信息就好了。同时也有其他工作是利用注意力机制等方法来学习不同模态之间的关联，来增强多模态的表示。

比较典型的工作有：Image Matters: Visually modeling user behaviors using Advanced Model Server、UMPR等。

#### 多任务学习

很多场景下我们模型优化的目标都是CTR，有一些场景只考虑CTR是不够的，点击率模型、时长模型和完播率模型是大部分信息流产品推荐算法团队都会尝试去做的模型。单独优化点击率模型容易推出来标题党，单独优化时长模型可能推出来的都是长视频或长文章，单独优化完播率模型可能短视频短图文就容易被推出来，所以多目标就应运而生。信息流推荐中，我们不仅希望用户点进我们的item，还希望能有一个不错的完播率，即希望用户能看完我们推荐的商品。或者电商场景希望用户不仅点进来，还希望他买下或者加入购物车了。这些概率实际上就是模型要学习的目标，多种目标综合起来，包括阅读、点赞、收藏、分享等等一系列的行为，归纳到一个模型里面进行学习，这就是推荐系统的多目标学习。

这方面比较典型的算法有：ESSM、MMoE、DUPN等。

#### 强化学习

强化学习与一般有监督的深度学习相比有一些很显著的优势，首先强化学习能够比较灵活的定义优化的业务目标，考虑推荐系统长短期的收益，比如用户留存，在深度模型下，我们很难设计这个指标的优化函数，而强化学习是可以对长期收益下来建模。第二是能够体现用户兴趣的动态变化，比如在新闻推荐下，用户兴趣变化很快，强化学习更容易通过用户行为动态产生推荐结果。最后是EE也就是利用探索机制，这种一种当前和长期收益的权衡，强化学习能够更好的调节这里的回报。

这方面比较典型的算法有：DQN、Reinforcement Learning for Slate-based Recommender Systems: A Tractable Decomposition and Practical Methodology；

#### 跨域推荐

一般一家公司业务线都是非常多的，比如腾讯既有腾讯视频，也有微信看一看、视频号，还有腾讯音乐，如果能够结合这几个场景的数据，同时进行推荐，一方面对于冷启动是非常有利的，另一方面也能补充更多数据，更好的进行精确推荐。

跨域推荐系统相比一般的推荐系统要更加复杂。在传统推荐系统中，我们只需要考虑建立当前领域内的一个推荐模型进行分析；而在跨域推荐中，我们更要关心在不同领域间要选择何种信息进行迁移，以及如何迁移这些信息，这是跨域推荐系统中非常关键的问题。

这方面典型的模型有：DTCDR、MV-DNN、EMCDR等；

### 重排序

我们知道常见的有三种优化目标：Point Wise、Pair Wise 和 List Wise。重排序阶段对精排生成的Top-N个物品的序列进行重新排序，生成一个Top-K个物品的序列，作为排序系统最后的结果，直接展现给用户。重排序的原因是因为多个物品之间往往是相互影响的，而精排序是根据PointWise得分，容易造成推荐结果同质化严重，有很多冗余信息。而重排序面对的挑战就是海量状态空间如何求解的问题，一般在精排层我们使用AUC作为指标，但是在重排序更多关注NDCG等指标。

重排序在业务中，还会根据一些策略、运营规则参与排序，比如强制去重、间隔排序、流量扶持等，但是总计趋势上看还是算法排序越来越占据主流趋势。重排序更多的是List Wise作为优化目标的，它关注的是列表中商品顺序的问题来优化模型，但是一般List Wise因为状态空间大，存在训练速度慢的问题。这方面典型的做法，基于RNN、Transformer、强化学习的都有，这方面因为不是推荐的一个核心，所以没有展开来讲，而且这一块比较依赖实际的业务场景。

这里的经典算法有：MRR、DPP、RNN等；

## 工程

推荐系统的实现需要依托工程，很多研究界Paper的idea满天飞，却忽视了工业界能否落地，进入工业界我们很难或者很少有组是做纯research的，所以我们同样有很多工程技术需要掌握。下面列举了在推荐中主要用到的工程技术：

- **编程语言**：Python、Java（scala）、C++、sql、shell；
- **机器学习**：Tensorflow/Pytorch、GraphLab/GraphCHI、LGB/Xgboost、SKLearn；
- **数据分析**：Pandas、Numpy、Seaborn、Spark；
- 数据存储：mysql、redis、mangodb、hive、kafka、es、hbase；
- 相似计算：annoy、faiss、kgraph
- 流计算：Spark Streaming、Flink
- 分布式：Hadoop、Spark

上面那么多技术，我内容最重要的就是加粗的三部分，第一是语言：必须掌握的是Python，C++和JAVA中根据不同的组使用的是不同的语言，这个如果没有时间可以等进组后慢慢学习。然后是机器学习框架：Tensorflow和Pytorch至少要掌握一个吧，前期不用纠结学哪个，这个迁移成本很低，基本能够达到触类旁通，而且面试官不会为难你只会这个不会那个。最后是数据分析工具：Pandas是我们处理单机规模数据的利器，但是进入工业界，Hadoop和Saprk是需要会用的，不过不用学太深，会用即可。

## 总结

本文从算法和工程两个角度分析了推荐系统的一个技术栈，但是还有很多方向遗漏，也有很多方向受限于现在的技术水平深度不够和有错误的情况，后续会不断补充和更正。

所以技术栈我列出的是一个非常广度的技术，实际上每一个技术钻研下去都需要非常多时间，而且不一定是你实际工作中会遇到的，所以不要被那么多技术吓到，也要避免陷入技术细节的海洋中。

我和非常多的大厂面试官讨论过技术深度和广度的问题，得出来的结论是对于入门的推荐算法工程师而言，实际上深度和广度的要求取决于你要去的组，有些组有很深的推荐技术沉淀，有很强的工程师团队，这样的组就会希望候选者能够在某个方面有比较深入的研究，这个方面既包含工程方面也包含研究方面。但是如果是比较新的组、或者技术沉淀不深、推荐不是主要任务的组，对深度要求就不会很高。总而言之，我认为对于应届生/实习生来说，在推荐最重要的工程技术/研究方向，至少在召回和排序模块，需要选一个作为方向，是需要较深钻研。对于其他技术/研究方向需要有一定了解，比如可以没用过强化学习，但是要知道强化学习能够在推荐中解决什么问题，剩下的可以等到真实**遇到需要后再去学习**。



**参考资料**

- [万字入门推荐系统](https://mp.weixin.qq.com/s/aaOosZ57qJpIU6cma820Xw)
- [张俊林：技术演进趋势：召回->排序->重排](https://mp.weixin.qq.com/s?__biz=MzU1NTMyOTI4Mw==&mid=2247496363&idx=1&sn=0d2b2ac176e2a72eb2e760b7b591788f&chksm=fbd740c7cca0c9d16c76fdeb1a874a53f7408d8125b2e1bed3173ecb69d131167c1c9c35c71f&scene=21#wechat_redirect)
- [微信"看一看"多模型内容策略与召回](https://mp.weixin.qq.com/s?__biz=MzU1NTMyOTI4Mw==&mid=2247503484&idx=2&sn=e2a2cdd3a517ab09e903e69ccb1e9f94&chksm=fbd77c10cca0f50642dde47439ed919aa2e61b7ff57bc4cbaacc3acaac3c620a1ed6f92684ab&scene=21#wechat_redirect)
- [多目标学习在推荐系统中的应用](https://mp.weixin.qq.com/s/u_5RdZ-BcIu_RoWNri76ig)
- [强化学习在美团“猜你喜欢”的实践](https://mp.weixin.qq.com/s?__biz=MjM5NjQ5MTI5OA==&mid=2651749434&idx=2&sn=343e811408542dd1984582b8639240a6&chksm=bd12a5778a652c61ed4297f1a17582cad4ca6b8e8d4d66843f169e0eda9f6aede988bc675743&mpshare=1&scene=23&srcid=1115EcgbMw6GAhMnzV0URvgd#rd)
- [推荐系统技术演进趋势：重排篇](https://mp.weixin.qq.com/s/YorzRyK0iplzqutnhEhrvw)
- [阿里强化学习重排实践](https://mp.weixin.qq.com/s/ylavFA_MXLUhIBLCqxAjLQ)
