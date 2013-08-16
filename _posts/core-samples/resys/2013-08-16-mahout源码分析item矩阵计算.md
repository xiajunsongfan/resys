---
layout: post
title:  "mahout源码分析item矩阵计算"
date:   2013-08-16 15:03:11
category: mahout
tags : [推荐, java, mahout, hadoop]
---
##mahout源码分析item矩阵计算
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