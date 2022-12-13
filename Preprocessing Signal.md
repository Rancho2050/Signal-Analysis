### 1 信号预处理

```matlab
Ts = readmatrix("seismicstation_ts.csv")
harp = readtimetable("harp.csv","SampleRate",1/Ts(1));
pax = readtimetable("pax.csv","SampleRate",1/Ts(2));
wanc = readtimetable("wanc.csv","SampleRate",1/Ts(3));
```

  `resample` 函数可以将非均匀或均匀数据重采样为新的固定采样率。对于非均匀采样信号，您需要向 `resample` 函数提供信号和时间向量。 

地震信号均为均匀采样信号，因此在本练习中，您只需提供信号和重采样因子 `p` 和 `q`。 

```matlab
y = resample(x,p,q)
```

`p` 和 `q` 是整数重采样因子。输出信号 `y` 具有 `x` 的 `p/q` 倍采样。

例如，

```matlab
y = resample(x,2,3)
```

以原始采样率的 `2/3` 倍速率对 `x`进行重采样。

查看脚本底部绘图上的 y 轴范围。HARP 和 PAX 信号比 Mount Wrangell 信号大得多。信号振幅与本应用无关，因此如果信号都经过归一化，比较起来会更容易。 

 `normalize` 函数可用于对时间表进行归一化。

```matlab
tbl = normalize(tbl)
```

默认情况下，`normalize` 函数使用 [Z 分数](https://www.mathworks.com/help/matlab/ref/double.normalize.html#mw_e6886c44-2923-4074-844a-f3e1a447359d)方法。

z 值以标准差为单位测量数据点与均值的距离。标准化后的数据集均值为 0，标准差为 1，并保留原始数据集的形状属性（相同的偏斜度和峰度）。

```matlab
wanc = resample(wanc,1,2)
harp = normalize(harp)
pax = normalize(pax)
wanc = normalize(wanc)
tiledlayout(3,1)
nexttile
plot(harp.Time,harp.Signal)
title("HARP")
nexttile
plot(pax.Time,pax.Signal)
title("PAX")
nexttile
plot(wanc.Time,wanc.Signal)
title("WANC")
```



