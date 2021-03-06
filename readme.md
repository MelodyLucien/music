基于随机森林算法的歌词分类研究
========================================================

本文档详细介绍如何使用R语言，进行文本情感分类研究。

## 1 加载包

各个package的主要功能如下：

+ tm 形成文档词条矩阵
+ Rwordseg 中文分词
+ FSelector 特征提取，有chi-square，information gain等等
+ RTextTools 文本挖掘分类算法,我这里用到的是随机森林和SVM


```r
library(tm)
library(Rwordseg)
```

```
## Loading required package: rJava
## # Version: 0.0-4
## 
## Attaching package: 'Rwordseg'
## 
## The following object is masked from 'package:tm':
## 
##     removeWords
```

```r
library(RTextTools)
```

```
## Loading required package: SparseM
## 
## Attaching package: 'SparseM'
## 
## The following object is masked from 'package:base':
## 
##     backsolve
## 
## KernSmooth 2.23 loaded
## Copyright M. P. Wand 1997-2009
```

```
## Warning: replacing previous import 'coerce' when loading 'SparseM'
```

```r
library(FSelector)
```


## 2 读取歌词文本

读入工作目录下的`sweetsong.csv`和`sadsong.csv`文件，分别为甜蜜歌词和伤感歌词。甜蜜歌词来自于百度音乐[甜蜜标签](http://music.baidu.com/tag/%E7%94%9C%E8%9C%9C)下的所有歌曲,
伤感歌词来自于百度音乐[伤感标签](http://music.baidu.com/tag/%E4%BC%A4%E6%84%9F)下的所有歌曲。采用python的scrapy爬虫框架抓取这些资料，源代码在[这里](https://github.com/phenix502/baidu)。




```r
# 防止读入的所有string都被当成factor
options(stringsAsFactors = FALSE)
# 读入csv文件
Infor.sweet <- read.csv("material/sweetsong.csv", header = TRUE)
Infor.sad <- read.csv("material/sadsong.csv", header = TRUE)
```


## 3 分词并形成语料库
使用中文分词包Rwordseg对歌词进行分词。并把分好的词合成语料库。

```r
## 由两个character类型的变量 形成语料库
removeEnglish <- function(x) {
    gsub("[a-z]+|[A-Z]+", "", x)
}

makeCorpus <- function(str1, str2) {
    # 伤感歌曲分词 组成语料库
    word.sad <- lapply(str1, removeEnglish)
    word.sad <- lapply(word.sad, segmentCN)
    corpus.sad <- Corpus(VectorSource(word.sad))
    
    # 甜蜜歌曲分词 组成语料库
    word.sweet <- lapply(str2, removeEnglish)
    word.sweet <- lapply(word.sweet, segmentCN)
    corpus.sweet <- Corpus(VectorSource(word.sweet))
    
    # 合成预料库
    corpus <- c(corpus.sad, corpus.sweet)
    return(corpus)
}

corpus <- makeCorpus(Infor.sweet$lyric, Infor.sad$lyric)
```






## 3 document-term matrix 函数实现
要将文本信息转为可以给各种分类计算的信息，首先要把文本信息转为各种能计算的数字。document-term matrix每一列是一个词语，每一行是词频数，当然一般用TF-IDF作为特征权值计算。document-term 矩阵就是分类算法的特征矩阵，只是没给每一个数据集标上所属的类别。


```r
dtm <- function(corpus, tfidf = FALSE) {
    
    ## 读取停止词
    mystopwords <- readLines("material/stopwords.txt")
    if (tfidf == TRUE) {
        ## 文档-词矩阵 词的长度大于1就纳入矩阵
        cor.dtm <- DocumentTermMatrix(corpus, control = list(wordLengths = c(2, 
            Inf), stopwords = mystopwords, weighting = weightTfIdf))
    } else {
        cor.dtm <- DocumentTermMatrix(corpus, control = list(wordLengths = c(2, 
            Inf), stopwords = mystopwords))
    }
    ## 去掉稀疏矩阵中低频率的词
    cor.dtm <- removeSparseTerms(cor.dtm, 0.98)
    
    ## 使得每一行至少有一个词不为0 rowTotals <- apply(cor.dtm, 1, sum) cor.dtm <-
    ## cor.dtm[rowTotals > 0]
    return(cor.dtm)
}
```


利用`dtm`函数，形成为文本词条矩阵

```r
corpus.dtm <- dtm(corpus)
corpus.dtm.tfidf <- dtm(corpus, tfidf = TRUE)
```


## 4 使用算法对歌词进行情感极性的分析

首先对每一首标上对应的类别。数据集中前764首歌曲是伤感歌曲，后面861首是甜蜜的歌曲。然后确定测试集和训练集范围。

然后传入一个document-term matrix，然后使用SVM和随机森林算法对输入的数据进行分类。

```r
## 传入一个document-term matrix，使用SVM和随机森林进行分类
algorithm_summary <- function(dtm) {
    
    # 类别向量
    label <- factor(c(rep("sad", 764), c(rep("sweet", 861))))
    # 从伤感歌词中挑选64首作为测试集，同理甜蜜类挑选61首作为测试集
    sad.test <- sample(1:764, 64, replace = FALSE)
    sweet.test <- sample(765:1625, 61, replace = FALSE)
    testSize <- c(sad.test, sweet.test)
    trainSize <- 1:1625
    trainSize <- trainSize[-testSize]
    
    # create a container
    container.song <- create_container(dtm, label, trainSize = trainSize, testSize = testSize, 
        virgin = FALSE)
    
    # training models
    SVM.song <- train_model(container.song, "SVM")
    RF.song <- train_model(container.song, "RF")
    
    # classifying data using trained models
    SVM_CLASSIFY.song <- classify_model(container.song, SVM.song)
    RF_CLASSIFY.song <- classify_model(container.song, RF.song)
    
    SVM_result <- create_precisionRecallSummary(container = container.song, 
        classification_results = SVM_CLASSIFY.song)
    RF_result <- create_precisionRecallSummary(container = container.song, classification_results = RF_CLASSIFY.song)
    return(list(SVM_RESULT = SVM_result, RF_RESULT = RF_result))
}
```


使用algorithm_summary函数对歌曲进行分类，我们看一下分类结果。

```r
result_all_corpus <- algorithm_summary(corpus.dtm.tfidf)
```

```
## Warning: NAs introduced by coercion
## Warning: NAs introduced by coercion
```

```r
result_all_corpus
```

```
## $SVM_RESULT
##       SVM_PRECISION SVM_RECALL SVM_FSCORE
## sad            0.74       0.72       0.73
## sweet          0.71       0.74       0.72
## 
## $RF_RESULT
##       FORESTS_PRECISION FORESTS_RECALL FORESTS_FSCORE
## sad                0.78           0.61           0.68
## sweet              0.67           0.82           0.74
```

可以看出分类结果不是很好，这是因为document-term matrix是一个高维稀疏的矩阵，为了提高分类效果，可以尝试特征提取。


## 5 特征提取

采用随机森林算法选取前100个重要的词语，`subset`即是前100有重要分类信息的词语


```r
# 转为data frame
corpus.df <- as.data.frame(inspect(corpus.dtm.tfidf))
```



```r

## 随机森林算法选取前100个重要的词语
label <- factor(c(rep("sad", 764), c(rep("sweet", 861))))
weights.rf <- random.forest.importance(label ~ ., corpus.df, 1)
subset <- cutoff.k(weights.rf, 100)
## 把提取的特征作为新的docment-term matrix
feature.df <- as.DocumentTermMatrix(corpus.df[subset], weighting = weightTf)
```

让我们再次看看分类效果


```r
result_feature <- algorithm_summary(feature.df)
```

```
## Warning: NAs introduced by coercion
## Warning: NAs introduced by coercion
```

```r
result_feature
```

```
## $SVM_RESULT
##       SVM_PRECISION SVM_RECALL SVM_FSCORE
## sad            1.00       0.91       0.95
## sweet          0.91       1.00       0.95
## 
## $RF_RESULT
##       FORESTS_PRECISION FORESTS_RECALL FORESTS_FSCORE
## sad                1.00           0.88           0.94
## sweet              0.88           1.00           0.94
```










