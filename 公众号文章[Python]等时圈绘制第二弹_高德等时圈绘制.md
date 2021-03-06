## 1. 唠嗑

最近需要参见面试 然后自己大论文的pyflink代码又还没有弄好~ 所以就断更了几天咯哈哈

抽空补上之前欠下来的等时圈绘制第二弹，基于高德的等时圈绘制。本次将从高德获取行程时间数据，然后在ArcGISPro中基于该数据绘制等时圈。相关原理不再赘述，可以先看看之前的第一弹

代码下载地址仍为之前mapbox等时圈绘制的项目地址：https://github.com/Bardbo/Plot_Isochrone

## 2. 快速使用

在代码目录进入命令行，输入：

```
python .\Get_Isochrone_from_gaode.py
```

即可获得一个行程时间的csv文件：time.csv

**如何将其转化为一个等时圈呢？**下面提供两种不同的方式，ArcGISPro和Python。

### 2.1 ArcGISPro绘制

1. 打开ArcGISPro，新建地图 (没有安装的同学可以在mapbox那篇推文底部获取安装包，推文传送门： https://mp.weixin.qq.com/s/GPj_hHhvl34_SvsntfitcA)
2. 在目录中**添加文件夹连接**（右下角目录——文件夹右键——添加文件夹连接——连接到time.csv所在文件夹
3. 更换底图（在界面左上方点击地图——底图——天地图-矢量(球面墨卡托)）
4. 将连接文件夹内的csv拖入ArcGISPro中间加载
5. 在界面左侧内容栏中右键刚加载入的time.csv——**显示XY数据**——在界面右侧出现的地理处理栏中设置X字段为lng，Y字段为lat，Z字段为time——运行
6. 地理处理栏上方点击后退箭头——搜索栏中搜索 **插值**——出现很多结果，选择**径向基函数(RBF)插值法**

7. 设置输入要素为time_XYTableToPoint（点层），设置Z字段为time，输出地统计图层空着，输出栅格随便输个名字（如sg），——运行（这一步就得到了栅格的等时圈了，如下图）

![栅格等时圈](https://i.loli.net/2020/10/24/78ZrdAWcQSf6RP5.png)

7. 地理处理栏上方点击后退箭头——搜索 **等值线** ——点击第一个搜索结果——输入栅格设置为sg（上一步生成的结果栅格）——等值线间距设置为600（**十分钟**）——运行即可得到等值线了（如果设置等值线类型为**等值线面**，就可以得到等值线面，生成后可以自己换颜色，下一步演示）
8. 设置等值线类型为**等值线面**——运行——在左侧内容栏中右键生成的等值线面层——符号系统——在右侧符号系统栏中，设置主符号系统为**分级色彩**，色带可以选择大家喜欢的黄色或绿色之类的（滑稽 你不对劲），按各自的审美来弄即可，如图一个奇奇怪怪的矢量等时圈就诞生了

![矢量等时圈](https://i.loli.net/2020/10/24/ejCu5OSmcrEYVBz.png)

### 2.2 当然啦 我们也可以使用Python进行绘制啦

绘制的代码为plot_isochrone_in_python.ipynb文件，需要在jupyter中打开（当然，ArcGISPro也是支持jupyter的）

主要是用到scipy中的插值函数，和matplotlib的绘图功能，绘图的代码比较简单，就不进行详细解释了，自己看看就行，结果如图：

![python_isochrone](https://i.loli.net/2020/10/25/b3ftKSjQ1a4zIhE.png)

## 3. 源码解释

前面快速使用中获取的是长沙一个点的公交出行（含地铁）等时圈，默认将该中心点的周边矩形范围划分为5行5列，也就是36个交点。这些我们都可以自己更改，但是本次代码仅写了三种出行方式，高德地图支持的另外几种方式没有写（但是原理是一样的）。

使用默认配置，我们至少需要更改两个参数，一个是**method参数**，表示不同的出行方式，可以是walking、driving、transit，其中transit表示公交出行；另一个参数是**等时圈的起点坐标center_coor**，需要是高德坐标系，因此建议使用高德坐标拾取器拾取（传送门：https://lbs.amap.com/console/show/picker）

源代码总共由两部分组成：generateCoor函数，这是根据左下角、右上角坐标返回矩形范围交叉点坐标的函数，可以不用管他；**GetTripTime类**，这个是核心，得介绍一下。

1. 初始化参数

method，center_coor前面介绍过了，然后**r参数**，表示矩形范围1/2边长，这个取得越大矩形范围就越大，等时圈的范围就越大，单位是度数，即经纬度；**参数m和n**，分别表示矩形的划分行数和列数，最后会获取（m+1）×（n+1）个点坐标的行程时间，这里取得越大则等时圈数据越精细，边界越离散，同时也越耗时；**key参数**，这个就是高德开放平台的webAPIkey了，还是**希望各位大佬自己注册申请哦**

2. 接下来会生成交点的坐标
3. 然后就是使用request来请求接口，获得数据了，三个不同的函数分别对应三种不同的出行方式，值得一提的是，公交出行的参数有一些不同的地方，大家可以对照高德的官方文档进行设置，那里有详细的解释，传送门：https://lbs.amap.com/api/webservice/guide/api/direction，此外驾车出行的参数也有一些别的，代码中未写出来，大家可以自己根据自己的需要进行更改
4. 然后写了一个main函数，作用就是循环遍历各个交点，获得中心点到达各个交点的时间，然后使用pandas将其存储为csv格式。

本次的分享就到这里啦，实际上其余接口的调用也是同样的道理，比如基于高德地图获取POI数据，获取公交线路站点数据等等，网上有相当多的教程，这类接口的调用还是挺简单的，大家有时间可以对代码进行优化哦，比如加入多线程之类的

到时候有时间我在GitHub上补一下POI数据、公交线路站点数据等获取的代码，但是可能就不会写文章来进行详细介绍了哈~有什么问题可以公众号留言给我