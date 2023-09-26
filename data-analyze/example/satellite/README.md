# 需求背景

​	该背景主要是对GPS定位卫星的数据进行解析和整理。需要注意如下几点：

​	（1）数据集中出现的相关业务指标含义无需深入了解。

​	（2）数据集中出现的时间戳可以无需关注，对于卫星来说更关注周内秒，默认数据集中时间维度均为同一周。

# 数据集

## 一.[bdStar_front.23O](./data/bdStar_front.23O)

​	该数据为非结构化数据集（以下简称“数据集1”），需要解析为结构化的数据集，在数据集中有价值的数据域大致如下：

```shell
> 2023 08 08 07 21 15.5000000  0 22      -0.000000000000
G18  20978262.711 6 110241526.633 6      1315.195          40.000    20978263.750 6  85902475.754 6      1024.828          36.000    20978264.188 7  85902480.758 7      1024.832          42.000  
G15  20304996.477 8 106703480.543 8     -1119.594          51.000    20304997.742 8  83145564.680 8      -872.414          51.000    20304997.742 8  83145571.680 8      -872.414          51.000  
> 2023 08 08 07 21 15.6000000  0 22      -0.000000000000
G18  20978237.695 6 110241395.172 6      1314.719          40.000    20978238.727 6  85902373.316 6      1024.461          38.000    20978239.172 7  85902378.316 7      1024.461          42.000  
G15  20305017.781 8 106703592.512 8     -1119.695          51.000    20305019.047 8  83145651.930 8      -872.492          51.000    20305019.047 8  83145658.926 8      -872.492          51.000  
G20  24365048.484 7 128039160.227 7     -2696.531          47.000    24365059.672 6  99770779.531 6     -2101.203          41.000                                                                  
> 2023 08 08 07 21 15.7000000  0 22      -0.000000000000
C13 
```

​	（1）以“>”开头的为一次时间戳，时间戳需要截取到毫秒，其余的字符串可以忽略。

​	（2）“>”开头的下一行开始为实际数据，这表示在这个时间下所产生的数据。

​	（3）在实际数据中取前7列为有效列，例如：

```shell
G18  20978262.711 6 110241526.633 6      1315.195          40.000    20978263.750 6  85902475.754 6      1024.828          36.000    20978264.188 7  85902480.758 7      1024.832          42.000  
```

​		实际的有效列为：

```shell
G18  20978262.711 6 110241526.633 6      1315.195          40.000
```

​		这里的"G18"可以理解为在这个时间点下的某个卫星编号，后六列为该卫星的核心指标。

​	（4）以“>”开头的时间戳需要解析为卫星周内秒，供后续计算。

## 二.[stim300_Inertial_data_m](./data/stim300_Inertial_data_m.txt)

​	该数据为结构化数据集（以下简称“数据集2”）：

​	（1）第一列为卫星周内秒。

​	（2）其余列为当前周内秒下的卫星轨道指标。

## 三.[refPath_frontAnt](./data/refPath_frontAnt)

​	该数据为非结构化数据集（以下简称“数据集3”），该数据集表示在卫星在周内秒的时刻下的某种状态指标，特别是经纬度坐标，需要将其解析为结构化数据，在解析过程中可能会有损坏的行无法读取，需要通过某种策略进行剔除。该数据集非常重要，可以理解为机器学习中的标签列。

# 规整策略

## 关联规则

​	**关联特征**：时间维度（统一换算成周内秒，没有周内秒特征的需要通过其他方式解析）。

​	**关联策略**：所有数据集通过周内秒进行关联（内链接）。周内秒在进行关联的时候尽可能进行完全匹配关联，如果不能进行完全匹配关联，尽可能关联到毫秒。具体如下：

​	1⃣️ 完全匹配

| 表名（A）     | 表名（B）     |
| ------------- | ------------- |
| 199293.700000 | 199293.700000 |

​	这种情况下就可以完全匹配。

​	2⃣️ 尽可能匹配

| 表名（A）     | 表名（B）     |
| ------------- | ------------- |
|               | 199293.690000 |
| 199293.700000 | 199293.708000 |
|               | 199293.720000 |
|               | 199293.780000 |

​	这种情况下需要尽可能的匹配，让需要关联的周内秒矫正到同一个100毫秒内，例如199293.700000只能关联[199293.700000, 199293.800000)区间内的时间。

​	在这种情况下分别求每个差值的绝对值，然后取绝对值最小的那个周内秒，例如周内秒199293.700000和{199293.708000, 199293.720000, 199293.780000}分别求差值的绝对值的为：{0.008, 0.02, 0.08}，这里去最小的0.008对应的那个周内秒即：199293.708000。

## 映射规则

​	通过关联规则之后将卫星指标整合到一张宽表中，主要如果有指定字段存在缺失值则整行进行剔除。具体如下：

​	【数据集1】解析的YYYYMMDD风格的时间戳，解析的周内秒，该时刻多个卫星的核心指标（参考**附录：周内秒卫星核心数据**）。

​	【数据集2】所有字段

​	【数据集3】所有字段



# 附录

## 周内秒卫星核心数据

### 数据透视

​	在同一个周内秒下有多个卫星，一颗卫星有6个核心指标。这些数据主要来源于数据集1，大致如下：

​	在199929.300000这个周内秒下数如下：

|      | X1   | X2   | X3   | X4   | X5   | X6   |
| ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| A02  |      |      |      |      |      |      |
| G81  |      |      |      |      |      |      |

​	需要进行数据透视成如下的样子：

| GPST          | A02_X1 | A02_X2 | A02_X3 | A02_X4 | A02_X5 | A02_X6 | G81_X1 | G81_X2 | G81_X3 | G81_X4 | G81_X5 | G81_X6 |
| ------------- | ------ | ------ | ------ | ------ | ------ | ------ | ------ | ------ | ------ | ------ | ------ | ------ |
| 199929.300000 |        |        |        |        |        |        |        |        |        |        |        |        |

### 缺失值处理

​	在同一个周内秒下有多个卫星，在不能时刻下出现的卫星有可能有一致的，也有可能完全不一致，也有可能都有。

​	例如：在T0时刻出现了卫星{A02, G81}，在T1时刻出现了卫星{G81, J81}，根据上述的数据透视规则个整理如下：

| GPST | A02_X1 | A02_X2 | A02_X3 | A02_X4 | A02_X5 | A02_X6 | G81_X1 | G81_X2 | G81_X3 | G81_X4 | G81_X5 | G81_X6 | J81_X1 | J81_X2 | J81_X3 | J81_X4 | J81_X5 | J81_X6 |
| ---- | ------ | ------ | ------ | ------ | ------ | ------ | ------ | ------ | ------ | ------ | ------ | ------ | ------ | ------ | ------ | ------ | ------ | ------ |
| T0   | ✅      | ✅      | ✅      | ✅      | ✅      | ✅      | ✅      | ✅      | ✅      | ✅      | ✅      | ✅      |        |        |        |        |        |        |
| T1   |        |        |        |        |        |        |        |        |        |        |        |        | ✅      | ✅      | ✅      | ✅      | ✅      | ✅      |

​	这种情况下透视表就有非常的稀疏，同时也会出现大量的缺失值。

​	在本需求中需要将卫星的核心指标整理的更加稠密。整理的大致规则如下：

​	1⃣️ 某个周内秒时间点有N颗卫星，每颗卫星有6个核心指标，则每个周内秒时间点共计4N颗卫星。

​	2⃣️ 某个周内秒时间点卫星个数可能很多，可能很少，也可能没有，如果卫星个数小于4颗，则剔除该周内秒时间点的数据。

​	3⃣️ 由于卫星编号不同导致每个卫星核心特征不同，这里无需关注整理出来的宽表特征中是哪个卫星的哪个指标，例如：卫星G18的6个特征在稀疏表中的特征名称为:{G81_X1, G81_X2, G81_X3, G81_X4, G81_X5, G81_X6}; 卫星A02的6个特征在稀疏表中的特征名称为:{A02_X1, A02_X2, A02_X3, A02_X4, A02_X5, A02_X6}，在最后整理出的稠密表中可以编号为：{X1, X2, ..., X8}。

​	4⃣️ 某个周内秒时间点卫星个数只去4颗卫星指标，共计24个特征。如果某个周内秒时间点卫星个数有多个则按照卫星编号的字典顺序取满4颗卫星的数据。例如：T0时刻有卫星：{G18, A02, J98, B06, Z09, H77}，按照规则只取 {A02, B06, G18, H77}这4颗卫星。如果无法理解可以参考《**附录：卫星核心指标候补逻辑**》

## 卫星核心指标候补逻辑

​	在同一个周内秒时间点中，所有卫星的多个核心指标可能是参差不齐的（如果按照数据对其方式理解为稀疏表），需要通过列站位候补的方式去除其中缺失值，同时保留预定数量的列。下面以保留4列为例。

【例1】

​	处理前：

|       | A    | B    | C    | D    | F    |
| ----- | ---- | ---- | ---- | ---- | ---- |
| Value | 1    | NA   | 4    | 3    | 7    |

​	处理后：

|       | X1   | X2   | X3   | X4   |
| ----- | ---- | ---- | ---- | ---- |
| Value | 1    | 4    | 3    | 7    |

【例2】

​	处理前：

|       | A    | B    | C    | D    | F    |
| ----- | ---- | ---- | ---- | ---- | ---- |
| Value | 1    | 3    | 4    | 3    | 7    |

​	处理后：

|       | X1   | X2   | X3   | X4   |
| ----- | ---- | ---- | ---- | ---- |
| Value | 1    | 3    | 4    | 3    |

【例3】

​	处理前：

|       | A    | B    | C    | D    | F    | X    | Y    |
| ----- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| Value | NA   | NA   | 4    | 3    | 7    | 9    | 1    |

​	处理后：

|       | X1   | X2   | X3   | X4   |
| ----- | ---- | ---- | ---- | ---- |
| Value | 4    | 3    | 7    | 9    |

​	
