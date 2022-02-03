# readme

[TOC]

## 概览

### 任务描述

​	本次任务需要对A股所有个股单个交易日（2021.07.06）毫秒级成交数据进行读取和汇总，并绘制成不同时间频率的K线图，实现股票行情回放和交互式K线图功能。本次任务的难点在于对大型原始数据的读取和处理，读取方面需要兼顾效率与实时性；处理方面需要将毫秒级别数据再采样成不同频率的K线数据，并计算成交量、均线、MACD等衍生指标。在数据读取和清洗后，需要对K线数据进行可视化处理，在该过程需要评估多种开源组件进行评估，在该过程需要兼顾数据交互的效率和绘图功能的多样性。最后，需要对原始数据设计数据采样接口，返还特定数据。

### 项目概览

​	本次任务使用`python3.8`作为编程语言，并使用`Jupyter Notebook`作为编译器，使用的第三方库及版本为：

|  第三方库  |   版本   |  第三方库  | 版本  |
| :--------: | :------: | :--------: | :---: |
| mplfinance | 0.12.8b6 | matplotlib | 3.5.1 |
| pyecharts  |  1.9.1   |   pandas   | 1.4.0 |
|   ta-lib   |  0.4.24  |  ipython   | 8.0.1 |

​	通过对`pyarrow`、`pandas.read_csv`、`dask.dataframe`等数据读取工具进行评估后，我选择使用`dask.dataframe`进行数据读取，并筛选出个股子集并保存为`csv`格式，方便后续数据处理。通过对`matplotlib.finance`、`highchart`、`pyecharts`的交互效率、数据吞吐量和自定义功能进行评估后，我选择使用`pyecharts`对数据进行可视化处理，因为其交互效率高、自定义功能多，更能够满足项目需求。

### 	功能概览

​	本次项目可以很好的完成基本任务要求，在大批量数据读取和数据处理方面表现良好，在数据可视化和交互方面具有一定亮点，任务实现方式与对应函数如下表所示：

|          **任务概览**          |                         **实现方式**                         | 函数名称      |
| :----------------------------: | :----------------------------------------------------------: | ------------- |
|     读取原始大型`CSV`数据      | 使用`dask.dataframe`读取后抽取个股子集数据，以`csv`格式存储  | _readData     |
|        股票行情倍速回放        |       使用`echarts.timeline`回放股票行情，并可调整速度       | echartsAction |
| 接入行情回放的交互式蜡烛图界面 | 使用`echarts.kline`和`echarts.timeline`可实现行情回放暂停并交互 | echarts       |
|       筛选不同的采样频率       | 使用`dataframe.resampe`和`echarts.tab`功能切换不同频率K线图  | _loadData     |
|     实时展现鼠标所在点信息     |       可显示K线四个价格、均线、成交量、衍生指标等信息        | echarts       |
|          添加更多指标          |              `MACD`、`KDJ`、均线等，支持自定义               | _loadData     |
|      成批量的接入原始数据      |      框架可对原始数据直接处理并切分成小数据加快处理速度      | get_data      |
|          采样数据接口          |       通过`dask.dataframe`切分子集后返回个股处理后数据       | get_data      |

### 内容结构

​	本项目内容由三块组成：存储数据的`data`, 存放展示实例的`demo`和存放代码的`src`。`src`的代码构成为：

1. `ipython_importer.py`：导入`ipynb`文件中的类；
2. `_readData.ipynb`: 使用`pandas.chunk`方法批量读取原始数据，并将原始数据拆分成个股`csv`形式；
3. `_loadData.ipynb`：读取个股数据，若个股数据尚未被拆分则从源数据中进行拆分；读取数据后对数据进行清洗，并转换成可以进行可视化的数据类型；
4. `_interCandle.ipynb`: 使用`matplotlib.finance`库构建的蜡烛图类；
5. `mplfinance.ipynb`: 导入`_interCandle.ipynb`中类进行蜡烛图绘制；
6. `echarts.ipynb`: 使用`pyecharts`库进行可交互的蜡烛图绘制；
7. `echartsAction.ipynb`: 使用`pyechatrs`库绘制行情回放；
8. `get_data.ipynb`: 数据获取接口

<img src="C:\Users\Yip\AppData\Roaming\Typora\typora-user-images\image-20220204002953656.png" alt="image-20220204002953656" style="zoom:50%;" />

`demo`中存放有内容展示的PPT文稿、展示如何使用接口和绘图的`mp4`文件、可以进行交互的`html`文件。

***K线图最终呈现效果可在demo文件夹中`html`文件内查看***

​	以上内容是对项目功能的总结性描述，以下内容将对项目功能进行分点细节性描述。 

------



## 数据读取

### 数据概览

​	本次任务原始数据为2021年7月6日A股所有个股毫秒级别的成交数据，共有1574万笔成交记录，45类数据，包含4379支个股。由于最终任务仅涉及个股数据，因此数据处理大体思路为：

> 所有数据读取 ➡️根据所需股票代码抽取子集➡️对子集数据进行清洗和存储➡️对子集数据进行可视化处理

### 数据读取

​	针对本次任务数据，我设计了两种数据读取方法：

1. 使用`pandas.dataframe.chunk`对原始数据进行分块读取后，将每一块数据分割成个股数据，并对每一块个股数据进行拼接，保存全部个股数据。该方式需要较长时间的数据初始化，但使用时调用速度快。
2. 使用`dask.dataframe`实时获取数据，每次绘制K线图时从源数据直接获取所需个股的K线数据并保存。该方式不需要数据初始化过程，但在绘图前需要读取原始数据中个股K线数据，需要一定调用时间。

#### `pandas.chunk`方法

​	为减少计算机读取数据时占用内存，加快读取速度，可以在调用`pandas.read_csv`方法时指定读取区块大小，并对每一区块进行单独处理。在该方法中，我首先读取了源数据中所有股票代码`tickerName`并去除重复值后以`tickle`形式保存。此后，指定数据每个区块大小，并匹配每个区块中个股对应数据，合并数据后以`csv`格式存储到本地，等待可视化操作时直接提取数据。该方法核心代码为：

```python
stockList = pd.read_pickle("../data/stockDivideData/uniqueStockList.pkl")
reader = pd.read_csv(file_path, sep=',', chunksize=1000000)
for i, chunk in enumerate(reader):
    for stockCode in stockList[:100]:
        stockFilePath = "../data/stockDivideData/StockData/" + stockCode + ".csv"
        if(i==0):
            chunk[chunk['tickerName']==stockCode].to_csv(stockFilePath)
        else:
            chunk[chunk['tickerName']==stockCode].to_csv(stockFilePath, mode="a", header=False)
```

该方法的优点在于只需对原始数据进行一次初始化处理，将初始化步骤与数据可视化步骤分开，在调用部分效率更高，但是需要更多初始化时间。

#### `dask.dataframe`方法

​		由于读取数据量较大，直接以`dataframe`形式读取效率较低，因此我选择使用`dask.dataframe`的并行方式对所有数据进行读取，而后对数据进行切片和清洗。`dask`是一个开源的python并行计算库，很好地适配了`pandas.dataframe`类型。使用`dask`读取源文件，可以获得：

```python
import dask.dataframe as dd
reader = dd.read_csv(
    "../data/originData.csv", 
    dtype={
        "localTime": str,
        "localTimeUnderMs": str,
    })
reader
```

<img src="C:\Users\Yip\AppData\Roaming\Typora\typora-user-images\image-20220202183140880.png" alt="image-20220202183140880" style="zoom:50%;" />

当前文件被分隔成63个可并行运算的任务，大大加快了文件读取和切片的效率。此后，程序可根据所需的股票数据从源数据中直接进行提取，由于使用了`dask.delayed`惰性调用方法，且通过并行计算加快运行速度，该调用方法的效率较高，将数据处理与可视化步骤合为一体，避免对所有数据进行预处理部分较长的初始化时间。但由于未经预处理，在每次调用时需要对源数据进行切片获取，因此在单个数据调用部分效率略低于上一方法。

#### 小结

​	在本项目中，我最终选择了使用`pandas.chunk`预处理一部分常见股票数据存储在本地数据中，并使用`dask.dataframe`获取本地未存储的股票数据。

## 数据清洗

​	数据清洗部分的核心任务是将tick级别数据聚合成不同频率的K线数据，并构建衍生数据以进行数据可视化。

#### 时间转换

​	首先，需要对源数据的时间进行转换，从UNIX结构转换成真实时间。该转换的大致步骤为：

1. 将`localTime`和`localTimeUnderMs`位数对齐并合并，转换成`int`类型；
2. 将合并后的`int`类型数据使用`pandas.to_datetime`转换成真实时间。

​	该转换的核心代码为：

```python
def time_adjust(self):
    self.initDf[['localTime', 'localTimeUnderMs']] = self.initDf[['localTime', 'localTimeUnderMs']].astype(str)
    self.initDf['localTimeUnderMs'] = self.initDf['localTimeUnderMs'].str.zfill(6)
    self.initDf["Time_use"] = self.initDf.apply(lambda x: pd.to_datetime(int(x['localTime']+x['localTimeUnderMs']), \
                                               origin = pd.Timestamp(self.tradeDay)), axis=1)  # Convert unix time to readable time.
    self.initDf.index = self.initDf["Time_use"]
    self.useDf = self.initDf.copy()
    print("Adjusting Time...")
```

#### K线四价转换

​	绘制K线时需要知道给定时间段中开盘价、收盘价、最高价和最低价。 因此，该转换的核心步骤在于数据聚合。通过对时间进行转换，并将时间设为数据索引，就可以使用`dataframe.resample`方法，在每个时间段对数据进行聚合。K线四价所用的价格数据是源数据中的`last`数据，对数据聚合后通过`resample.max()`获取该时段最高价，`resample.min(), resample.first(), resample.last()`获取该时段的最低价、开盘价和收盘价。

```python
def data_combine(self):
    freq = self.freq
    df_use = self.useDf.resample(freq, closed="right")
    df_concat = pd.DataFrame([df_use["last"].max(), df_use["last"].min(), \
                              df_use["last"].first(), df_use["last"].last(), df_use['volume'].sum(), \
                              df_use['turnover'].sum()]).T
    df_concat.columns = ["high", "low", "open", "close", "volume", "turnover"]
    return df_concat
```

#### 成交量转换

​	源数据记录的成交量是从开盘至该时刻的累计成交量，在K线图中所需的成交量是当前时刻的成交量。因此，转换过程只需计算两个时刻成交量、成交额之差即可。

```python
def volume_adjust(self):
    self.useDf['turnover'] = self.useDf['turnover'].diff()
    self.useDf['volume'] = self.useDf['volume'].diff()
    print("Adjusting Volume...")
```

​	在通过`resample`方法聚合数据后，可以通过`resample.sum()`计算每个时段的成交量、成交额，并进行可视化。

#### 均价计算

​	K线图中除了蜡烛图外最重要的一个组成部分是成交价格均线。对成交金额和成交量进行计算后，通过成交金额除以成交量可以得到每个时间段平均的成交价格。此后，通过计算移动平均线可以获得`MA5, MA10, MA30`等数据。

## 可视化K线实现

#### `mplfinance`功能概览

1. 交互式显示鼠标所在点数据信息
2. 输入所需股票代码以切换数据显示
3. 点击切换K线显示频率
4. 可以显示均线数据
5. 可以显示成交量数据
6. 可以自定义指标显示（MACD, KDJ等）

![image-20220203222432265](C:\Users\Yip\AppData\Roaming\Typora\typora-user-images\image-20220203222432265.png)

#### `Echarts`功能概览

1. 可以根据需求选择股票代码显示
2. 可以点击切换K线显示频率
3. 可以自定义要显示的均线指标，点击可取消显示
4. 可以交互显示股票K线四价、均线指标
5. 可以交互显示成交量
6. 可以交互显示多种自定义指标(MACD, KDJ...)
7. 可以通过拖动调整K线图缩放比例
8. 通过`timeline`功能实现动态数据回放
9. 可导出成`html`类型文件在浏览器查看K线图或动态回放

![image-20220203171817443](C:\Users\Yip\AppData\Roaming\Typora\typora-user-images\image-20220203171817443.png)

#### 小结

​	通过对比`mplfinance`和`pyecharts`，我最终决定使用`pyecharts`作为交互式K线图呈现。相比于`mplfinance`，该图表具有以下优势：

1. 相比于静态类型的`matplotlib`，`pyecharts`有更顺滑的交互界面
2. `pyecharts`可以拖动调整显示比例，更能显示细节
3. 可以导出`html`类型界面，实现前后端分离

相比于`pyecharts`，`matplotlib`的优势在于拥有更多自定义空间，且可以通过输入类型的`TextBox`组件和按钮类型的`RadioButtons`组件对显示股票和K线频率进行调整。

## 行情回放

​	通过`pyecharts.timeline`可以对股票当日行情进行动态回放。动态回放具有以下功能：

1. 可以根据需求显示股票数据
2. 可以点击取消均线数据显示
3. 在行情回放同时可以进行K线数据交互
4. 成交量数据可交互式显示
5. 行情回放系统可以暂停
6. 通过滑动可以加快回放速度；回放速度可自定义
7. 通过鼠标滑轮可以放大细节呈现

![image-20220203230323555](C:\Users\Yip\AppData\Roaming\Typora\typora-user-images\image-20220203230323555.png)

## 数据接口

​	数据接口存放在`get_data.ipynb`文件中，可以指定股票代码、抽样频率、起始时间和结束时间从源数据中抽取部分数据。该接口所需输入类型为：

1. `security` -- 股票代码，需写全称，如`000001.SZ`
2. `unit` -- 抽样频率，是字符串类型，如`60S`代表一分钟K线数据，`180S`代表三分钟K线数据
3. `start_time` -- 起始时间，数据开始采样的时间
4. `end_time` -- 结束时间，数据停止采样的时间

## 改进方向

​	由于项目时间较紧以及经验所限，本项目在各方面仍存在一定问题和优化空间，如下表所示：

|          **任务概览**          |                           存在问题                           |                      优化方向                      |
| :----------------------------: | :----------------------------------------------------------: | :------------------------------------------------: |
|     读取原始大型`CSV`数据      | 实时读取原始数据并切割子集（获取个股对应数据）效率较低；若先切割全部子集（获取所有个股对应数据）并保存，后根据需求读取对应数据需要较大存储空间且数据初始化时间较长。 | 进一步优化数据切割子集效率，优化数据存储和读取效率 |
|        股票行情倍速回放        | 当前行情回放方式为指定回放速度，无法中途进行速度调整；行情回放功能需要读取所有数据后进行绘图，缺少实时绘图功能。 |   进一步优化行情回放的交互界面，优化数据读取方式   |
| 接入行情回放的交互式蜡烛图界面 |   缺少实时更新数据的处理方法，只能对历史数据进行可视化处理   |                  优化数据读取方式                  |

