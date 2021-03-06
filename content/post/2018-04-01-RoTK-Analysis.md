---
title: "用R分析光荣《三国志》系列人物数据"
date: 2018-11-30
author: "潘新晨"
categories: ["R语言"]
tags: ["三国", "R"]
slug: RoTK-Analysis
meta_extra: "审稿：李舰"
forum_id: 420245
---

## 前言

写这篇文章有两个原因，第一个是最近在看吴秀波演的《军师联盟》，这部剧剧情紧凑，演员演技精湛，有很多令人惊艳的细节， 再一次勾起了我对三国的兴趣。从小到大玩过不少三国游戏，看过很多三国的书，电视剧如央视版三国，高希希版新三国也都不在话下，而这些年除了偶尔玩玩《三国志10》并没有再对三国有什么研究， 想通过这个分析再重温下三国里的那些人物和故事。第二个原因是自己有比较长一段时间没怎么写R， 工作上用到R的机会也不是很多，手有点生，打算用这个数据练练手，加之这是个中文数据，以前没有过用R处理中文数据的经验。

人物数值系统一直是光荣历史游戏的一个特点，它是光荣结合史实，小说，野史等资料对人物的一个全面评价。光荣公司的《三国志》系列可以说是最经典的三国游戏，从1985年第一代推出以来，到现在已经有了十三代作品。而每代出场的数百个武将，经过这么多个版本，他们的各项属性是否会有什么大的变化呢，而这些变化反映了什么呢，这也是这次分析主要想研究的。

我用的武将数据是一个台湾网友（cws0324@yahoo.com.tw）收集制作的，它包含了三国志 1-11 所有的武将数据。给人物以数值化好处自然不言而喻，它可以让我们对游戏中武将的强弱有个直观的了解，但坏处就是我们有时候只看重数值，而忽略了该人物在历史上真实的一面，这问题在日本战国人物里更严重些。E.g. 伊达政宗，竹中半兵卫。想研究的话请参考

[为什么日本战国时期有很多名将？](https://www.zhihu.com/question/38221711)(第一个回答把我笑死了)

[国内非专业圈中（民科？）日本战国时代历史学习的氛围和现状有感](https://zhuanlan.zhihu.com/p/23140678)

马伯庸在知乎一个问题的回答里做过类似的分析，他用的工具是Excel, 有兴趣的可以去看看. [光荣公司《三国志》游戏里的武将设定，是按照三国历史设计的吗？](https://www.zhihu.com/question/21530813) 

我还用该数据+shiny做了一个三国志人物数据查询工具，[https://nathanpan.shinyapps.io/RoTC-Searching/](https://nathanpan.shinyapps.io/RoTC-Searching/)

由于自己也是对演义了解的更多，所以分析的时候更多会以演义的角度，再结合史实。话不说多，开始我们的分析。

## 1. 数据处理

首先看下这份数据本身的格式。

![](https://spsufawi.rbind.io/content/post/RoTC-Analysis_files/attr.png)

我们发现有多个相同的变量分部在不同的列，而且版本信息占据了多个格子。这种数据不手动处理很难读进R（可能可以？），在excel里也许能直接用，但在R里这不属于我们所说的[干净数据](http://r4ds.had.co.nz/tidy-data.html)。干净数据有如下定义

1. 每一个变量必须有它自己的列
2. 每一个样本必须有它自己的行
3. 每一个数值必须在它自己的格子

我决定自己手动把版本数据分开到11个表格里，每一个如下图所示。(自然我也可以手动直接得到我们最终想要的数据，但我还是决定用R实现它。）

![](https://spsufawi.rbind.io/post/RoTC-Analysis_files/attr1.png)

初步处理过后的数据我放到了[这里](https://github.com/spsufawi/My-Blog/blob/master/content/post/Characters.xlsx)

这些是我们需要用到的package.

```r
library(readxl)
library(dplyr)
library(data.table)
library(ggplot2)
```

为了能在R里使用中文，我们用下面的代码将系统locale设置为”Chs”, 这里用的操作系统是Win10家庭版。

```r
Sys.setlocale('LC_ALL','Chs')
```

接着用 `readxl::read_excel` 读取数据，每一个版本的数据分别存在了1个data frame里，我们总共有11个data frame，而这11个data frame又都再存在一个list里，因为 `lapply` 返回的就是list结构，我们给它命名为 `dt` .

```r
dt <- lapply(1:11, function(x) read_excel("Characters.xlsx", x)) 
```

由于每一代武将所拥有的属性不一样，为了后边方便，我希望能做一个大的data frame，它的变量有姓名，所有版本的属性以及该人物出场的版本。下面我们来看如何达到这个目的。

我先把每一个版本的数据移除不需要的变量和那一个版本没有出现过的武将，部分NPC各项属性全都为0，我们也将它去掉，把清理过后的数据存到一个新的变量里，叫做 `series` 。

```r
# Column 2-8 为不需要的变量
series <- lapply(1:11, function(x) {select(dt[[x]], -c(2:8)) %>%
    filter(complete.cases(dt[[x]][,-c(2:8)])) %>%
    mutate(版本 =  paste0("三國志", x)) %>%
    filter(智力!= 0) }) # 通常一项属性为0，其它属性也都为0 
```

第一代前六个武将是如下几位。

```r
head(series[[1]])
```

```
# A tibble: 6 x 7
  姓名    體力  武力  智力  魅力  運勢 版本   
  <chr>  <dbl> <dbl> <dbl> <dbl> <dbl> <chr>  
1 丁奉     81.   22.   81.   29.   47. 三國志1
2 于禁     82.   72.   20.   25.   28. 三國志1
3 公孫瓚   84.   70.   67.   89.   28. 三國志1
4 太史慈   88.   97.   47.   84.   34. 三國志1
5 孔融     84.   82.   61.   50.   77. 三國志1
6 文聘     88.   84.   22.   64.   83. 三國志1
```

前边说过，每一代游戏武将拥有的属性不一样，第一代有体力，第二代把统率分成了陆指和水指, 第九代没有魅力。那我们来看下所有作品都有属性是什么，以及每一代都有的属性。

排除姓名和版本，历代都出现过的属性只有武力和智力，这两个属性也是我们后边会主要分析的。

```r
common_attr <- Reduce(intersect, sapply(series, colnames))
all_attr <- Reduce(union, sapply(series, colnames))
all_attr[2:9]
```

```
[1] "體力" "武力" "智力" "魅力" "運勢" "版本" "政治" "陸指"
```

```r
common_attr[2:3]
```

```
[1] "武力" "智力"
```

因为现在我们的11个data frame是分开的，我们需要把它们合并，但每个data frame的列数不一样，这该如何合并呢。最简单的方法就是用 `plyr::rbind.fill()` ， 它可以自动的把缺失值用NA补上。在用这个function之前我尝试了一个非常复杂的方法，这种方法需要调整变量顺序，这里不多阐述了。

我还调整了版本这个categorical variable的level，使它是按照1-11代的顺序排列， 主要是为了之后做图的时候X轴的值（版本）能够按照正常顺序排列。

```r
series_full <- do.call(plyr::rbind.fill, series) %>%
   mutate(版本 = factor(版本, levels = paste0("三國志", 1:11))) %>%
   select(c(1,11,3:10,2)) # 体力只出现于第一个版本，把它移到最后一列
```

现在我们得到了我们想要的这个大的data frame,它是这个样子的。

```r
head(series_full)
```

```
 姓名 統率 武力 智力 魅力 運勢    版本 政治 陸指 水指 體力
1   丁奉   NA   22   81   29   47 三國志1   NA   NA   NA   81
2   于禁   NA   72   20   25   28 三國志1   NA   NA   NA   82
3 公孫瓚   NA   70   67   89   28 三國志1   NA   NA   NA   84
4 太史慈   NA   97   47   84   34 三國志1   NA   NA   NA   88
5   孔融   NA   82   61   50   77 三國志1   NA   NA   NA   84
6   文聘   NA   84   22   64   83 三國志1   NA   NA   NA   88
```

## 2. 各项数据分析

### 2.1 各代人物数量分析

先来看下各代出场的人物数量，

```r
char_freq <- group_by(series_full, 版本) %>% summarise(人數 = n())
char_freq
```

```
# A tibble: 11 x 2
   版本      人數
   <fct>    <int>
 1 三國志1    256
 2 三國志2    352
 3 三國志3    770
 4 三國志4    454
 5 三國志5    571
 6 三國志6    520
 7 三國志7    539
 8 三國志8    635
 9 三國志9    674
10 三國志10   650
11 三國志11   702
```

不出所料第一版是出场人物最少的，这些武将想必应该都是我们耳熟能详的武将吧，就像三国杀最早的标准包里只有三国和群雄势力的二十几个最主要人物，而现在秦宓，孙资，刘放这种角色都已登场。

再来看十一部作品中都出场的有多少位人物。

```r
show_in_all <- Reduce(intersect, lapply(series, "[", 1))
nrow(show_in_all)
```

```
[1] 221
```

1-11代每一代都出现的角色有221名，我们从里边随机选10个，不出意外都是你可以随便说出他的故事的角色(说出你的故事)。

```r
sample_n(show_in_all, 10)
```

```
# A tibble: 10 x 1
   姓名 
   <chr>
 1 丁奉 
 2 楊秋 
 3 張昭 
 4 孔融 
 5 潘璋 
 6 劉循 
 7 陳宮 
 8 顏良 
 9 魏延 
10 袁術 
```

那有哪些角色出现在了第一部，却在后边某些版本没出现呢。

```r
show_in_all <- Reduce(intersect, lapply(series, "[", 1))
setdiff(filter(series_full, 版本 == "三國志1")$姓名, show_in_all$姓名)
```

```
 [1] "朱褒"   "朱儁"   "呂公"   "宋謙"   "李堪"   "侯選"   "胡軫"  
 [8] "馬延"   "張既"   "張顗"   "梁綱"   "陳紀"   "陳珪"   "陳琳"  
[15] "陳嬉"   "陳應"   "陳蘭"   "傅士仁" "傅幹"   "楊奉"   "楊彪"  
[22] "趙岑"   "劉永"   "劉理"   "劉賢"   "劉璝"   "樂就"   "蔣欽"  
[29] "蔣幹"   "蔡邕"   "鮑信"   "鍾進"   "韓浩"   "譙周"   "嚴畯"  
```

这些人有些也是比较出名的，比如在《三国演义》中了周瑜反间计的蒋干（没猜方片？），蜀国坚定的投降派，大学者谯周，东吴大将蒋钦，据传文章能治曹操头痛的建安七子之一的陈琳， 其他大部分都是不太出名的角色，如韩遂，袁术的一帮子人。

### 2.2. 三国主君数值对比

先把刘备孙权和曹操的数据抓出来单独存在一个data  frame里，把它叫做 `emperor` 

```r
# 刻意调整了level的顺序，来对应游戏中吴国势力的红色，蜀国势力的绿色，以及魏国势力的蓝色
emperor <- filter(series_full, 姓名 %in% c("劉備", "曹操", "孫權")) %>%
  mutate(姓名 = factor(姓名, levels = c("孫權", "劉備", "曹操")))
```

用这个函数来画属性对比折线图。

```r
emperor_attr <- function(attrs){
  result <- ggplot(emperor, aes_string(x = "版本", y = attrs, 
    group = "姓名")) + 
    geom_point(size = 2, aes(color = 姓名)) +
    geom_line(aes(color = 姓名)) + 
    geom_text(aes_string(label = attrs), hjust = 1, vjust = -0.5) + 
    ylab(attrs) + 
    ggtitle(paste0("三國主君歷代遊戲", attrs, "對比"))
  result
}
```

#### 2.2.1 武力对比

```r
emperor_attr("武力")
```

![](https://spsufawi.rbind.io/post/RoTC-Analysis_files/figure-html/unnamed-chunk-15-1.png)

《三国志1》作为一款1985年出品的游戏，可能在人物数值上稍欠考究，在这一代中，孙权和曹操的武力达到了惊人的94，93。之后几乎是一直在下降，最终三人的武力都在70上下，基本是个准二流武将的水平。

从资料来看，曹操的武力高于一般士兵是没问题的。

```
兵谋叛，夜烧太祖帐，太祖手剑杀数十人，馀皆披靡，乃得出营；其不叛者五百馀人。——《三国志魏书武帝纪第一》
```

关于孙权的武力的描述不多，不过有记载他曾经干过老虎。虽然是用了武器。

```
二十三年十月，权将如吴，亲乘马射虎於庱亭。马为虎所伤，权投以双戟，虎卻废，常从张世击以戈，获之。——《吴传记》
```

正史上关于刘备的武力几乎没有什么记述，相信光荣对刘备的武力的估计多从演义的来。三英战吕布虽然是虚构的，但我们有理由相信罗贯中的虚构大部分会建立在一定的史实基础上，如果刘备手无缚鸡之力，没拿过武器，估计情节就是双英战吕布了。而且刘备早年讨伐黄巾贼也都是身先士卒，多次亲临沙场，这也是需要勇武的，给个70的武力不为过。

所以孙曹刘武力都最终稳定在70上下，我认为比较科学。

#### 2.2.2 智力对比

```r
emperor_attr("智力")
```

![](https://spsufawi.rbind.io/post/RoTC-Analysis_files/figure-html/unnamed-chunk-16-1.png)

从图中可以看到除了曹操每一代智力都稳定在90以上，孙权和刘备的智力都呈下降趋势，分别稳定在80和70左右，这还是比较合理的，一来是刘备在整部《三国演义》里都没有什么出彩的料敌制胜，智谋过人的表现，在诸葛亮出山以前长期依附着不同势力，从曹操，到袁绍，再到刘表，一直没有自己的一片根据地。当然刘备最大的招牌也不是智谋，是仁德，是能得人心，早期实力不济的时候依然选择救援公孙瓒，孔融等诸侯，带新野襄阳数十万百姓逃到江夏，这也能解释为什么关羽张飞等能一直死心塌地跟着刘备，为什么诸葛亮愿意出山。

而曹操192年领兖州牧，收黄巾贼三十万编制为青州兵，之后虽然和吕布，张绣等作战时互有胜负，但至少作为一方势力已成气候。后边能携天子令诸侯，破吕布，收徐州，败袁绍，在不到二十年的时间里统一北方，靠着是自己的大局观，战略眼光，带兵能力以及智谋。曹操手下能臣谋士虽然多，做整体战略部署谋划，拍板的还是曹操。并不是刘备集团那样寡头制，前期诸葛亮，中期一段时间诸葛亮和庞统，法正去世后就真正的只有诸葛亮一个人了。

```
亮叹曰：「法孝直若在，则能制主上，令不东行；就复东行，必不倾危矣。」
```

孙权18岁接替他哥孙策的位置，接位初期政权不稳，多个州郡发生叛乱，就连孙权的堂兄弟都与曹操内通。而孙权此时一方面继续重用程普，张昭等老臣，一方面广纳贤才，陆逊，鲁肃，诸葛瑾都在这一时期加入了孙权阵营。在完成了江东内的平乱维稳之后，孙权将目标锁定为江夏黄祖，虽未最终攻下江夏，但也多次打败刘表军，并且最终杀死黄祖。之后赤壁之战大获全胜。再后来与曹操在濡须的战斗中，被评价为"生子当如孙仲谋"。虽然孙权一直没打下合肥，并且在逍遥津之战成就了张辽，并且在晚年因继承人问题造成了 [二宫之争](https://zh.wikipedia.org/wiki/%E4%BA%8C%E5%AE%AE%E4%B9%8B%E7%88%AD), 令孙吴折损不少人才(陆逊，步骘的死多少都与此事相关)，但没有完美的人，贾诩也许算一个。

智谋不是决定成败的唯一因素，天时地利甚至个性都影响着曹操刘备孙权的发展进程，但它的确是关键的一环。综上所述，关于三人的智力的数值和变化我认为较为准确。

#### 2.2.3 统率对比

![](https://spsufawi.rbind.io/post/RoTC-Analysis_files/figure-html/unnamed-chunk-17-1.png)

刘备和孙权的统率最终又交汇到了75左右。統率指的是带兵打仗能力，刘备一生虽然败仗打得多，但汉中之战也算是个人巅峰，孙权统治江东五十余年，一直没在曹魏的淮南地区占得便宜，虽说镇守合肥的历来都是曹魏的名将，但也稍微说明了点孙权的打仗能力。

临时决定确认下《三国志1》是否整体数值偏高。

```r
series_full %>% group_by(版本) %>% 
  summarise(智力均值 = mean(智力), 武力均值 = mean(武力))
```

```
# A tibble: 11 x 3
   版本     智力均值 武力均值
   <fct>       <dbl>    <dbl>
 1 三國志1      56.2     57.3
 2 三國志2      56.4     59.3
 3 三國志3      56.6     59.5
 4 三國志4      58.6     61.4
 5 三國志5      59.9     59.2
 6 三國志6      59.1     58.4
 7 三國志7      57.7     58.8
 8 三國志8      56.4     58.6
 9 三國志9      59.4     55.4
10 三國志10     58.5     56.4
11 三國志11     59.0     53.7
```

看来并没有。

### 2.3 历代最强/最弱

#### 2.3.1 历代最强武力

下面这个function是用来算历代游戏中各人物进入最强武力，最强智力，最弱武力等等top10的次数。

```r
# If n is positive, selects the top n rows. If negative, selects the bottom n rows.
top_low_attr <- function(attrs, bw = 1){
  col_name <- enquo(attrs) 
  result <- series_full %>% group_by(版本) %>% 
    top_n(n = bw * 10, !!col_name) %>% 
    group_by(姓名) %>%
    summarise(次數 = n(), 均值 = ceiling(mean(!!col_name))) %>%
    arrange(desc(次數), desc(均值)) %>%
    slice(1:10)
    result
}
```

这跨越了21年的11代游戏里，武力最强者是否有很大变化。从下边这个表格来看，吕布，关羽，张飞，马超，许褚这五个人每一次都进入了最强武力top 10。其它四人也都没有悬念，但太史慈5次进入top这让我有点意外。太史慈出道作品是北海救孔融，成名之作是和孙策的单挑，加入东吴后成功镇压刘磐，总体说来还是十分勇武的，放在游戏中90武力保底没问题，没想到光荣给过多次95以上的武力。

注：这武力均值是该角色进入前10的时候的武力值的平均数

```r
top_low_attr(武力)
```

```
# A tibble: 10 x 3
   姓名    次數  均值
   <chr>  <int> <dbl>
 1 呂布      11  100.
 2 張飛      11   99.
 3 關羽      11   98.
 4 馬超      11   98.
 5 許褚      11   97.
 6 趙雲      10   98.
 7 典韋       9   96.
 8 黃忠       6   96.
 9 文醜       6   96.
10 太史慈     5   96.
```

#### 2.3.2 历代最高智力

把左慈这个神棍去掉的话就有失悬念了，唯一意外的是诸葛亮有一次没进前十，有可能是我的代码没考虑第十名并列的情况，不过诸葛亮智力怎么会排到第十？

```r
top_low_attr(智力)
```

```
# A tibble: 10 x 3
   姓名    次數  均值
   <chr>  <int> <dbl>
 1 龐統      11   98.
 2 司馬懿    11   98.
 3 荀彧      11   97.
 4 周瑜      11   97.
 5 諸葛亮    10  100.
 6 郭嘉      10   98.
 7 陸遜      10   96.
 8 賈詡       9   96.
 9 徐庶       9   96.
10 左慈       4   99.
```

那我们来看下是哪一代诸葛亮掉队了。原來是第七部，这部游戏的人物数值设计师看来是个诸葛亮黑，或者他把诸葛亮其他能力调高了？

```r
series_full %>% filter(姓名 == "諸葛亮")  %>%
  select("智力", "版本")
```

```
智力     版本
1   100  三國志1
2   100  三國志2
3   100  三國志3
4   100  三國志4
5   100  三國志5
6   100  三國志6
7    92  三國志7
8   100  三國志8
9   100  三國志9
10  100 三國志10
11  100 三國志11
```

嚯，原来如此，在历代诸葛亮武力都在5，60，甚至3，40的情况下，第七部诸葛亮一跃成为了一名武力为87的武将，有理由相信这是光荣为了平衡而nerf诸葛亮智力的原因。

```r
series_full %>% filter(姓名 == "諸葛亮") 
```

```
    姓名 統率 武力 智力 魅力 運勢     版本 政治 陸指 水指 體力
1  諸葛亮   NA   72  100   97   84  三國志1   NA   NA   NA   69
2  諸葛亮   NA   65  100   98   NA  三國志2   NA   NA   NA   NA
3  諸葛亮   NA   61  100   95   NA  三國志3   92   92   78   NA
4  諸葛亮   97   55  100   96   NA  三國志4   96   NA   NA   NA
5  諸葛亮   NA   60  100   97   NA  三國志5   96   NA   NA   NA
6  諸葛亮   97   55  100   98   NA  三國志6   98   NA   NA   NA
7  諸葛亮   NA   87   92   95   NA  三國志7   98   NA   NA   NA
8  諸葛亮   NA   50  100   91   NA  三國志8   98   NA   NA   NA
9  諸葛亮   92   33  100   NA   NA  三國志9   98   NA   NA   NA
10 諸葛亮   93   37  100   92   NA 三國志10   98   NA   NA   NA
11 諸葛亮   92   38  100   92   NA 三國志11   95   NA   NA   NA
```

#### 2.3.3 历代最弱智力

提到兀突骨可能有人不认识，但估计都知道诸葛亮七擒孟获，孟获被抓了六次之后第七次找的藤甲兵援军的老大就是他。(我一直把他和沙摩柯搞混了)。兀酱是三国演义虚构的角色，虽然自己的三万人中了诸葛亮的火攻全军覆没，也不至于智力在11作游戏中九次排倒数吧，为兀突骨鸣不平。

这表格中少数部落的人占了50%，兀突骨，忙牙长，金环三结，孟优都是南蛮人，拥有三国演义最喜感名字的俄何燒戈是羌族武将（迷当大王，郝萌φ(￣∇￣o)不服）。演义中基本蛮人羌人作为援军出场的下场都不太好。刘备伐吴请来的沙摩柯被周泰所杀，雅丹和越吉一个被活捉一个被杀，接着就是俄何烧戈了（历史上俄何和烧戈是两个人）。

```r
top_low_attr(智力, -1)
```

```
# A tibble: 10 x 3
   姓名      次數  均值
   <chr>    <int> <dbl>
 1 兀突骨       9    9.
 2 俄何燒戈     7   13.
 3 潘鳳         7   12.
 4 忙牙長       7   10.
 5 楊秋         5   16.
 6 金環三結     5   15.
 7 蔡和         5    9.
 8 王雙         4   16.
 9 曹豹         4   14.
10 孟優         3   12.
```

其它人物没太多可说的。蔡和，被曹操喊去卧底，估计一身演技还没施展，就被周瑜识破了。

上将潘凤打不过华雄为啥智力那么低，何况也是韩馥让他去的，他的大斧饥渴难耐那是电视剧给他加的台词。都怪罗贯中。

```
“末将遵命！取兵器来！” 《央视版三国》
“小小娃娃口出狂言，我乃潘凤，快来送死！” 《貂蝉》
“华雄的拳在真正天才的眼里，不过是慢动作而已。” 《终极三国》
“有何不敢？我的大斧早就饥渴难耐了！” 《三国》
```

#### 2.3.4 历代最差魅力

宦官黄皓，佞臣岑昏入围比较没悬念，演义中被描述成”平生性急，轻于杀戮，众皆恶之“的韩玄，卖了张鲁协助曹军最后却身首异处的杨松，投降东吴的糜芳都比较合理。

```r
top_low_attr(魅力, -1)
```

```
# A tibble: 10 x 3
   姓名      次數  均值
   <chr>    <int> <dbl>
 1 兀突骨       9    9.
 2 俄何燒戈     7   13.
 3 潘鳳         7   12.
 4 忙牙長       7   10.
 5 楊秋         5   16.
 6 金環三結     5   15.
 7 蔡和         5    9.
 8 王雙         4   16.
 9 曹豹         4   14.
10 孟優         3   12.
```

### 2.4 智勇双全

#### 2.4.1 有勇有谋

下面来看下在武力比智力高的人里智力最高的是谁

```r
war_int <- series_full %>% filter(武力 > 智力) %>% 
  group_by(版本) %>%
  top_n(n = 1, 智力) %>%
  select(c("姓名", "武力", "智力"))

table(war_int$姓名)
```

```
曹操 郝昭 李嚴 孫策 徐盛 張遼 張任 趙雲 
   1    1    1    3    2    4    1    3 
```

孙策，赵云，张辽， 徐盛，张任, 郝昭，这些确实都是三国里最厉害的一群人物。

#### 2.4.2 文武双全

智力比武力高的人里武力最高的会有谁。

```r
int_war <- series_full %>% filter(智力 >= 武力) %>% 
  group_by(版本) %>%
  top_n(n = 1, 武力) %>%
  select(c("姓名", "武力", "智力"))

table(int_war$姓名)
```

```
曹操 姜維 孫權 
   2    9    1 
```

基本都是姜伯约，不过光荣公司把姜维的武力智力调换下相信也不会有人有意见。

再来看下武力+智力最高的人是谁，是否和上边的人物有所出入

```r
war_int2 <- series_full %>%  mutate(智勇 = 智力 + 武力) %>%
  group_by(版本) %>%
  top_n(n = 1, 智勇)
table(war_int2$姓名) 
```

```
曹操 姜維 孫權 
   2    9    1 
```

基本没区别。

### 2.5 五虎大将 vs 五子良将

来对比下蜀国五虎大将和五子良将吧。先来放一张光荣风格的武将头像图。

![](https://spsufawi.rbind.io/content/post/RoTC-Analysis_files/wuhua-vs-wuzi.jpg)

五子良将指的是魏国将领张辽，徐晃，乐进，张郃，于禁。这个称号来自陈寿的《三国志》

```
太祖建茲武功，而时之良將，五子为先 --陈寿 《三国志魏志卷十七》
```

正史上并没有记载有五虎将这个称号，陈寿将关羽，张飞，赵云，黄忠，马超合编到了《三国志·蜀书·关张马黄赵传》中，后来人们就以此构想出了五虎上将，可能也是受到了五子良将的启发。

#### 2.5.1 武力对比

```r
wuzi <- c("張郃", "徐晃", "張遼", "于禁", "樂進")
wuhu <- c("關羽", "張飛", "趙雲", "馬超", "黃忠")

general <- series_full %>% filter(姓名 %in% c(wuzi, wuhu)) %>% 
  mutate(稱呼 = ifelse(姓名 %in% wuzi, "五子良將", "五虎大將"))

wuhu_vs_wuzi <- function(attrs){
  ggplot(general, aes_string(x = "版本", y = attrs, group = "姓名")) + 
    geom_point(size = 2, aes(color = 稱呼)) + 
    geom_line(aes(color = 稱呼)) +
    scale_y_continuous(limits = c(0, 100), breaks = seq(0, 100, 10)) + 
    geom_text(data = filter(general, 版本 == "三國志1"), 
      aes(label = 姓名, hjust = 1, vjust = -1)) +
    ggtitle(paste0("五子良將和五虎大將歷代", attrs, "對比"))
}
```

```r
wuhu_vs_wuzi("武力")
```

![](https://spsufawi.rbind.io/post/RoTC-Analysis_files/figure-html/unnamed-chunk-30-1.png)

从这个图来看基本五虎武力是碾压五子的，而五子里又属张辽徐晃武力最高。前者没有什么好说的，毕竟关羽赵飞的“万人敌”的称号是《三国志魏书程昱传》里记载的。赵云先不说在历史上真实形象如何，就凭他在演义里长坂坡，汉水拒敌的表现，以及他的人气，光荣就不敢不给赵云95以上的武力。五虎将的武力历代基本没什么变化，维持在95左右。乐进和于禁的武力在2代一度跌破60，后边走高达到了80多。

张辽不说太多，我们来看下曹丕，孙权对他的评价。

```
曹丕：「此亦古之召虎也。」「合肥之役，辽、（李）典以步卒八百，破贼十万，自古用兵，未之有也。
      使贼至今夺气，可谓国之爪牙矣。其分辽、典邑各百戶，赐一子爵关內侯。」
孙权：「张辽虽病，不可当也，慎之！」
```

介绍徐晃的武力从演义入手比较好。徐晃的武力历代都在90左右, 说明这个数值基本没什么可变化的理由。徐晃主要战绩为五十回合战平许褚，二十回合败给颜良，樊城之战八十回合打退手臂有伤年近六十的关羽。二十回合输给颜良可以说确实是完敗，但颜良可是《三国演义》里武力最顶尖的人物，被关羽一刀秒杀是关于趁人不备，演义原文是这么写的。

```
颜良正在麾下，见关公到来，恰欲问之，马已至近。云长手起，一刀斩颜良于马下。
```
所以综合1v1的战绩，加上徐晃潼关之战和樊城打败关羽的武功，徐晃的武力值没什么问题。

乐进在三国志1和三国志2武力都小于60，这几乎是没什么依据的。下边是《三国志-张乐于张徐传》记载的，

```
从征张绣於安众，围吕布於下邳，破别将，击眭固於射犬，攻刘备於沛，皆破之，...，
从击袁绍於官渡，力战，斩绍将淳于琼。从击谭、尚於黎阳，斩其大将严敬，行游击将军。
别击黄巾，破之，...，从击袁谭於南皮，先登，入谭东门。谭败，别攻雍奴，破之
```

从乐进传里来看，乐进参加的战斗几无败仗，作战勇猛，身先士卒，陈寿给予评价“骁果”，意为勇猛刚毅。

最后来看看张郃，还是《三国志魏书十七》

```
又与张辽讨陈兰、梅成等，破之。从破马超、韩遂於渭南。围安定，降杨秋。
与夏侯渊讨鄜贼梁兴及武都氐。又破马超，平宋建 ...,
...,
备以精卒万馀，分为十部，夜急攻郃。<u>郃率亲兵搏战</u>，备不能克。
依阻南山，不下据城。郃绝其汲道，击，大破之。南安、天水、安定郡反应亮，郃皆破平之。
```

于禁就不单独说了，武力变化不大，最后投降关羽晚节不保，也许影响了一点他的数值。

#### 2.5.2 智力对比

```r
wuhu_vs_wuzi("智力")
```

![](https://spsufawi.rbind.io/post/RoTC-Analysis_files/figure-html/unnamed-chunk-31-1.png)

智力从图来看五子除了张辽总体趋势都是在上升的，并且平均来看高于五虎将，主要是张飞和马超拖了后腿，而黄忠马马虎虎，智力维持在60上下。

感觉相对于五虎大将，五子良将更多的是属于帅才，武力不一定是最顶尖的，但带兵打仗，以弱胜强，摧城拔寨的本事要强于五虎大将。

### 2.6 数值变化最大人物

#### 2.6.1 武力值变化最大人物

先来看武力变化比较大的人是谁

```r
series_full %>% group_by(姓名) %>% 
  summarise(武力變化 = max(武力) - min(武力)) %>%
  top_n(10, 武力變化) %>%
  arrange(desc(武力變化))
```

```
# A tibble: 10 x 2
   姓名  武力變化
   <chr>    <dbl>
 1 孔融       77.
 2 貂蟬       71.
 3 鍾繇       68.
 4 韓馥       66.
 5 小喬       65.
 6 大喬       63.
 7 丁奉       62.
 8 曹熊       61.
 9 裴秀       60.
10 劉繇       59.
```

又是一位建安七子，武力变化最大的是孔融让梨故事的主人公孔文举。孔融武力最高达到了82，最低却只有5。也许考虑到了孔融早期为十八路诸侯中的一路，参与了讨伐董卓。但无论是正史还是演义，记载的孔融打过的仗基本都是败的。后汉书有个很有意思的记载。

```
《后汉书·卷七十》：「建安元年，为袁谭所攻，自春至夏，战士所余裁数百人，
流矢雨集，戈矛內接。融隐机读书，谈笑自若。城夜陷，乃奔东山，妻子为谭所虏。」
```

和之后孔融小孩的”覆巢之下，安有完卵乎”有异曲同工之妙。

```r
series_full %>% filter(姓名=="孔融")
```

```
   姓名 統率 武力 智力 魅力 運勢     版本 政治 陸指 水指 體力
1  孔融   NA   82   61   50   77  三國志1   NA   NA   NA   84
2  孔融   NA   35   82   87   NA  三國志2   NA   NA   NA   NA
3  孔融   NA   58   83   64   NA  三國志3   76   67   63   NA
4  孔融   68   51   82   65   NA  三國志4   76   NA   NA   NA
5  孔融   NA   37   89   72   NA  三國志5   75   NA   NA   NA
6  孔融   63   48   85   71   NA  三國志6   75   NA   NA   NA
7  孔融   NA   56   75   59   NA  三國志7   67   NA   NA   NA
8  孔融   NA   32   68   56   NA  三國志8   74   NA   NA   NA
9  孔融   23    7   69   NA   NA  三國志9   64   NA   NA   NA
10 孔融   30   11   74   60   NA 三國志10   78   NA   NA   NA
11 孔融   30    5   72   65   NA 三國志11   75   NA   NA   NA
```

#### 2.6.2 智力值变化最大人物

```r
series_full %>% group_by(姓名) %>% 
  summarise(智力變化 = max(智力) - min(智力)) %>%
  top_n(10, 智力變化) %>%
  arrange(desc(智力變化))
```

```
# A tibble: 11 x 2
   姓名   智力變化
   <chr>     <dbl>
 1 鮑信        67.
 2 韓玄        66.
 3 劉璋        65.
 4 穆順        57.
 5 橋玄        56.
 6 曹訓        55.
 7 全琮        54.
 8 蔡中        53.
 9 韓浩        53.
10 審配        53.
11 於夫羅      53.
```

韩玄智力变化非常大，在历史上韩玄其实和小说里的形象相差甚远，以下摘自维基百科。

```
清代汪应铨《韩玄墓记》载；韩玄“威信智略，足以服人”，“宽厚爱人，玄与三郡俱降，兵不血刃，百姓安堵，可谓知顺逆之理，
有安全之德。”对韩玄评价甚高，似在为其正名。
```

所以光荣对于韩玄的设计，更多的参考了演义，可这类角色谁又会关心呢。

再来说下刘璋。三國志9和三國志11給出了慘不忍睹的5的智力值。我对刘璋最大的印象是"暗弱"，史书对刘璋的评价也普遍是非人雄，非英杰，愚弱。但却有这么一个评价说了点他的功。

```
“刘璋虽暗懦，然国富民盛，守之以恩，无所得罪也。” --叶适（《习学记言》）
```

胸无大志不是错，刘璋的目标可能就是偏安一隅，能让国富民安就挺好。无奈刘备的目标是天下，占荆州，夺西蜀，待天下有变。
当时看演义，读到黄权用牙齿咬着刘璋的衣服不让刘璋去涪城见刘备，王累以死相逼都拦不住，真是一声叹息。蜀中多才俊，无奈碰到刘璋这个主君。

```r
series_full %>% filter(姓名 == "劉璋")
```

```
   姓名 統率 武力 智力 魅力 運勢     版本 政治 陸指 水指 體力
1  劉璋   NA   51   70   94   60  三國志1   NA   NA   NA   74
2  劉璋   NA   50   70   90   NA  三國志2   NA   NA   NA   NA
3  劉璋   NA   52   51   82   NA  三國志3   63   47   21   NA
4  劉璋   48   53   50   82   NA  三國志4   63   NA   NA   NA
5  劉璋   NA   33   60   85   NA  三國志5   43   NA   NA   NA
6  劉璋   38   31   63   87   NA  三國志6   55   NA   NA   NA
7  劉璋   NA   27   48   53   NA  三國志7   46   NA   NA   NA
8  劉璋   NA   17   37   80   NA  三國志8   46   NA   NA   NA
9  劉璋    3    3    5   NA   NA  三國志9   33   NA   NA   NA
10 劉璋   18   15   10   75   NA 三國志10   45   NA   NA   NA
11 劉璋   16    5    9   65   NA 三國志11   38   NA   NA   NA
```

### 2.7 重名人物

玩《三国志10》的时候看到过两个马忠，两个李丰，那么还有没有别的重名人物呢。

```r
series_full %>% group_by(版本, 姓名) %>%
  summarise(人数 = n()) %>%
  filter(人数 > 1)
```

```
# A tibble: 16 x 3
# Groups:   版本 [9]
   版本     姓名   人数
   <fct>    <chr> <int>
 1 三國志3  李豐      3
 2 三國志3  馬忠      2
 3 三國志4  馬忠      2
 4 三國志5  馬忠      2
 5 三國志6  馬忠      2
 6 三國志7  馬忠      2
 7 三國志8  李豐      2
 8 三國志8  馬忠      2
 9 三國志8  張溫      2
10 三國志9  李豐      2
11 三國志9  馬忠      2
12 三國志10 李豐      3
13 三國志10 馬忠      2
14 三國志11 李豐      3
15 三國志11 馬忠      2
16 三國志11 張南      2
```

从结果来看，一共有三个李丰，两个马忠，两个张温，两个张南。 我每个都查了下，确实如此，不是数据错误。里边最出名的相信是东吴那个抓住了关羽的马忠了。

## 3. 后记

在这篇文章完成并成功发到用blogdown建成的网站后，真正体会到了什么叫“书到用时方恨少，事因经过始知难”。写之前本以为凭借自己对三国的了解，应该来说写起来不是很困难，结果发现我知道的东西和别人聊天用是足够，但是写下来就不行了，只知其一不知其二，都需要现查。书读的不少，文字水平却没感觉提高，对自己写的东西读起来总觉得别扭，有时一句话多次删了又加上，用词造句为了避免重复，也是煞费苦心。想起初中高中写英语作文的时候老师常说的，你这里用了think，后边就换个词，比如consider, believe, suggest. 

对这篇文章里写的R还是比较满意的，相对来说比较简练，尽量减少了变量数量和行数，在google过程中也学到了不少东西。

另外文中关于人物评价有部分是自己主观的想法，也许经不起推敲，如果有什么错误的地方，欢迎讨论和指正。
