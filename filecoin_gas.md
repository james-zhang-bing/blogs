[TOC]

------------------------

# gas分析整理

## 名词解释

- `feeCap` :使用一个`gas`所需金额的上限，打个比方，就是每一升油价格的上限，也就是说，消息发送者所能接受的油价上限。
- `baseFee` :使用一个`gas`所需金额。也就是“汽油”的市场价。
- `gasPremium`:每个`gas`的支付给矿工的金额，`gas`的数量为`gasLimit`,`gasPremium`会受到`feeCap`和`baseFee`的影响
- `gasLimit`: 所使用`gas`的个数上限

## 最大消耗费用

也就是执行消息所要消耗的金额的上限，不管是燃烧掉的、还是给矿工的小费，其总和都不会大于此。

$$
\begin{aligned}
执行消息所需金额上限&=gasLimit \cdot feeCap    \\
\end{aligned}
$$

## 基础费用消耗

`baseFee`即，基本gas价格，市场gas价格/升

$$
\begin{aligned}
基本费用消耗&=基本gas价格 \cdot gas使用量    \\
\end{aligned}
$$

- 如果`baseFee`大于`feeCap`(消息设置的gas价格上限，即使用一个gas所需消耗的金额不得超过此数字)，则以`feeCap`作为`baseFee`(基础费用,即消耗一个gas所需的金额)
- `baseFee`若超过`feeCap`,超出部分由矿工支付(惩罚矿工)。

$$
基本gas燃烧额度=
\begin{cases}
    &baseFee \cdot gasUsed &,\text{(当$ baseFee< feeCap $ 时)}\\
    &feeCap \cdot gasUsed &,\text{(当$baseFee>=feeCap$时，少燃烧部分由矿工承担)}
\end{cases}\\
超出部分由矿工支付=(basefee-feecap)\cdot gasUsed\\
$$

------

## 矿工小费

`gasPremium`为每用一个gas给矿工的小费，即矿工小费单价

$$
矿工小费=
\begin{cases}
    &矿工小费单价 \cdot gasLimit,\text{(当 baseFee+gasPremium<=feeCap 时)}\\
    &(消息费用上限feeCap-基础消耗费用baseFee)\cdot gasLimit,\text{(当 基础gas单价+矿工小费单价>feeCap 时)}
\end{cases}
$$

## 设置的gasLimit过高，导致气体烧掉的gas

$$
OverEstimation=gasLimit超出预估大小导致烧掉的总gas=
\begin{cases}
& gasLimit&,&\text{当 gasUsed为0时,}\\
& 0&,&\text{当 $gasLimit< 1.1 \cdot gasUsed $ 时,}\\
& 剩余气体\times 超出率&，&\text{当 $gasLimit>= 1.1 \cdot gasUsed $ 时,}
\end{cases}\\

\begin{aligned}
剩余气体&=gasLimit-gasUsed,\\
超出率&={gasLimit \over gasUsed}-1.1， \text{（超出率最大取1）}\\
燃烧单个gas的价格&=baseFee<=feeCap
\end{aligned}
$$

### 矿工最终受到的惩罚

$$
惩罚金额=3\times(baseFee-feeCap)\times (gasUsed+OverEstimation)
$$

## 返还的气体

$$
返还的gas=gasLimit-烧掉的gas
$$

## gasOutPut中各费用去向

```go
type GasOutputs struct {
    BaseFeeBurn        abi.TokenAmount // to burn actor   --> baseFeeToPay * gasUsed
    OverEstimationBurn abi.TokenAmount // to burn actor -->烧毁多余气体的费用 --> out.OverEstimationBurn=baseFeeToPay * GasBurned

    MinerPenalty abi.TokenAmount    // to reward actor -->to burn actor 应烧毁的费用不够，就惩罚矿工剩余的烧毁费用
    MinerTip     abi.TokenAmount //to reward actor
    Refund       abi.TokenAmount //to msg.from -->剩余费用返还给发送者 --> 剩余费用=总费用-烧毁费用-多余气体烧毁费用-矿工小费

    GasRefund int64 
    GasBurned int64
}
```

----------------------------------------------------------------

# 系统自动估计相关gas参数

## 系统估计feeCap

- 第一种情况
  
$$
  \begin{aligned}
feeCap&={parentBaseFee \cdot (1+{1\over 8})^{maxqueueblks} \cdot 2^8 \over 2^8 }\\
&=parentBaseFee\cdot({9\over 8})^{maxqueueblks}\\
\end{aligned}\\
$$
  
  目前，`maxqueueblks`=20  
  故 ${feeCap=parentBaseFee\cdot({9\over 8})^{20}\approx 10.54 \cdot BF_{n-1}}$

然后，`msg.feeCap`=`feeCap`+`GasPremium` ；  也就是将矿工的小费也包含到其中去。

----------------------------------------------------------------

- 最后，**当** `feeCap`*`GasLimit`>`MaxFee` **时**：$
  \begin{aligned}
  feeCap&={MaxFee \over GasLimit}
  \end{aligned}
  $
  
  `MaxFee`的默认值如下

``` go
Fees: MinerFeeConfig{
            MaxPreCommitGasFee:     types.MustParseFIL("0.025"),
            MaxCommitGasFee:        types.MustParseFIL("0.05"),
            MaxWindowPoStGasFee:    types.MustParseFIL("5"),
            MaxPublishDealsFee:     types.MustParseFIL("0.05"),
            MaxMarketBalanceAddFee: types.MustParseFIL("0.007"),
        },
```

------------------------

## 系统预估的GasLimit

系统调用`GasEstimateGasLimit`方法来预估GasLimit，实际上就是新建一台虚拟机，用虚拟机跑了一遍消息。再将虚拟机执行结果中的`GasUsed`作为`GasLimit`返回。

不用担心，此时的虚拟机运行结果不会上链，并不会导致账户发生变化。

它将消息中的gas暂时设为

> GasLimit=10000000000 此为一个区块的总的gasLimit上限，以保证gasLimit足够大
> 
> GasFeeCap=101 这是最小的baseFee+1

因此，能够成功进行`GasLimit`估算的先决条件是，`msg.From`里的钱要大于`10000000000*101`atto 。  

为了模拟执行消息，它先用虚拟机，把在消息池中的消息执行一遍，然后再执行要计算gasUsed的消息。最后返回gasUsed。  

值得一提的是，最后将计算出来的`gasUsed`赋值给消息中的`GasLimit`时，官方代码中将其乘以一个系数**1.25**，使`msg.GasLimit`的值较大。

## 系统预估矿工小费

官方代码预估`GasPremium` 所用的主要参数有`nblocksincl` 其值用来回溯2*`nblocksincl`个`tipset`。  
目前默认`nblocksincl`为2，也就是会向前回溯4个`tipset`。  

预估矿工小费的具体算法为：

列出4个`tipset`内所有消息的`gasLimit`和`GasPremium`，将消息按照`gasLimit`从大到小排列，以前n个消息的`gasLimit`总和为${目标gas\cdot 前4个TipSet的区块总数\over 2}$

即，  

$$
m_1+m_2+m_3+m_4+···+m_{n-1}<={5\times 10^9\cdot 前4个TipSet的区块总数\over 2}<=m_1+m_2+m_3+m_4+···+m_n
$$

其中，${m_n}$为第n个消息中的`gasLimit`。

则预估`GasPremium`为

$$
GasPremium=
\begin{cases}
    &{{p_{n-1}+p_n}\over 2}&, \text{（当 $P_{n-1}\neq0 $时）}\\
&1 &,\text{（当$P_{n-1}=0 $时）}
\end{cases}
$$

${P_n}$为第n条消息的`GasPremium`。

当所有消息的`gasLimit` 总和依然小于${5\times 10^9\cdot 前4个TipSet的区块总数\over 2}$ 时， 

$$
GasPremium=
\begin{cases}
    &2\cdot MinGasPremium&=2\times 10^3  &,\text{（当 ${nblocksincl=1}$时）}\\
    &1.5\cdot MinGasPremium &=1.5\times 10^3  &,\text{（当 ${nblocksincl=2}$时）}\\
    &MinGasPremium&=10^3 &,\text{（当 ${nblocksincl\neq 1 }$且${nblocksincl\neq 2}$）时}
\end{cases}
$$

# baseFee的计算

与baseFee计算相关的变量有：

> 上一个tipSet的baseFee  : `parentBaseFee`    
> 当前tipSet中消息的gasLimit总数 :`totalLimit`   
> 当前tipSet的区块数量 : `noOfBlocks`  

其计算过程如下，

$$
\begin{aligned}
baseFee&=parentBaseFee+{{parentBaseFee \cdot  ({totalLimit \over noOfBlocks}-BlockGasTarget)\over BlockGasTarget}\over BaseFeeMaxChangeDenom}\\
&=parentBaseFee+{{parentBaseFee \cdot  ({totalLimit \over noOfBlocks}-BlockGasTarget)\over BlockGasTarget\cdot BaseFeeMaxChangeDenom}}\\
\text{又因为，}\\
&BlockGasTarget=5000000000\\
&BaseFeeMaxChangeDenom=8\\
\text{即,}\\
BF_n&=BF_{n-1}+{{BF_{n-1} \cdot  ({TL \over BLKS}-5000000000)\over 5000000000\times 8}}\\
&=BF_{n-1}+{BF_{n-1} \cdot  ({TL \over BLKS}-5\times 10^9)\over 5\times 10^9\times 8}\\
&=BF_{n-1}+{BF_{n-1} \cdot  ({TL \over BLKS}-5\times 10^9)\over 4\times 10^{10}}\\
&=BF_{n-1}\cdot({TL\over BLKS}\cdot 4\times 10^{-10}+{7\over 8})

\end{aligned}
$$

上述算式中，  

> `BLKS`=`tipSet`中的区块数量  
> `BF`=`baseFee`,字符`n`表示当前为第几个`Epoch`  
> `TL`=当前同一`TipSet`中所有消息的`GasLimit`总和   

因为`BlockGasTarget`为区块消息中gasLimit之和的一半，所以可将上述式子理解为

$$
\begin{aligned}
 baseFee=上轮的baseFee\text{ } \times ({区块平均gasLimit\over 区块目标gasLimit \times 8}+{7\over 8})
\end{aligned}
$$

又因为区块的GasLimit有上限值，其值为区块目标Gas的2倍  
所以`baseFee`最大值为上一轮的${9\over8}$ ;  
`baseFee`最小值为上一轮的${7 \over8}$ ;  
即，`baseFee`的变化幅度最大为${1\over8}$ 。

$$
baseFee=ParentBaseFee+ParentBaseFee\times \Delta , \Delta \in [-\frac{1}{8},\frac{1}{8}]
$$

# 聚合费

$$
聚合费={\max(baseFee,5\ nanoFil)\times 扇区个数 \times gasUsed \over 20}
$$
