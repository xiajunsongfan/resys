---
layout: post
title:  " Mahout 协同过滤思想--ItemBase"
date:   2013-08-13 19:03:11
category: resys
---
##Mahout 协同过滤思想--ItemBase
**author : xiajun**</br>
---
当根据item算出矩阵后的协同过滤实现思想
ABCD为商品 U1,U2为用户对商品的评价</br>
**NAN**在矩阵中很重要,mahout做过滤都是使用NAN(原理NAN+任意数字后仍然为NAN)
<table style="border-collapse: collapse;font-family: sans-serif;">
<tr style="height:35px">
<td style="width:40px;">
</td>
<td style="width:40px;">
A
</td>
<td style="width:40px;">
B
</td>
<td style="width:40px;">
C
</td>
<td style="width:40px;border-right-color:red">
D
</td>
<td style="width:40px;">
U1
</td>
<td style="width:40px;">
U2
</td>
</tr>
<tr style="height:35px">
<td style="width:40px;">
A
</td>
<td style="width:40px;">
NAN
</td>
<td style="width:40px;">
0.12
</td>
<td style="width:40px;">
0
</td>
<td style="width:40px;border-right-color:red">
0.4
</td>
<td style="width:40px;">
2
</td>
<td style="width:40px;">
?
</td>
</tr>
<tr style="height:35px">
<td style="width:40px;">
B
</td>
<td style="width:40px;">
0.1
</td>
<td style="width:40px;">
NAN
</td>
<td style="width:40px;">
0
</td>
<td style="width:40px;border-right-color:red">
0.5
</td>
<td style="width:40px;">
?
</td>
<td style="width:40px;">
3
</td>
</tr>
<tr style="height:35px">
<td style="width:40px;">
C
</td>
<td style="width:40px;">
0
</td>
<td style="width:40px;">
0.2
</td>
<td style="width:40px;">
NAN
</td>
<td style="width:40px;border-right-color:red">
0.3
</td>
<td style="width:40px;">
2
</td>
<td style="width:40px;">
?
</td>
</tr>
<tr style="height:35px">
<td style="width:40px;">
D
</td>
<td style="width:40px;">
0.4
</td>
<td style="width:40px;">
0.2
</td>
<td style="width:40px;">
0
</td>
<td style="width:40px;border-right-color:red">
NAN
</td>
<td style="width:40px;">
3
</td>
<td style="width:40px;">
1
</td>
</tr>
</table>
###1.转换user向量
将U1 转成 U1:[{A:2},{C:2},{D:3}]</br>
将U2 转成 U2：[{B:3}，{D:1}]
###2.将1步转成item:{user:pre}形式 prePartialMultiply2-Job
A:{U1:2}
C:{U1:2}
D:{U1:3}

B:{U2:3}
D:{U2:1}

###3.转换Item矩阵为行形式prePartialMultiply1-Job
A:[{B:0.12},{D:0.4},{A:NAN}] 自己和自己的相似度为NAN</br>
B:[{A:0.1},{D:0.5},{B:NAN}]</br>
C:[{B:0.2},{D:0.3},{C:NAN}]</br>
D:[{A:0.4},{B:0.2},{D:NAN}]</br>
###4.将2、3两步作为输入文件传入partialMultiply-Job
以上文件将以item为key传入reduce方法中

	void reduce(VarIntWritable key,Iterable values,Context context){

    List<Long> userIDs = Lists.newArrayList();//存放Item中的用户
    List<Float> prefValues = Lists.newArrayList();//存放用户对应的评分
    Vector similarityMatrixColumn = null;
    for (VectorOrPrefWritable value : values) {//迭代values 获取key对应的用户和item
      if (value.getVector() == null) {//为用户行为转换过来的 (Item:{user:pre})
        userIDs.add(value.getUserID());
        prefValues.add(value.getValue());
      } else {//判断是否是item矩阵文件(第3步矩阵转换时使用Item:V(Item:pre))
        if (similarityMatrixColumn != null) {
          throw new IllegalStateException("");
        }
        similarityMatrixColumn = value.getVector();//取出V(Item:pre)
      }
    }

    if (similarityMatrixColumn == null) {
      return;
    }

    VectorAndPrefsWritable vectorAndPrefs = new VectorAndPrefsWritable(similarityMatrixColumn, userIDs, prefValues);
    context.write(key, vectorAndPrefs);
    }

经过reduce后输出文件格式为:Item:[[{Item1:pre-I1},{Item2:pre-I2}],{User1,User2},{Pre-u1,pre-u2}]</br>
A:[[{B:0.12},{D:0.4},{A:NAN}],{U1},{2}]

B:[[{A:0.1},{D:0.5},{B:NAN}],{U2},{3}]

C:[[{B:0.2},{D:0.3},{C:NAN}],{U1},{2}]

D:[[{A:0.4},{B:0.2},{D:NAN}],{U1,U2},{3,1}]
###5.aggregateAndRecommend-JOb 将文件转为用户推荐
Map：PartialMultiplyMapper 以第4步为输入文件

	void map(VarIntWritable key,
    VectorAndPrefsWritable vectorAndPrefsWritable,
                     Context context) throws IOException, InterruptedException {

    Vector similarityMatrixColumn = vectorAndPrefsWritable.getVector();//获取向量(item矩阵中对应的item)[{B:0.12},{D:0.4}]
    List<Long> userIDs = vectorAndPrefsWritable.getUserIDs();//获取用户list
    List<Float> prefValues = vectorAndPrefsWritable.getValues();//获取用户评分list

    VarLongWritable userIDWritable = new VarLongWritable();
    PrefAndSimilarityColumnWritable prefAndSimilarityColumn = new PrefAndSimilarityColumnWritable();

    for (int i = 0; i < userIDs.size(); i++) {//迭代用户列表
      long userID = userIDs.get(i);
      float prefValue = prefValues.get(i);
      if (!Float.isNaN(prefValue)) {
        prefAndSimilarityColumn.set(prefValue, similarityMatrixColumn);
        userIDWritable.set(userID);
        context.write(userIDWritable, prefAndSimilarityColumn);//输出格式 user:[pre,[{item1:pre-I1},{item2:pre-I2}]]
      }
     }
	}
**输出：**

U1:[2,[{B:0.12},{D:0.4},{A:NAN}]]

U1:[2,[{B:0.2},{D:0.3},{C:NAN}]]

U1:[3,[{A:0.4},{B:0.2},{D:NAN}]]

U2:[3,[{A:0.1},{D:0.5},{B:NAN}]]

U2:[1,[{A:0.4},{B:0.2},{D:NAN}]]</br>
**注意：**C商品已经被踢掉了

**Reduce:**AggregateAndRecommendReducer

输入：

U1:{[2,[{B:0.12},{D:0.4},{A:NAN}]];[2,[{B:0.2},{D:0.3},{C:NAN}]];[3,[{A:0.4},{B:0.2},{D:NAN}]]}

U2:{[3,[{A:0.1},{D:0.5},{B:NAN}]],[1,[{A:0.4},{B:0.2},{D:NAN}]]}

将每个用户评分和对应的向量相似度相乘 如U1: [2,[{B:0.12},{D:0.4}]] 为一个评分和对应的向量 pre_B=2x0.12

计算方式：

U1:[{B:2x0.12},{D:2x0.4},{A:2xNAN},{B:2x0.2},{D:2x0.3},{C:2xNAN},{A:3x0.4},{B:3x0.2},{D:3xNAN}] 

U2:[{A:3x0.1},{D:3x0.5},{B:3xNAN},{A:1x0.4},{B:1x0.2},{D:1xNAN}]

----

U1->A (2xNAN+3x0.4)/(0.4)  相关度 NAN 由于U1已经购买A所以过滤掉

U1->B (2x0.12+2x0.2+3x0.2)/(0.12+0.2+0.2) 相关度 2.296 为了平衡相似评分所以u与A的相关度要除以u与A的相似度之和

U1->C (无)

U1->D (3xNAN+2x0.4+2x0.3)\(0.4+0.3) 相关度 NAN D 已经购买

U2->A (3x0.1+1x0.4)\(0.1+0.4) 相关度 1.4  推荐

U2->B (3xNAN+1x0.2)\(0.2) 相关度 NAN 已购买

U2->C (无)

U3->D (3x0.5+1xNAN)\(0.5) 相关度 NAN 已购买

结果为 U1推 B：2.296  U2推 A:1.4</br>
**注意mahout使用了一个很隐蔽的去除购买商品的方式就是使用NAN**

如果为无评分数据时
U1->A (3x0.4)/***(0.4)*** 不需要除以(0.4) U1->A 相关度为 1.2

U1->B (2x0.12+2x0.2+3x0.2) 相关度为 1.24
