---
title: "Collaborative Filtering Recommendation"
author: "Claus Lv"
date: "Monday, August 18, 2014"
output: html_document
---

## Item based

item based 协同推荐算法
基于物品的协同过滤ItemCF

数据集字段：

- User_id: 用户ID
- Item_id: 物品ID
- preference:用户对该物品的评分


算法的思想：

1. 建立物品的同现矩阵A，即统计两两物品同时出现的次数
数据格式：Item_id1:Item_id2        次数

2. 建立用户对物品的评分矩阵B，即每一个用户对某一物品的评分
数据格式：Item_id          user_id:preference

3. 推荐结果=物品的同现矩阵A * 用户对物品的评分矩阵B
数据格式：user_id           item_id,推荐分值

4. 过滤用户已评分的物品项

5.对推荐结果按推荐分值从高到低排序


## R 实现

load library

```r
library(plyr)
```


### Create Sample Data

```r
user <- c(1,1,1,2,2,2,2,3,3,3,3,4,4,4,4,5,5,5,5,5,5)
item <- c(101,102,103,101,102,103,104,101,104,105,107,101,103,104,106,101,102,103,104,105,106)
pref <- c(5.0,3.0,2.5,2.0,2.5,5.0,2.0,2.0,4.0,4.5,5.0,5.0,3.0,4.5,4.0,4.0,3.0,2.0,4.0,3.5,4.0)
train <- data.frame(user,item,pref)

summary(train)
```

```
##       user           item          pref     
##  Min.   :1.00   Min.   :101   Min.   :2.00  
##  1st Qu.:2.00   1st Qu.:102   1st Qu.:2.50  
##  Median :3.00   Median :103   Median :4.00  
##  Mean   :3.29   Mean   :103   Mean   :3.55  
##  3rd Qu.:5.00   3rd Qu.:104   3rd Qu.:4.50  
##  Max.   :5.00   Max.   :107   Max.   :5.00
```


```r
#计算用户列表
usersUnique <- function(){
    users <- unique(train$user)
    users[order(users)]
}

#计算商品列表方法
itemsUnique <- function(){
    items <- unique(train$item)
    items[order(items)]
}

# 用户列表
users <- usersUnique() 
# 商品列表
items <- itemsUnique() 

#建立商品列表索引
index <- function(x) which(items %in% x)
data <- ddply(train,
              .(user,item,pref),
              summarize,
              idx=index(item)) 

head(data)
```

```
##   user item pref idx
## 1    1  101  5.0   1
## 2    1  102  3.0   2
## 3    1  103  2.5   3
## 4    2  101  2.0   1
## 5    2  102  2.5   2
## 6    2  103  5.0   3
```

### 同现矩阵
按用户分组，找到每个用户所选的物品，单独出现计数，及两两一组计数。

例如：用户ID为3的用户，分别给101,104,105,107，这4个物品打分。
1) (101,101),(104,104),(105,105),(107,107)，单独出现计算各加1。
2) (101,104),(101,105),(101,107),(104,105),(104,107),(105,107)，两个一组计数各加1。
3) 把所有用户的计算结果求和，生成一个三角矩阵，再补全三角矩阵，就建立了物品的同现矩阵。

如下面矩阵所示：

      [101] [102] [103] [104] [105] [106] [107]
[101]   5     3     4     4     2     2     1
[102]   3     3     3     2     1     1     0
[103]   4     3     4     3     1     2     0
[104]   4     2     3     4     2     2     1
[105]   2     1     1     2     2     1     1
[106]   2     1     2     2     1     2     0
[107]   1     0     0     1     1     0     1


```r
#同现矩阵
cooccurrence <- function(data){
    n <- length(items)
    co <- matrix(rep(0,n*n),nrow=n)
    for(u in users){
        idx <- index(data$item[which(data$user==u)])
        #每个用户的商品两两出现的集合
        m <- merge(idx,idx)
        for(i in 1:nrow(m)){
            #根据商品的索引找到同现矩阵中的位置进行次数累加
            co[m$x[i],m$y[i]] = co[m$x[i],m$y[i]]+1
        }
    }
    return(co)
}
```

### 推荐算法
按用户分组，找到每个用户所选的物品及评分

例如：用户ID为3的用户，分别给(3,101,2.0),(3,104,4.0),(3,105,4.5),(3,107,5.0)，这4个物品打分。
1) 找到物品评分(3,101,2.0),(3,104,4.0),(3,105,4.5),(3,107,5.0)
 2) 建立用户对物品的评分矩阵

       U3
[101] 2.0
[102] 0.0
[103] 0.0
[104] 4.0
[105] 4.5
[106] 0.0
[107] 5.0

矩阵计算推荐结果

同现矩阵*评分矩阵=推荐结果


```r
#推荐算法
recommend <- function(udata=udata,co=coMatrix,num=0){
    n <- length(items)
    
    # all of pref
    pref <- rep(0,n)
    pref[udata$idx] <- udata$pref  
    
    # 用户评分矩阵
    userx <- matrix(pref,nrow=n)
    
    # 同现矩阵*评分矩阵
    r <- co %*% userx  
    
    # 推荐结果排序
    # 把该用户评分过的商品的推荐值设为0
    r[udata$idx] <- 0
    idx <- order(r,decreasing=TRUE)
    topn <- data.frame(user=rep(udata$user[1],length(idx)),
                       item=items[idx],
                       val=r[idx])
    # 推荐结果取前num个
    if(num>0){
        topn <- head(topn,num)
    }
    #返回结果
    return(topn)
}
```

## Run

```r
#生成同现矩阵
co <- cooccurrence(data) 
co
```

```
##      [,1] [,2] [,3] [,4] [,5] [,6] [,7]
## [1,]    5    3    4    4    2    2    1
## [2,]    3    3    3    2    1    1    0
## [3,]    4    3    4    3    1    2    0
## [4,]    4    2    3    4    2    2    1
## [5,]    2    1    1    2    2    1    1
## [6,]    2    1    2    2    1    2    0
## [7,]    1    0    0    1    1    0    1
```

```r
#计算推荐结果
recommendation <- data.frame()
for(i in 1:length(users)){
  udata <- data[which(data$user==users[i]),]
  recommendation <- rbind(recommendation,recommend(udata,co,0)) 
} 
recommendation <- recommendation[which(recommendation$val > 0),]
recommendation
```

```
##    user item  val
## 1     1  104 33.5
## 2     1  106 18.0
## 3     1  105 15.5
## 4     1  107  5.0
## 8     2  106 20.5
## 9     2  105 15.5
## 10    2  107  4.0
## 15    3  103 24.5
## 16    3  102 18.5
## 17    3  106 16.5
## 22    4  102 37.0
## 23    4  105 26.0
## 24    4  107  9.5
## 29    5  107 11.5
```

