### 0 频谱分析工作流

使用信号分析器

```
Ts = readmatrix("seismicstation_ts.csv")
harp = readtimetable("harp.csv","SampleRate",1/Ts(1));
pax = readtimetable("pax.csv","SampleRate",1/Ts(2));
wanc = readtimetable("wanc.csv","SampleRate",1/Ts(3));
signalAnalyzer(wanc)
```



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



### 2 对齐信号

`finddelay` 函数使用互相关性来估计信号之间的延迟。在本练习中，您将使用此函数来对齐地震信号

This code imports the signals.

```matlab
Ts = readmatrix("seismicstation_ts.csv");
harp = readtimetable("harp.csv","SampleRate",1/Ts(1));
pax = readtimetable("pax.csv","SampleRate",1/Ts(2));
wanc = readtimetable("wanc.csv","SampleRate",1/Ts(3));
```

This code cleans up the signals.

```matlab
wanc = timetable(harp.Time,resample(wanc.Signal,1,2),'VariableNames',"Signal");
harp = normalize(harp);
pax = normalize(pax);
wanc = normalize(wanc);
```

This code overlaps the HARP and PAX signals. They should line up perfectly, but there's a small delay.

```matlab
plot(harp.Time,harp.Signal)
hold on
plot(pax.Time,pax.Signal)
hold off
legend("HARP","PAX")
xlim(seconds([2800 3500]))
```

要计算信号之间的互相关性，请使用 `xcorr` 函数。

```matlab
[c,lags] = xcorr(x,y)
```

线性卷积的定义：
$$
y(n) = \sum_{k=-\infty}^\infty x(k)h(n-k)
$$
互相关的定义：
$$
y(n) = \sum_{k=-\infty}^\infty x(k)h(n+k)
$$
可以看出，线性卷积与互相关的区别在于计算之前是否存在反褶，其计算思路是一致的，都是通过滑窗的思想。

 如何通过修改时间表来对齐信号？您可以将该延迟添加到 `harp` 中的 `Time` 变量中，但也可以设置时间表的 `StartTime` 属性，这样更节省内存。 

您可以使用圆点表示法设置时间表属性：  

```matlab
tbl.Properties.PropName
harp.Properties.StartTime = harpDelay
paxDelay = finddelay(pax.Signal, wanc.Signal)
paxDelay = paxDelay*Ts(2)
paxDelay = seconds(paxDelay)
pax.Properties.StartTime = paxDelay
plot(harp.Time,harp.Signal)
hold on
plot(pax.Time,pax.Signal)
hold off
legend("HARP","PAX")
xlim(seconds([2800 3500]))
```



### 3 合并时间表

This code imports the signals.

```matlab
Ts = readmatrix("seismicstation_ts.csv");
harp = readtimetable("harp.csv","SampleRate",1/Ts(1));
pax = readtimetable("pax.csv","SampleRate",1/Ts(2));
wanc = readtimetable("wanc.csv","SampleRate",1/Ts(3));
```

This code cleans up the signals.

```matlab
wanc = timetable(harp.Time,resample(wanc.Signal,1,2),'VariableNames',"Signal");
harp = normalize(harp);
pax = normalize(pax);
wanc = normalize(wanc);
```

This code aligns the signals.

```matlab
harp.Properties.StartTime = seconds(finddelay(harp.Signal,wanc.Signal)*Ts(1));
pax.Properties.StartTime = seconds(finddelay(pax.Signal,wanc.Signal)*Ts(2));
```

现在，所有信号都具有相同的采样率并已对齐，您可以使用 `synchronize` 函数将它们合并到一个表中。

按该顺序同步 `harp`、`pax` 和 `wanc` 信号。将新表命名为 `quakes`。

```
quakes = synchronize(harp,pax,wanc)
```

新变量名称是 `Signal_harp`、`Signal_pax` 和 `Signal_wanc`。您可以通过更新表的 `VariableNames` 属性来选择更简洁的名称。 

```
tbl.Properties.VariableNames = ["Name1" "Name2"]
```

将 `quakes` 中的变量名称更新为 HARP、PAX 和 WANC。

```
quakes.Properties.VariableNames = ["HARP","PAX","WANC"]
```

您通过使用  `stackedplot` 函数可以很方便地在一个时间表中显示多个变量。

```matlab
stackedplot(tbl)
stackedplot(quakes)
```

由于 HARP 和 PAX 信号比 WANC 信号开始得晚，HARP 和 PAX 信号的开头包含 NaN。同样，HARP 和 WANC 信号的末尾也包含 NaN。

如果输入信号包含 NaN，一些函数比如 `pspectrum` 将无法执行。在这种情况下，常用的解决方案是修剪信号的开头和末尾。

前面提到，您可以使用 `timerange` 函数来提取时间表的区域。

```
tr = timerange(seconds(st),seconds(en))
tbl = tbl(tr,:)

tr = timerange(seconds(2800),seconds(3500))
quakesROI = quakes(tr,:)
stackedplot(quakesROI)
```

在接下来的课程中，您不用从 `csv` 文件中加载地震波，只需使用 `quakes` 时间表中时间范围在 2800-3500 秒的数据。

尝试在信号分析器中查看 `quakesROI` 信号：

```matlab
signalAnalyzer(quakesROI)
```



### 4 频谱分析

#### 4.1 自定义功率谱图

This code sets up the activity.

```
load quakes
pspectrum(quakes,"FrequencyLimits",[0 1])
```

要自定义功率谱图，您可以使用 `pspectrum` 函数获得功率谱和对应的频率。

```
[p,f] = pspectrum(quakes)
```

`semilogx` 函数将创建一个绘图，其中 x 轴具有对数刻度。

```
semilogx(f,p)
```

现在，您可以看到信号中的低频峰值，但仍需要修改 y 轴。

注意脚本中的第一个绘图的 y 标签是“功率谱 (dB)”。一般情况下，我们通过计算 `10*log10(p)` 以使用 `db`刻度来可视化功率谱，其中 *p*是频谱。

您可以直接使用 `10*log10(p)`，也可以使用 `db` 函数进行相同的计算。

```
db(p,"power")
semilogx(f,db(p,"power"))
xlabel("Frequency (HZ)")
ylabel("Power Spectrum (DB)")
legend("HARP","PAX","WANC")
```

 现在您可以很轻松地比较每个台站的地震信号的功率谱。三个信号都具有非常相似的低频，但只有来自 Mount Wrangell 台站的信号包含显著的高频成分。

这表示 Mount Wrangell 信号中的低频成分极有可能会与 PAX 和 HARP 信号匹配。在下一章中，您将对 Mount Wrangell 信号进行滤波，只保留其中低频成分。

在滤波之前，您将创建 Mount Wrangell 信号的时频图，以进一步研究频谱。  

![1670981399987](D:\# 工作\ENN\动设备故障诊断\Signal Process-Matlab\figures\1670981399987.png)

#### 4.2 时频分析

频谱图与尺度图：Spectrogram、Scalogram

Fourier(sines)->Spectrogram    `pspectrum` 信号很好

Wavelets(一系列小波)->Scalogram   `cwt`

Both: 将信号分割成若干个短的分块，然后计算每个分块的**频谱**。每个频率的功率用一种颜色表示，因此每像素表示该频率在当时的功率。

##### 4.2.1 创建时频图

您可以通过使用带输入 `"spectrogram"` 的 `pspectrum` 函数创建一个频谱图。

一次只能创建一个信号的频谱图，但 `quakes` 表包含三个信号。您可以使用点索引分别输入信号和时间向量。

```
pspectrum(quakes.WANC, quakes.Time,"spectrogram")
```

![1670987716791](D:\# 工作\ENN\动设备故障诊断\Signal Process-Matlab\figures\1670987716791.png)

创建频谱图时，您可以使用 `"FrequencyLimits"` 选项指定频率范围。将频率范围设置为 2 Hz 至 10 Hz。

```
pspectrum(quakes.WANC,quakes.Time,"spectrogram","FrequencyLimits",[2 10])
```

![1672015284886](D:\# 工作\ENN\动设备故障诊断\Signal Process-Matlab\figures\1672015284886.png)

现在黄色频带更加突出。然而，仍有很多背景噪声。请注意，颜色栏为从深蓝色到黄色，但频谱图上看不到深蓝色。

您可以使用 `"MinThreshold"` 选项去除低功率频率。  

频谱图的背景大部分是绿色，因此可使用绿色颜色栏值来选择阈值。

```matlab
pspectrum(quakes.WANC, quakes.Time, "spectrogram","FrequencyLimits", [2 10], "MinThreshold",-50)
```

![1672015536847](D:\# 工作\ENN\动设备故障诊断\Signal Process-Matlab\figures\1672015536847.png)

现在频谱图包含更多详细信息，但频带周围的区域仍含有不少噪声。

在创建频谱图时，您可以设置更多选项，也可以尝试不同的时频可视化方法。
您可以用 `cwt` 函数创建尺度图。

```
cwt(sig,fs)
```

第一个输入是信号，第二个输入是采样率。

您还可以使用 `cwt` 函数设置频率范围。

```
cwt(sig,fs, ...
    "FrequencyLimits",[a b])
```

**任务**

创建 Mount Wrangell 信号的尺度图。前面提到，采样率是 `1/0.02`。

将频率范围设置为 2 Hz 到 10 Hz。

```matlab
cwt(quakes.WANC, 1/0.02,"FrequencyLimits", [2 10])
```

![1672016742386](D:\# 工作\ENN\动设备故障诊断\Signal Process-Matlab\figures\1672016742386.png)

`cwt` 函数没有 `MinThreshold` 选项，但您可以通过设置颜色图范围来实现相同的效果。

```
caxis([a b])
```

请注意，尺度图颜色栏显示的是幅度，而不是功率。您将使用蓝色范围内的一个限值，因为其他颜色在图上不明显。

**任务**

将颜色图范围设置为从 `0` 到 `2`。

```
caxis([0 2])
```

![1672016939218](D:\# 工作\ENN\动设备故障诊断\Signal Process-Matlab\figures\1672016939218.png)



### 5 滤波

#### 5.1 低通滤波器

低频和高频对应于两个不同的地震事件。您将从印度尼西亚地震中提取低频面波。然后提取它在阿拉斯加引起的地震活动。

![1672017440670](D:\# 工作\ENN\动设备故障诊断\Signal Process-Matlab\figures\1672017440670.png)

**背景**

HARP 和 PAX 地震台站记录了苏门答腊地震的低频面波。

虽然从时域上看不明显，但已有有力证据表明 Mount Wrangell 地震台站也记录了相同的面波：

1. 信号之间的互相关性
2. 功率谱显示在 0.05 Hz 附近出现相似的峰值


在本练习中，您将对 Mount Wrangell 信号进行滤波，以便在时域中查看低频。

在给定频率上添加一条垂直线，有助于选择用于滤波的通带频率。

```matlab
xline(pass)
```

This code plots the power spectrum for the earthquake signals.

```matlab
load quakes
[p,f] = pspectrum(quakes);
semilogx(f,db(p,"power"))
legend("HARP","PAX","WANC")
xlabel("Frequency (Hz)")
ylabel("Power Spectrum (dB)")
```

**任务**

在功率谱图上 `0.1` Hz 处添加一条 x 线。

```
xline(0.1)
```

![1672020039856](D:\# 工作\ENN\动设备故障诊断\Signal Process-Matlab\figures\1672020039856.png)

当您对 Mount Wrangell 信号在 `0.1` Hz 处进行低通滤波时，主要保留的是垂直线左侧的频率。

可以使用 `lowpass` 函数进行低通滤波。

```matlab
lowpass(tbl,pass)
```

您只需对存储在 `quakes` 的 `WANC` 变量中的 Mount Wrangell 信号进行滤波。要对时间表中的一个变量进行滤波，您可以使用冒号运算符 (`:`) 来获取给定变量名称的所有时间戳。

```matlab
tbl(:,"VarName")
```

**任务**

在 `0.1` Hz 处对 `quakes` 中的 `WANC` 变量进行低通滤波。 

```
tbl = quakes(:,"WANC")
lowpass(tbl, 0.1)
```

![1672020954002](D:\# 工作\ENN\动设备故障诊断\Signal Process-Matlab\figures\1672020954002.png)

该图显示 Mount Wrangell 信号的时域和频域。

请注意，虽然滤波去除了一些高频，但仍有超过 `0.1` 的频率。

接下来，我们将滤波后的信号保存到一个变量中，以便将该信号与 HARP 和 PAX 信号进行比较。  

**任务**

重复任务 2 中的命令，但这次请使用一个名为 `lowWANC` 的输出变量。

```
tbl = quakes(:,"WANC")
lowWANC = lowpass(tbl, 0.1)
```

`lowWANC` 是具有一个名为 `WANC` 的变量的时间表。由于 `lowWANC` 的时间向量仍与 `quakes` 的相同，您可以在 `quakes` 时间表中添加 `lowWANC` 作为新变量。

要向表中添加新变量，可以使用圆点表示法。

```matlab
tbl.NewVar = data
```

**任务**

将 `lowWANC.WANC` 添加到 `quakes` 表中。将新变量命名为 `FiltWANC`。

```
quakes.FiltWANC = lowWANC.WANC
```

**任务**

使用 `figure` 命令创建一个新图窗。然后创建一个包含 `quakes` 中所有信号的堆叠图。

```
figure
stackedplot(quakes)
```

![1672021666210](D:\# 工作\ENN\动设备故障诊断\Signal Process-Matlab\figures\1672021666210.png)

图 `FiltWANC` 看起来与 HARP 和 PAX 相似，但有许多锯齿状曲线。**锯齿状曲线包含滤波后残留的高频成分**。

要减少滤波信号中的高频成分，可以增大[陡度](https://www.mathworks.com/help/signal/ref/lowpass.html#mw_3aa14271-5d7b-4a2a-8b29-b1f47d3ce414)。 

```
sig = lowpass(tbl,pass,"Steepness",s)
```

**任务**

更新任务 3 的 `lowpass` 代码。将 `"Steepness"` 选项设置为 0.95。

```
tbl = quakes(:,"WANC")
lowWANC = lowpass(tbl, 0.1, "steepness",0.95)
```

![1672021942925](D:\# 工作\ENN\动设备故障诊断\Signal Process-Matlab\figures\1672021942925.png)

#### 5.2 带通滤波器

在上一个练习中，您提取了低频，这些低频对应于苏门答腊地震的面波。

现在您需要得到高频成分，它对应于 Mount Wrangell 附近的本地地震。这些本地地震的频率范围是 2 Hz 到 10 Hz。

要提取这些频率，可以使用带通滤波器只保留该范围内的频率。

```matlab
bandpass(tbl,[f1 f2])
```

This code sets up the activity.

```matlab
load quakes
[p,f] = pspectrum(quakes);
figure
semilogx(f,db(p,"power"))
legend("HARP","PAX","WANC","Lowpass WANC","Location","best")
xlabel("Frequency (Hz)")
ylabel("Power Spectrum (dB)")
```

![1672022277796](D:\# 工作\ENN\动设备故障诊断\Signal Process-Matlab\figures\1672022277796.png)

**任务**

对 `quakes` 中的 `WANC` 变量执行从 `2` 到 `10` Hz 的带通滤波。

```matlab
tbl = quakes(:, 'WANC')
bandpass(tbl,[2 10])
```

![1672022867454](D:\# 工作\ENN\动设备故障诊断\Signal Process-Matlab\figures\1672022867454.png)

**任务**

重复任务 1 中的命令，但这次请使用一个名为 `bandWANC` 的输出变量。

```
tbl = quakes(:, "WANC")
bandWANC = bandpass(tbl, [2 10])
```

在绘制低通和带通滤波信号之前，请将它们合并到一个时间表中。

```matlab
tbl = timetable(t,sig1,sig2,...
    'VariableNames',["A" "B"])
```

**任务**

创建一个名为 `compfilt` 的新时间表。使用以下输入：

1. `quakes` 或 `bandWANC` 中的时间向量
2. `bandWANC.WANC`
3. `quakes.FiltWANC`
4. 将变量名称设置为 `"Bandpass"` 和 `"Lowpass"`

```
compfilt = timetable(quakes.Time, bandWANC.WANC, quakes.FiltWANC, 'VariableNames', ["Bandpass" "Lowpass"])
```

![1672023535403](D:\# 工作\ENN\动设备故障诊断\Signal Process-Matlab\figures\1672023535403.png)

**任务**

通过输入 `figure` 创建一个新图窗。然后创建一个堆叠图 `compfilt`。

将 x 范围设置为从 `2900` 秒到 `2950` 秒，以便放大两次本地地震。

```
figure
stackedplot(compfilt)
xlim(seconds([2900 2950]))
```

![1672023871604](D:\# 工作\ENN\动设备故障诊断\Signal Process-Matlab\figures\1672023871604.png)

地球科学家认为苏门答腊的地震引发了阿拉斯加的地震。通过滤波，他们能够比较来自 Mount Wrangell 地震仪的低频和高频信号。这为他们确认两次遥远地震之间关系提供了的宝贵信息。

如果您滚动绘图，可以看到每个高频脉冲都出现在低频波的峰值附近。

您还可以在信号分析器中比较信号：

```matlab
signalAnalyzer(compfilt)
```

尝试不使用 `lowpass` 或 `bandpass` 函数，而是在信号分析器中对 Mount Wrangell 信号进行滤波。

#### 5.3 信号测量

您在上一个练习中已看到，苏门答腊的地震非常强烈，波及了遥远的阿拉斯加，引发了当地小等级的地震。要在数据中识别出单个地震，您需要从信号中提取一些信息。

从信号中提取测量值在机器学习应用中很常见，在这种应用中，您需要根据数据计算*特征*，即提取的信息。请查看[机器学习入门之旅](https://matlabacademy.mathworks.com/selfpaced/machinelearning?s_tid=course_dlor_bodych1)的[特征工程](https://matlabacademy.mathworks.com/selfpaced/machinelearning?s_tid=course_dlor_bodych1#chapter=4&lesson=1&section=1)一章，了解更多详细信息。  

**背景**

Mount Wrangell 信号的频谱图已显示。地震发生在频带为 2 Hz 到 10 Hz 的区域。您是否能根据此频谱图找到这些地震的时间戳？

在基于频谱图进行任何计算之前，您首先需要获得数组（而不仅仅是图像）。

```matlab
[p,f,t] = pspectrum(sig,time,"spectrogram")
```

三个输出是每段信号的**频谱估计值、频率和时刻**。

![1672024356352](D:\# 工作\ENN\动设备故障诊断\Signal Process-Matlab\figures\1672024356352.png)

This code loads the earthquake data.

```matlab
load quakes
load compfilt
```

This code creates the spectrogram from the Spectral Analysis chapter.

```matlab
pspectrum(quakes.WANC,quakes.Time,"spectrogram","FrequencyLimits",[2 10],"MinThreshold",-50);
```

**任务**

复制用于创建频谱图的代码。添加三个名为 `p`、`f` 和 `t` 的输出参数。

```matlab
[p,f,t] = pspectrum(quakes.WANC,quakes.Time,"spectrogram","FrequencyLimits",[2 10],"MinThreshold",-50);
```

要找出哪些时间戳包含大量高频成分，您可以**对每个时间戳的所有频率的功率求和**。

![1672024642779](D:\# 工作\ENN\动设备故障诊断\Signal Process-Matlab\figures\1672024642779.png)

**任务**

计算 `p` 的 `sum`。将输出命名为 `psum`。

然后创建 `psum` 对 `t` 的图。

```matlab
psum = sum(p)
plot(t, psum)
```

![1672024990150](D:\# 工作\ENN\动设备故障诊断\Signal Process-Matlab\figures\1672024990150.png)

您在 `psum` 中可以看到几个峰值，但有些峰值远不如其他峰值突出。为了突显峰值，您可以计算功率。

```matlab
db(p,"power")
```

**任务**

计算出 `psum` 的功率并将其命名为 `pwr`。

然后创建 `pwr` 对 `t` 的图。

```matlab
pwr = db(psum, "power")
plot(t, pwr)
```

![1672025168676](D:\# 工作\ENN\动设备故障诊断\Signal Process-Matlab\figures\1672025168676.png)

**背景**

要找到每次地震的时间戳，您可以查找这些尖峰的位置。

您可以使用“求局部极值”的交互式任务来自动识别信号中的局部最小值或最大值。

**任务**

1. 将“求局部极值”实时任务添加到任务 4 和 5 的代码节中
2. 选择 `pwr` 作为输入数据
3. 将输出变量命名为 `findquakes`
4. 选择 `t` 作为 x 轴数据

**在“实时编辑器”选项中，点击“任务”下拉菜单，找到“数据预处理”工作区，选择“Find Local Extreme”子任务。**

![1672031838743](D:\# 工作\ENN\动设备故障诊断\Signal Process-Matlab\figures\1672031838743.png)

在"input data"与"X-axis"设置如下图所示，并将输出变量命名为 `findquakes` 。

![1672031739168](D:\# 工作\ENN\动设备故障诊断\Signal Process-Matlab\figures\1672031739168.png)

使用默认选项找到太多的局部极值。您可以通过修改选项筛查出地震的局部极值。

如果您已知要查找的地震次数，可以输入该值作为极值的最大数量。在大多数情况下，起初极值的数量是未知的，因此，您可以转而调整相对高差和间隔选项。

**峰值的相对高差是其高度和位置相对于其他峰值的度量。**

**任务**

将最小相对高差选项增大到 `10`。

将“Min prominence” 设置为10即可。

![1672032188846](D:\# 工作\ENN\动设备故障诊断\Signal Process-Matlab\figures\1672032188846.png)

![1672032200286](D:\# 工作\ENN\动设备故障诊断\Signal Process-Matlab\figures\1672032200286.png)

```
plot(compfilt.Time,compfilt.Bandpass)
xline(t(findquakes))
title("Bandpass Signal")
```

![1672032445123](D:\# 工作\ENN\动设备故障诊断\Signal Process-Matlab\figures\1672032445123.png)

#### 5.4 函数参考

##### 5.4.1 使用时间表

|                             函数                             |             说明              |                           示例                            |
| :----------------------------------------------------------: | :---------------------------: | :-------------------------------------------------------: |
| [`readtimetable`](https://www.mathworks.com/help/matlab/ref/readtimetable.html) |      基于文件创建时间表       |     `sig = readtimetable("sig.csv","SampleRate",fs)`      |
| [`sig.VariableName`](https://www.mathworks.com/help/matlab/matlab_prog/access-data-in-a-table.html) |          圆点表示法           |                `plot(sig.Time,sig.Signal)`                |
| [`seconds`](https://www.mathworks.com/help/matlab/ref/duration.seconds.html) |      以秒为单位指定时间       |                 `tstart = seconds(2000)`                  |
| [`timerange`](https://www.mathworks.com/help/matlab/ref/timerange.html) |  选择某时间范围内的时间表行   | `lim = timerange(seconds(5),seconds(10))sig = sig(lim,:)` |
| [`synchronize`](https://www.mathworks.com/help/matlab/ref/timetable.synchronize.html) |  基于公共时间向量同步时间表   |          `sigcombined = synchronize(sig1,sig2)`           |
| [`stackedplot`](https://www.mathworks.com/help/matlab/ref/stackedplot.html) | 基于公共 x 轴绘制几个变量的图 |                    `stackedplot(sig)`                     |

##### 5.4.2  预处理信号

|                             函数                             |                    说明                     |                             示例                             |
| :----------------------------------------------------------: | :-----------------------------------------: | :----------------------------------------------------------: |
| [`resample`](https://www.mathworks.com/help/signal/ref/resample.html) | 以原始采样率的 `p/q` 倍对输入序列进行重采样 |                `sig = resample(sig,*p*,*q*)`                 |
| [`normalize`](https://www.mathworks.com/help/matlab/ref/double.normalize.html) |          使用 Z 分数方法归一化信号          |                    `sig = normalize(sig)`                    |
| [`finddelay`](https://www.mathworks.com/help/signal/ref/finddelay.html) |             估计信号之间的延迟              | `delay = finddelay(sig1.Signal,sig2.Signal)sig1.Properties.StartTime = seconds(delay/fs)` |

##### 5.4.3 频谱分析

|                             函数                             |                  说明                   |                       示例                        |
| :----------------------------------------------------------: | :-------------------------------------: | :-----------------------------------------------: |
| [`pspectrum`](https://www.mathworks.com/help/signal/ref/pspectrum.html) |               显示功率谱                |                 `pspectrum(sig)`                  |
| [`semilogx`](https://www.mathworks.com/help/matlab/ref/semilogx.html) |         在频率轴上使用对数刻度          | `[p,f] = pspectrum(sig)semilogx(f,db(p,"power"))` |
| [`pspectrum`](https://www.mathworks.com/help/signal/ref/pspectrum.html#mw_c525d878-6c0e-4ceb-a23c-9fe9922f4278) |               显示频谱图                |          `pspectrum(sig,"spectrogram")`           |
| [`cwt`](https://www.mathworks.com/help/wavelet/ref/cwt.html) | 显示连续小波变换 *需要 Wavelet Toolbox* |               `cwt(sig.Signal,fs)`                |

##### 5.4.4 滤波

|                             函数                             |          说明          |                示例                 |
| :----------------------------------------------------------: | :--------------------: | :---------------------------------: |
| [`lowpass`](https://www.mathworks.com/help/signal/ref/lowpass.html) |   对信号进行低通滤波   |          `lowpass(sig,10)`          |
| [`bandpass`](https://www.mathworks.com/help/signal/ref/bandpass.html) |   对信号进行带通滤波   |       `bandpass(sig,[8 12])`        |
| [`lowpass`](https://www.mathworks.com/help/signal/ref/lowpass.html?s_tid=doc_ta#mw_3aa14271-5d7b-4a2a-8b29-b1f47d3ce414) | 使用修改的陡度进行滤波 | `lowpass(sig,10,"Steepness",0.95);` |

##### 5.4.5 信号测量

|                             函数                             |                说明                |
| :----------------------------------------------------------: | :--------------------------------: |
| [**求局部极值**任务](https://www.mathworks.com/help/matlab/ref/findlocalextrema.html) | 在实时编辑器中求局部最大值和最小值 |

##### 5.4.6 比较可视化图

| 时域 ![img](https://matlabacademy-content.mathworks.com/4.45/R2022b/cn/content/Signal%20Processing/Onramp/Summary/images/time.png) | 频域 ![img](https://matlabacademy-content.mathworks.com/4.45/R2022b/cn/content/Signal%20Processing/Onramp/Summary/images/spectrum.png) |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| 时频域 ![img](https://matlabacademy-content.mathworks.com/4.45/R2022b/cn/content/Signal%20Processing/Onramp/Summary/images/spectro.png) |                                                              |

如需了解更多内容，请参加 [MATLAB 信号处理](https://matlabacademy.mathworks.com/details/signal-processing-with-matlab/mlsg)，深入了解下列方法：

- 生成信号和常见信号操作
- 估计功率谱密度
- 数字滤波器的表征与设计
- 流式信号处理

#### 5.5 相关主题

|      | [Simulink 信号处理快速入门](https://www.mathworks.com/videos/getting-started-with-simulink-for-signal-processing-1586429627003.html) - 使用 Simulink 执行频谱分析 |      | [使用深度学习进行信号处理](https://www.mathworks.com/solutions/deep-learning/deep-learning-signal-processing.html) - 信号处理专用功能概述 |
| ---- | ------------------------------------------------------------ | ---- | ------------------------------------------------------------ |
|      | [DSP System Toolbox 快速入门](https://www.mathworks.com/help/dsp/getting-started-with-dsp-system-toolbox.html) - 流式信号处理系统的设计和仿真 |      | [示例](https://www.mathworks.com/help/signal/examples.html) - 各种信号处理应用的示例代码 |

####  5.6 训练

|      | [自定进度的在线课程](https://matlabacademy.mathworks.com/) - 以交互方式按照您自己的进度学习 MATLAB |      | [教师授课课程](https://www.mathworks.com/services/training/courses.html) - 参加由经验丰富的 MATLAB 讲师授课的现场课程。 |
| ---- | ------------------------------------------------------------ | ---- | ------------------------------------------------------------ |
|      |                                                              |      |                                                              |

#### 5.7 参考资料

|      | [文档](https://www.mathworks.com/help/index.html) - 所有 MATLAB 函数的详尽解释及其用法示例 |      | [MATLAB 基础函数参考](https://www.mathworks.com/content/dam/mathworks/fact-sheet/matlab-basic-functions-reference.pdf) - 常用 MATLAB 函数速查表 |
| ---- | ------------------------------------------------------------ | ---- | ------------------------------------------------------------ |
|      |                                                              |      |                                                              |

 