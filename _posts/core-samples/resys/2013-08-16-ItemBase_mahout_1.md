---
layout: post
title:  "ItemBase_mahout源码分析一"
date:   2013-08-16 15:03:11
category: mahout
tags : [推荐, java, mahout, hadoop]
---
##item矩阵计算
**author : xiajun**
-
**简介：**

itemMatrix是计算两个商品之间的距离。

**mahout实现:**

	user	item	pre
	 001	10001    3
	 001	10004	 4
	 002	10001	 2

**PreparePreferenceMatrixJob是处理数据格式**

1.itemIDIndex 将item 进行索引化

输出:index:item  这样做的原因是mahout矩阵计算使用的是int 而item可以能大于int所以进行取hashCode的值

2.toUserVectors 将文本user:item:pre转成用户向量

ToItemPrefsMapper:

输入：user  item pre

输出：user:{item:pre}用户为key,商品和评分为值。如果是0,1阵值就是商品无评分

ToUserVectorsReducer:

此reduce使用了minPreferences参数将一个用户评价的商品数少于参数值时被过滤

输出：user:[{item:pre},{item,pre}]将同一个用户评价的商品聚合起来.并统计了用户总数.

输出路径：userVectors

numUsers.bin中存放的是用户总数

3.toItemVectors 将用户向量转成item向量

ToItemVectorsMapper：

输入路径：userVectors(上一步的输出)

输入：user:[{item:pre},{item,pre}] (上一步的输出)

此map使用了maxPrefsPerUser参数如果用户评价的商品大于此参数值那将随机获取参赛值个评价(由于0.8及以前版本中存在bug建议将此值尽量设置大)

输出：item:{user:pre}

reduce输出:item:[{user:pre},{user:pre}]


**RowSimilarityJob是mahout实现itemMatrix的主要job**

1.normsAndTranspose 获取对商品评价的人数，对商品的最大评分。

norms=norm += value * value;//如果是0，1阵 结果就是对该商品评价的人数

map 输出: user:{item:pre}

reduce输出： user:[{item:pre},{item:pre}]

2.pairwiseSimilarity 计算相似度

map:将item与item间的评分聚合

输出:itemA:{itemB:(PitemAxPitemB)} itemA的评分乘以itemB的评分

reduce 输入itemA:[itemB:Pab,itemC:Pac]

相似度计算：

	double similarityValue = similarity.similarity(b.get(), normA, norms.getQuick(b.index()), numberOfColumns);
	//b.get()为Pab normA为评价了A的人数,norms.getQuick(b.index())为评价了B的人数,numberOfColumns为整个input文件中的总人数.
		
	public double similarity(double dots, double normA, double normB, int numberOfColumns) {//欧式距离
	   double euclideanDistance = Math.sqrt(normA - 2 * dots + normB);
	   return 1.0 / (1.0 + euclideanDistance);
	}
输出:itemA:{itemB:Pabsimi,itenC:Pacsimi}

3.asMatrix 获取每个item相似度高的top值

map输入 itemA:{itemB:Pabsimi,itenC:Pacsimi}

输出：itemA:{itemB:Pabsimi,itenC:Pacsimi} top个

reduce：输出 itemA:{itemB:Pabsimi,itenC:Pacsimi}  再次过滤top值

**参数介绍：**

numRecommendations：为每个用户生成的推荐用户数量

usersFile：设置偏爱的用户

itemsFile：设置偏爱的项目

filterFile：userID,itemID  这种格式的文件，用于排除推荐

booleanData：是否为无评分的

maxPrefsPerUser：最多取n个评价了item的用户及评分（如：一个用户对应100个商品时，那么只有n个会进入矩阵计算,在recommendjob类中使用的位置不同但同样是一个用户参与矩阵计算的最大商品数）

minPrefsPerUser：用户最少对多少个商品进行了评分后才可以进行计算(如：一个用户对应的商品少于n时此用户不进入矩阵计算)

maxSimilaritiesPerItem：每个产品最多考虑多少个最相似产品

maxPrefsPerUserInItemSimilarity：取出n个用户的对商品的评价进行计算（在使用recommendjob时 和maxPrefsPerUser一样）

similarityClassname：使用的相似算法

threshold：丢弃相似值低于该项目的项