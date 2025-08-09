# Analysis for Frequency FFT

倍频分析认为 lq 电流反馈数据 FFT 变换后的频域幅值体现了机器人关节的健康状态。就伺服上位机计算倍频对应的幅值而言，自然只需要截取匀速段后点击“频域显示”的按钮即可；而对于 python 编程实现而言，计算机不见得能像人一样一眼认出匀速段，所以这里定义匀速段为 lq 电流数据中最长的一段连续索引，使得在这段索引中 lq 电流的极差小于某个给定的 epsilon —— 你或许也会发现 epsilon 其实是不太好确定的参量，毕竟不同关节数据的噪声幅值是不同的，这也是为什么我在搜索最长匀速段之前先对数据作了两次 Savitzky-Golay 滤波，从而让 epsilon 的选取可以足够小，同时不至于与极差的选取产生冲突以免突变点被误识别进匀速段。

较为理想的效果如下所示。时域图像的正确性由解码规则保证，给定采样频率后 lq 电流数据的单边幅值谱也是确定的，然而稍稍仔细一看便会发现，如果截取匀速段的长度不同，倍频对应的幅值也会有所差异，目前初步比对得到的粗糙结论是以上位机手动截取得到的幅值结果为基准，python 计算出的误差可以超过 25%，具体见 `tests/decode/test_analysis_freq_fft.py` 测试用例。所以一个大概的感觉是，倍频幅值不会是一个特别鲁棒的特征，当然如果不关注具体数值大小或者能提取更稳定的特征则另当别论了。

![](../images_decode/lq%20current%20fft%20J1-speed%20ratio0.1.png){ width=90% style="display: block; margin: 0 auto;" }

# Analysis for dual difference

![](../images_decode/dual%20diff%20J4.png){ width=90% style="display: block; margin: 0 auto;" }

# Analysis for Overshoot and Steady Time

下图算得的超调量 0.0549 和 0.0069 均与伺服上位机局部放大后得到的结果相同，但稳定时间却非常令人疑惑：如果当指令和反馈关节角小于某个阈值时，我们认为机器人关节运动实际已经稳定，那么这个阈值该如何选取呢？毕竟不同关节的超调图像有所差异，就算将阈值取作超调量的百分之多少的话，你也很难保证这么做确实是合理的，又或者说，我们该如何保证该阈值下机器人关节确实稳定了呢？

![](../images_decode/overshoot%20J2.png){ width=90% style="display: block; margin: 0 auto;" }