# DVB-S2 Transceiver

## TX System
![](https://i.imgur.com/cy19ZSi.png)

### DVB-S2 TX System MATLAB
```Matlab=
% Parameter setting
frame_N = 10;             % number of frame
% data = (randi([0 1], 1, 3072*frame_N)); % CR:1/4
data = (randi([0 1], 1, 11712*frame_N));
%CR:3/4; modcod-qpsk:7, 8psk:14, 16apsk:19, 32apsk:24
DVB_S2_GP.MODCOD = 7;     % qpsk:1~11, 12~17:8psk, 18~23:16APSK, 24~28:32apsk
DVB_S2_GP.LS_mode = 1;    % 0/1: Normal/Short frame
DVB_S2_GP.Pilot_mode = 0; % pilot mode: off
DVB_S2_GP.OVSR = 4;       % oversampling ratio
[DVB_S2_GP.APSK_mode, DVB_S2_GP.CR_mode] = MODCOD_LUT(DVB_S2_GP.MODCOD);
DVB_S2_GP = DVB_S2_Parameters(DVB_S2_GP);

% DVB-S2 TX System %
DVB_S2_GP = DVB_S2_TX_Gen(DVB_S2_GP, frame_N, data);
```

```
DVB_S2_GP =
struct with fields:

        MODCOD: 7
       LS_mode: 1
    Pilot_mode: 0
          OVSR: 4
    PulseShape: [1×33 double]
           s_B: [1×165600 double]
```

## Channel Module
```matlab=
SNR_dB = 20;           % Additive White Gaussian Noise
CFO = 0.001;    % carrier frequency offset
CPO = rand(1);  % carrier phase offset
DVB_S2_GP = Channel_noise(DVB_S2_GP, SNR_dB, CFO, CPO);
```

## RX Inner Receiver
通訊系統接收到訊號需要將其進行同步和解調得到傳輸的數據，接收端的內部接收器(Inner-Receiver)基頻訊號處理系統架構如下圖。

![](https://i.imgur.com/vTFLR2E.png)


### 匹配濾波器
首先，因應發射端的脈波整形濾波器(Pulse shape filter, PS)，接收訊號透過匹配濾波器(Matched filter, MF)抑制 AWGN 雜訊。
傳送端PS使用squared-root raisedcosine(SRRC)濾波器，接收端則利用同樣的SRRC脈波對其進行摺積(convolution)，將脈波修正為 raised-cosine(RC)波形，此方法在時域上具有最小化符碼間干擾(intersymbol-interference, ISI)的特性，並且在頻域上可以限制訊號頻寬。


#### Matched filter MATLAB
```matlab=
% Matched Filter
DVB_S2_GP.MatchedFilter = DVB_S2_GP.PulseShape;
DVB_S2_GP = DVB_S2_MatchedFilter(DVB_S2_GP);
```



### 符碼時間同步
使用Gardner 提出的演算法，利用接收到的樣本估計時間相位偏差。
原理是由於理想符碼時間相位是在眼圖開最大位置，即其功率最大位置，從計算連續時間符碼區間內的平均功率$J(\tau)$作為觀察，將其微分後即為零點，即是理想符碼時間相位，用來判斷當下的時間相位相對於理想時間相位是提前(Early)或是落後(Late)。
而對$J(\tau)$做微分之公式，實際上簡化後，即為重新取樣的觀測值和它的微分值取共軛複數，做相乘後取實部。

\begin{aligned}
J(\tau)=\frac{1}{N}*\sum_{k=0}^{N-1}\mid r_M(k\cdot T_{sym}+\tau)\mid^2
\end{aligned}
\begin{aligned}
e_D[k]=\frac{dJ(\tau)}{d\tau}=Re\{r_M(k\cdot T_{sym}+\tau)\cdot (\frac{dr_M(k\cdot T_{sym}+\tau)}{d\Delta\tau})^*\}
\end{aligned}
![](https://i.imgur.com/JBaKnjS.png)

#### Symbol Timing Synchronizer MATLAB
```matlab=
% Symbol Timing Synchronizer
DVB_S2_GP = DVB_S2_Blind_Symbol_Sync(DVB_S2_GP);
figure(); plot(DVB_S2_GP.r_sym(7000:end-1000), '+')
axis([-1000, 1000, -1000, 1000])
```

## Result
#### Scenario: SNR=20 dB

![](https://i.imgur.com/3Mzhs6T.png)

#### Scenario: SNR=20 dB and fs/1000

![](https://i.imgur.com/hReXDiP.png)



#### QPSK之理論位元錯誤率和演算法結果錯誤率比較

![](https://i.imgur.com/AQeVQ2h.png)