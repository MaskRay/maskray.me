---
layout: post
title: 整係數Discrete Fourier transform技巧
author: MaskRay
tags: [algorithm, math, floating point]
mathjax: true
---

## Discrete Fourier transform

<!-- more -->

向量$\mathbf{a}$的DFT $\mathbf{A}$定義爲：

$$A_j=\sum_{k=0}^{n-1}{a_k\omega^{jk}}$$

其中$\omega$是primitive root of unity of order $n$，即：$\omega^n=1$且$\omega^j!=1 \text{ for } 0<j<n$。
通常取$exp(2i\pi/n)$或$exp(-2i\pi/n)$。

## Inverse transform

性質：$\frac{1}{n}\sum_{k=0}^{n-1}{\omega^{jk}} = [j\;mod\;n=0]$ (j模n爲0時取1，否則取0)

$$\sum_{k=0}^{n-1}{A_k\omega^{-jk}} =
\sum_{k=0}^{n-1}{\sum_{l=0}^{n-1}{a_l\omega^{kl}}\omega^{-jk}} =
\sum_{k=0}^{n-1}{\sum_{l=0}^{n-1}{a_l\omega^{(l-j)k}}} =
\sum_{l=0}^{n-1}{a_l\sum_{k=0}^{n-1}{\omega^{(l-j)k}}} =
\sum_{l=0}^{n-1}{n[(l-j)\;mod\;n=0]} =
na_j$$

因此

$$a_j=\frac{1}{n}\sum_{k=0}^{n-1}{A_k\omega^{-jk}}$$

假設加減乘除時間複雜度爲$O(1)$，樸素實現的時間複雜度爲$O(n^2)$。
Cooley–Tukey algorithm是最常見的fast Fourier transform算法，時間複雜度爲$O(n log(n))$。
有radix-2、radix-4、split-radix等多種形式。

注意使用的$\omega^{-k}$是DFT用的係數的倒數。如果使用DFT實現IDFT可以先翻轉向量：
```cpp
reverse(a+1, a+n);

// compute DFT

for (long i = 0; i < n; i++)
  a[i] *= 1.0/n;
```

## Decimation in time FFT

Cooley-Tukey是一種radix-2分治算法，把偶數下標的子序列的Fourier transform與奇數下標的子序列的Fourier transform，用$O(n)$時間合併。

Decimation in time在變換之後對下標進行bitreverse操作。

使用Stockham FFT可以省去bitreverse操作。

## Decimation in frequency FFT

Sande-Tukey是另一種radix-2分治算法，把前一半子序列的Fourier transform與後一半子序列的Fourier transform，用$O(n)$時間合併。

Decimation in frequency在變換之前對下標進行bitreverse操作。如果Decimation in time的DFT與Decimation in frequency的IDFT一起用，可以省略兩處bitreverse操作。

DIF的bufferfly操作不適合fused multiply-add。

## Memory-bound

FFT是memory-bound的。比較快的實現會在N較大時使用遞歸，而N較小時使用迭代。

## Number theoretic transform

DFT可以用於複數以外的其他ring，常用於$\mathbb{Z}/m$。

使用128 bits模數需要高效的`u64*u64%u64`，其中模數是常數。

### 硬件除法指令(32 bits、64 bits)

`DIV`指令性能很低。

```c
extern inline u64 mul_mod(u64 a, u64 b, u64 m)
{
  u64 r;
  asm("mulq %2\n\tdivq %3" : "=&d"(r), "+a"(a) : "rm"(b), "rm"(m) : "cc");
  return r;
}
```

到AVX-512也沒有提供把兩個64 bits乘數的積放在一個128 bits寄存器的指令，GCC沒有提供用乘法、移位等模擬的u128除以u64的常量除法。

### 64位mantissa浮點數(32 bits、64 bits)

當模數$P<2^{63}$時可以用64位mantissa浮點數計算`u64*u64%u64`。

由等式$$(a\cdot b\% m) = a\cdot b - \lfloor\frac{a\cdot b}{m}\rfloor\cdot m$$

兩邊模$2^{64}$，得
\begin{eqnarray*}
(a\cdot b\% m)\% 2^{64} &=& (a\cdot b - \lfloor\frac{a\cdot b}{m}\rfloor\cdot m) \% 2^{64} \\
&=& ((a\cdot b)\%2^{64} - (\lfloor\frac{a\cdot b}{m}\rfloor\cdot m)\%2^{64}) \% 2^{64}
\end{eqnarray*}

即用`u64`乘法計算$a\cdot b$的低64位，減去$\lfloor\frac{a\cdot b}{m}\rfloor\cdot m$的低64位。其中$\lfloor\frac{a\cdot b}{m}\rfloor<m$，可以用64位mantissa浮點數(Intel x87 80-bit double-extended precision)計算，再round成`u64`。

round時若向上取整了，減數會大於被減數。若$m<2^{63}$，可以根據差的符號位判斷。

```c
u64 mul_mod(u64 a, u64 b, u64 P)
{
  u64 x = a*b, r = x - P*u64((long double)a*b/P+0.5);
  return i64(r) < 0 ? r + P : r;
}
```

存儲$P$的倒數$Q=\frac{1}{P}$，用`(long double)a*b*Q`代替`(long double)a*b/P`能快些。此時$Q$會引入額外的誤差，Matters Computational說適用於$m<2^{62}$，原因不明。

### 編譯器生成的常量除法(32 bits)

對於固定的模$P$，GCC/llvm可以生成`u64%u32`的高效代碼。llvm的`lib/Transforms/Utils/IntegerDivision.cpp`。

### Montgomery modular multiplication+Barret reduction(32 bits)

Faster arithmetic for number-theoretic transforms，用乘法和移位代替除法。

快速計算`u64%u32`，用乘法和移位代替除法。設$2^{m-1}\leq P<2^m$，$\alpha$爲大於等於$m$的整數，$\beta$爲負整數。

\begin{eqnarray*}
Q &=& \frac{\lfloor\frac{a}{2^{m+\beta}}\rfloor\lfloor\frac{2^{m+\alpha}}{P}\rfloor}{2^{\alpha-\beta}} \\
&\leq& \frac{\frac{a}{2^{m+\beta}}\frac{2^{m+\alpha}}{P}}{2^{\alpha-\beta}} \\
&=& \frac{a}{P} \\
\end{eqnarray*}

$Q\leq\frac{a}{P}$，$a-QP$是$a\%P$的估計值，若大於等於$P$則減去$P$的倍數。

\begin{eqnarray*}
Q &>& \frac{(\frac{a}{2^{m+\beta}}-1)(\frac{2^{m+\alpha}}{P}-1)}{2^{\alpha-\beta}}-1 \\
&=& \frac{a}{P}-\frac{a}{2^{m+\alpha}}-\frac{2^{m+\beta}}{P}+2^{\beta-\alpha}-1 \\
&\geq& \lfloor\frac{a}{P}\rfloor-\frac{a}{2^{m+\alpha}}-\frac{2^{m+\beta}}{P}+2^{\beta-\alpha}-1 \\
\end{eqnarray*}

因此

\begin{eqnarray*}
0\leq \lfloor\frac{a}{P}\rfloor-Q &\leq& \frac{a}{2^{m+\alpha}}+\frac{2^{m+\beta}}{P}-2^{\beta-\alpha}+1  \\
\end{eqnarray*}

令$\alpha=m, \beta=-1$，則$0\leq \lfloor\frac{a}{P}\rfloor-Q \leq 2$，即估計值最多小2，最多兩個conditional move指令(`if (r >= P) r -= P;`)即可修正餘數。

令$\alpha=m+1, \beta=-2$，則估計值最多小1。

```c
extern inline u64 barrett_30_2(u64 a, u64 P, u64 M)
{ // 2^29 < P < 2^30, M = floor(2^61/P)
  u64 r = a-((a>>28)*M>>33)*P;
  if (r >= P) r -= P;
  return r;
}

extern inline u64 barrett_30_1(u64 a, u64 P, u64 M)
{ // 2^29 < P < 2^30, M = floor(2^60/P)
  u64 r = a-((a>>29)*M>>31)*P;
  if (r >= P) r -= P;
  if (r >= P) r -= P;
  return r;
}
```

當模數爲常量時，該算法不如編譯器生成的常量除法。若模數不固定時可以考慮使用。

## Cyclic convolution

兩個長爲$n$的序列$\mathbf{a}$與$\mathbf{b}$的cyclic convolution的長度也是$n$。第$j$項定義爲：

$$(\mathbf{a}\ast \mathbf{b})_j = \sum_{k=0}^{n-1}{a_kb_{(j-k)\;mod\;n}}$$

\begin{eqnarray*}
  (a\ast b)_j &=& \sum_{p=0}^{n-1}{\sum_{q=0}^{n-1}{[(p+q)\;mod\;n=j]a_pb_q}}\\
  &=& \sum_{p=0}^{n-1}{\sum_{q=0}^{n-1}{[(p+q-j)\;mod\;n=0]a_pb_q}}\\
  &=& \sum_{p=0}^{n-1}{\sum_{q=0}^{n-1}{\frac{1}{n}\sum_{k=0}^{n-1}{\omega^{pk}\omega^{qk}\omega^{-jk}}a_pb_q}}\\
  &=& \frac{1}{n}\sum_{k=0}^{n-1}{(\sum_{p=0}^{n-1}{a_p\omega^{kp}})(\sum_{q=0}^{n-1}{b_q\omega^{kq}})\omega^{-jk}} \\
  &=& \frac{1}{n}\sum_{k=0}^{n-1}{(A_kB_k)\omega^{-jk}} \\
\end{eqnarray*}

## Linear convolution

兩個長爲$n$的序列$\mathbf{a}$與$\mathbf{b}$的linear convolution的長度是$2n$。

$$(\mathbf{a}\ast_l \mathbf{b})_j = \sum_{k=0}^{j}{a_kb_{(j-k)\;mod\;n}}$$

第$2n-1$項總是0。多項式乘法是一種常見的linear convolution應用。

Zero pad原序列到長度$2n$後，計算cyclic convolution即可得到linear convolution。

$$\mathbf{a'} = [a_0,a_1,\ldots,a_{n-1},0,0,\ldots,0]$$

$$\mathbf{b'} = [b_0,b_1,\ldots,b_{n-1},0,0,\ldots,0]$$

$$(\mathbf{a'}\ast_l \mathbf{b'})_j = \sum_{k=0}^{j}{a_kb_{j-k}}\text{ if }0<=j<n$$

## 範圍和精度

向量卷積需要注意精度問題。

使用`complex<double>`計算convolution，需要保證結果每一項的實數部分在$[-2^{53}+1, 2^{53}-1]$(`Number.MIN_SAFE_VALUE`、`Number.MAX_SAFE_INTEGER`)範圍內，$2^{53}-1$是double能精確表示的最大整數。採取round half to even規則，$2^{53},2^{53}+1$均表示爲$2^{53}$，無法區分。

設每項係數的絕對值小於等於$v$，那麼convolution結果每一項絕對值小於等於$nv^2$，若$nv^2\leq 2^{53}-1$則可放心使用`complex<double>`。

`complex<double>`還要受到浮點運算誤差影響。根據Roundoff Error Analysis of the Fast Fourier Transform，沒仔細看，relative error均值爲`log2(n)*浮點運算精度*變換前係數最大值`，對於結果$x$，這個量達到$0.5/x$就很可能出錯。增長速度可以看作是$log(n)v^2$，不如$nv^2$。因此通常不必擔心浮點運算的誤差。

對於模$P$的number theoretic transform，$v\leq P-1$，若$nv^2\leq P$則可放心使用。

關於精度，另外可參考<https://github.com/simonlindholm/fft-precision/blob/master/fft-precision.md>。

1004535809 (=479*2**21+1), 998244353 (=119*2**23+1), 897581057 (=107*2**23+1)，這三個數均小於$2^{30}$，兩倍不超過`INT32_MAX`(兩個+-P之間的數加減不會超出int32_t表示範圍)，且可表示爲$k*n+1$，其中$n$爲2的冪，適合用作number theoretic transform的模。3是它們共同的原根。
另一个候選素數爲880803841 (=105*2**23+1)，26是一個原根。

設$\mathbf{a},\mathbf{b}$係數取自$[0,v]$的uniform distribution，則$\mathbf{a}\ast\mathbf{b}$係數均值爲$nv^2/4$，方差爲$nv^4/9$。若把係數平移至$[-v/2, v/2]$，則$\mathbf{a}\ast\mathbf{b}$係數均值$0$，方差爲$nv^4/144$。若$\mathbf{a},\mathbf{b}$其中之一independent and identically distributed，則方差會很小。可以用Chebyshev's inequality等估計係數絕對值，上界$nv^2$可以減小。即使$\mathbf{a},\mathbf{b}$不是independent and identically distributed，也可以用$\mathbf{a}\ast\mathbf{b}=\mathbf{a}\ast(\mathbf{b+c})-\mathbf{a}\ast\mathbf{c}$來計算，$\mathbf{c}$是independent and identically distributed uniform $[-v/2,v/2]$。

下面考察模P多項式乘法，碰到精度問題時的兩種應對方案：sqrt decomposition、Chinese remainder theorem。

## 方案0：sqrt decomposition(FFT, NTT)

適用於FFT與NTT。
取$M$爲接近$\sqrt{v}$的整數，分解$\mathbf{a}=\mathbf{a0}+M\mathbf{a1}$、$\mathbf{b}=\mathbf{b0}+M\mathbf{b1}$，則：

$$\mathbf{a}\ast\mathbf{b} = \mathbf{a0}\ast\mathbf{b0}+M(\mathbf{a0}\ast\mathbf{b1}+\mathbf{a1}\ast\mathbf{b0})+M^2(\mathbf{a1}\ast\mathbf{b1})$$

適當選擇$M$可以使$\mathbf{a0},\mathbf{a1}$的係數小於等於$\lfloor\sqrt{v}\rfloor$，convolution結果係數最大值爲$n(\lfloor\sqrt{v}\rfloor)^2\approx nv$，比原來的$nv^2$小。

求出$DFT(\mathbf{a0}), DFT(\mathbf{a1}), DFT(\mathbf{b0}), DFT(\mathbf{b1})$後，計算等式右邊四個convolution，帶權相加即得到原convolution。

如上樸素方案需要4次長爲$2n$的DFT、1次長爲$2n$的inverse DFT。

一種優化方案是使用Toom-2 (Karatsuba)計算$\mathbf{a0}\ast\mathbf{b0}, \mathbf{a1}\ast\mathbf{b1}, (\mathbf{a0}+\mathbf{a1})\ast(\mathbf{b0}+\mathbf{b1})$，可以減少爲3次DFT、1次inverse DFT。$\mathbf{a0}\ast\mathbf{b1}+\mathbf{a1}\ast\mathbf{b0} = (\mathbf{a0}+\mathbf{a1})\ast(\mathbf{b0}+\mathbf{b1}) - \mathbf{a0}\ast\mathbf{b0} - \mathbf{a1}\ast\mathbf{b1}$。

容易擴展到cube root decomposition等。對於number theoretic transform，分成$k$份需要$2k$個DFT和$2k-1$個IDFT，不如用Chinese remainder theorem。

### 優化0：實部表示低位、虛部表示高位

FFT可以計算複數的DFT，但在樸素的多項式乘法中，FFT只作用於實數向量，虛數部分浪費了。
sqrt decomposition把一個係數拆分成兩項，我們可以把兩項裝載到實數部分和虛數部分。
具體方法如下：

取正整數$M$與$\sqrt{P}$接近且$Z=P-M*M\% P$是一個儘可能小的正整數。

分解$\mathbf{a}=\mathbf{a0}+M\mathbf{a1}$、$\mathbf{b}=\mathbf{b0}+M\mathbf{b1}$。
考察如下的變換：

\begin{eqnarray*}
IDFT(DFT(\mathbf{a0}+i\sqrt{Z}\mathbf{a1})\cdot DFT(\mathbf{b0}+i\sqrt{Z}\mathbf{b1})) &=& (\mathbf{a0}+i\sqrt{Z}\mathbf{a1})\ast (\mathbf{b0}+i\sqrt{Z}\mathbf{b1}) \\
&=& (\mathbf{a0}\ast \mathbf{b0})-Z(\mathbf{a1}\ast\mathbf{b1})+i\sqrt{Z}((\mathbf{a0}\ast\mathbf{b1})+(\mathbf{a1}\ast\mathbf{b0})) \\
\end{eqnarray*}

注意每一項絕對值的值域變爲$\sqrt{1+Z}$倍了，因此計算對精度有更高的要求。取較小的$Z$可以降低精度要求。

右邊同餘於$(\mathbf{a0}\ast \mathbf{b0})+M^2(\mathbf{a1}\ast\mathbf{b1})+i\sqrt{Z}(\mathbf{a0}\ast\mathbf{b1}+\mathbf{a1}\ast\mathbf{b0})$。
提取虛部與實部，將虛部除以$i\sqrt{Z}$再乘以$M$，再加上實部即得：

$$(\mathbf{a0}\ast \mathbf{b0})+M(\mathbf{a0}\ast\mathbf{b1}+\mathbf{a1}\ast\mathbf{b0})+M^2(\mathbf{a1}\ast\mathbf{b1})$$

正是我們需要的形式。

這個優化需要2次長爲$2n$的DFT、1次長爲$2n$的inverse DFT。

### 優化1：正交計算兩個實係數向量DFT(FFT)

這是另一種利用向量的虛數部分的方法。

$\mathbf{a}$的共軛的DFT可由$\mathbf{a}$的DFT求出：

\begin{eqnarray*}
  DFT(conj(\mathbf{a}))_j &=& \sum_{k=0}^{n-1}{conj(a_k)\omega^{jk}} \\
  &=& \sum_{k=0}^{n-1}{conj(a_k\omega^{-jk})} \\
  &=& conj(\sum_{k=0}^{n-1}{a_k\omega^{-jk}}) \\
  &=& conj(DFT(\mathbf{a})_{-j\;mod\;n}) \\
  &=& conj(rev(DFT(\mathbf{a}))_j) \\
\end{eqnarray*}

\begin{eqnarray*}
DFT(re(\mathbf{a})) &=& (DFT(\mathbf{a})+DFT(conj(\mathbf{a})))/2 = (DFT(\mathbf{a}) + conj(rev(DFT(\mathbf{a}))))/2 \\
DFT(im(\mathbf{a})) &=& (DFT(\mathbf{a})-DFT(conj(\mathbf{a})))/(2i) = (DFT(\mathbf{a}) - conj(rev(DFT(\mathbf{a}))))/(2i) \\
\end{eqnarray*}

換言之，計算一個複數向量的DFT，可以通過簡單變換，得到實數部分向量的DFT與實數部分向量的DFT。買一送一。

分解$\mathbf{a}=\mathbf{a0}+M\mathbf{a1}$、$\mathbf{b}=\mathbf{b0}+M\mathbf{b1}$後，根據上面的公式，用$DFT(\mathbf{a0}+i\mathbf{a1})$計算$DFT(\mathbf{a0})$和$DFT(\mathbf{a1})$，同法計算$DFT(\mathbf{b0})$和$DFT(\mathbf{b1})$。然後用$IDFT(DFT(\mathbf{a0})\cdot DFT(\mathbf{b0}) + i DFT(\mathbf{a0})\cdot DFT(\mathbf{b1}))$計算出$\mathbf{a0}\ast\mathbf{b0}$與$\mathbf{a0}\ast\mathbf{b1}$，同法計算出$\mathbf{a1}\ast\mathbf{b0}$與$\mathbf{a1}\ast\mathbf{b1}$。

需要2次長爲$2n$的DFT、2次長爲$2n$的inverse DFT。

### 奇偶項優化(FFT)

該優化可以和其他方式疊加。

把$\mathbf{a}$看作多項式$A(x)=\sum_{k=0}^{n-1}{a_kx^k}$，同樣地，$\mathbf{b}$看作多項式$B(x)$。

偶次項$A0(x)=\sum_{0\leq k<n, k\;\text{is even}}{a_kx^k}$, 奇次項$A1(x)=\sum_{0\leq k<n, k\;\text{is odd}}{a_kx^k}$，同樣地，定義$B0(x)$與$B1(x)$。$A0(x)$的係數爲$\mathbf{a0}$，$A1(x)$的係數爲$\mathbf{a1}$，令其長爲$n$，高位用$0$填充。

\begin{eqnarray*}
A(x)B(x) &=& (A0(x^2)+x A1(x^2))(B0(x^2)+x B1(x^2)) \\
&=& A0(x^2)B0(x^2)+x^2A1(x^2)B1(x^2) + x(A0(x^2)B1(x^2)+A1(x^2)B0(x^2))  \\
\end{eqnarray*}

用正交計算兩個實係數向量DFT的方式，用2次長度爲$n$(之前都是  $2n$)的DFT計算$DFT(\mathbf{a0}), DFT(\mathbf{a1})$, $DFT(\mathbf{b0}), DFT(\mathbf{b1})$。$\mathbf{a1}$循環右移1位的DFT的第$j$項等於$DFT(\mathbf{a1})_j\omega^j$，因此根據$A1(x^2)$的DFT的係數可以得到$x^2A1(x^2)$的DFT的係數。

構造長爲$n$的向量$\mathbf{c}$：
$$c_j = (a0_j b0_j + a1_j b1_j \omega^j) + i(a0_j b1_j + a1_j b0_j)$$

$DFT(\mathbf{c})$的實部爲結果的偶次項係數，虛部爲結果的奇次項係數。

需要2次長爲$n$的DFT、1次長爲$n$的inverse DFT。

## 方案1：Chinese remainder theorem(NTT)

適用於NTT。
取$t$個可用於number theoretic transform的質數$P_0,P_1,\ldots,P_{t-1}$，使$M = \prod_{0\leq j<t}{P_j}\geq nv^2$，計算$t$個NTT，之後用Chinese remainder theorem合併。

求$x \equiv v_j\quad(\text{mod}\;M/P_j),\quad 0\leq j<t$有至少兩種算法。

### 經典算法(Gauss's algorithm)

Gauss之前也有很多人提出。

對於每個$P_j$用Blankinship's algorithm計算$P_j p_j \equiv 1\quad(\text{mod}\;M/P_j)$。

$$x = (\sum_{j=0}^{t-1}{v_jP_jp_j}) \% M$$

注意$M\geq nv^2$可能超出機器single-precision表示範圍，該算法不適合求$x\% P$。

### Garner's algorithm

定義
\begin{eqnarray*}
c_1 &=& inv(P_0, P_1) \\
c_2 &=& inv(P_0P_1, P_2) \\
c_3 &=& inv(P_0P_1P_2, P_3) \\
\ldots \\
\end{eqnarray*}

\begin{eqnarray*}
y_0 &=& v_0 \% P_0  & x_0 &=& y_0 \\
y_1 &=& (v_1-x_0)c_1 \% P_1 & x_1 &=& x_0 + y_1P_0 \\
y_2 &=& (v_2-x_1)c_2 \% P_2 & x_2 &=& x_1 + y_2P_0P_1 \\
y_3 &=& (v_3-x_2)c_3 \% P_3 & x_3 &=& x_2 + y_3P_0P_1P_2 \\
\ldots \\
\end{eqnarray*}

$x_j$滿足前$j+1$個方程，$x=x_{t-1}$滿足所有方程。

稍加變形可用於求$x\% P$。

\begin{eqnarray*}
y_0 &=& v_0 \% P_0  & x_0 &=& y_0 \% P \\
y_1 &=& (v_1-(y_0)\% P_1)c_1 \% P_1 & x_1 &=& (x_0 + y_1P_0) \% P \\
y_2 &=& (v_2-(y_0+y_1P_0)\% P_2)c_2 \% P_2 & x_2 &=& (x_1 + y_2P_0P_1) \% P \\
y_3 &=& (v_3-(y_0+y_1P_0+y_2P_0P_1)\% P_3)c_3 \% P_3 & x_3 &=& (x_2 + y_3P_0P_1P_2) \% P \\
\ldots \\
\end{eqnarray*}

原來的每個$x_j$是精確值，現在只有$x_j%P$的結果，因此計算$y_j$時之前的$x_{0\.dotsj-1}$都不可復用，需要重新計算。時間複雜度上升爲$O(t^2)$。

## 測試

<https://gist.github.com/MaskRay/fac2042058dd5d9e59953f18f3f3978a>

NTT int使用小于$2^{31}$的素数，编译器用乘法模拟常数除法。
NTT long，编译器无法优化常数除法，性能很低，使用浮点mul_mod会略快于128位除以64位的DIV指令。

```
n	microseconds	algorithm

131072  3860    NTT dit2 int
131072  6104    FFT dif2
131072  6712    FFT dit2
131072  6912    NTT dif2 int
131072  6936    Montgomery+Barrett NTT dif2 int
131072  9592    NTT dif2 long non-constant P
131072  10122   NTT dit2 long
131072  13169   NTT dif2 int non-constant P
131072  15419   NTT dif2 long

262144  8993    NTT dif2 int
262144  9036    NTT dit2 int
262144  9670    Montgomery+Barrett NTT dif2 int
262144  15484   FFT dit2
262144  17601   FFT dif2
262144  19731   NTT dit2 long
262144  20527   NTT dif2 long non-constant P
262144  21910   NTT dif2 long
262144  29457   NTT dif2 int non-constant P

524288  18502   NTT dif2 int
524288  20110   Montgomery+Barrett NTT dif2 int
524288  23156   NTT dit2 int
524288  39890   FFT dif2
524288  39904   FFT dit2
524288  44145   NTT dif2 long non-constant P
524288  45038   NTT dit2 long
524288  46334   NTT dif2 long
524288  65265   NTT dif2 int non-constant P

1048576 43648   NTT dit2 int
1048576 45704   NTT dif2 int
1048576 46167   Montgomery+Barrett NTT dif2 int
1048576 104362  NTT dit2 long
1048576 107571  NTT dif2 long non-constant P
1048576 119743  FFT dif2
1048576 122029  NTT dif2 long
1048576 122174  FFT dit2
1048576 144370  NTT dif2 int non-constant P

2097152 122989  Montgomery+Barrett NTT dif2 int
2097152 137276  NTT dif2 int
2097152 143955  NTT dit2 int
2097152 293222  FFT dif2
2097152 338580  FFT dit2
2097152 352833  NTT dif2 int non-constant P
2097152 360372  NTT dif2 long non-constant P
2097152 422108  NTT dit2 long
2097152 423817  NTT dif2 long

4194304 455859  NTT dit2 int
4194304 467340  NTT dif2 int
4194304 490114  Montgomery+Barrett NTT dif2 int
4194304 779945  FFT dif2
4194304 839698  FFT dit2
4194304 904096  NTT dit2 long
4194304 956174  NTT dif2 long
4194304 969572  NTT dif2 long non-constant P
4194304 1074858 NTT dif2 int non-constant P

8388608 1052072 NTT dit2 int
8388608 1138089 NTT dif2 int
8388608 1189775 Montgomery+Barrett NTT dif2 int
8388608 1737166 FFT dif2
8388608 1839095 FFT dit2
8388608 2053195 NTT dif2 long
8388608 2072172 NTT dif2 long non-constant P
8388608 2186451 NTT dit2 long
8388608 2893584 NTT dif2 int non-constant P
```

花哨的Montgomery+Barrett不如常量除法的NTT int，好處是一份代碼可以適用於多個模數，而NTT int得用template或其他方式爲各個模數生成不同代碼。

不受到Level 3 cache制約時，Montgomery NTT只需要NTT int 60%的時間，此時每次重新計算unit root代替lookup table會快些。

一般來說，decimation in frequency(Sande-Tukey，從較大的n計算到較小的n)優於decimation in time(Cooley-Tukey，從較小的n計算到較大的n)，可能是因爲decimation in frequency的butterfly數據依賴小些。

FFT有超過10%時間花在計算unit roots上，而NTT只有5%。考慮到FFT往往能正交計算兩個序列，而NTT只能計算一個，且double有53位精度而NTT int只能使用$2^{32}$以下的素數(當前代碼只能處理$2^{31}$以下的)，FFT通常優於NTT。

## References

感謝ftiasch老師教導。

- 黄嘉泰(fotile96), <https://async.icpc-camp.org/d/408-fft>
- Jörg Arndt, Matters Computational
- Handbook of Applied Cryptography
- 毛嘯，再探快速傅裏葉變換，2016年信息學奧林匹克中國國家隊候選隊員論文集
- [Faster arithmetic for number-theoretic transforms](https://arxiv.org/pdf/1205.2926.pdf)
- [Speeding Up Barrett and Montgomery Modular Multiplications](http://homes.esat.kuleuven.be/~fvercaut/papers/bar_mont.pdf)

uwi，<https://www.hackerrank.com/rest/contests/w23/challenges/sasha-and-swaps-ii/hackers/uwi/download_solution>：Modified Montgomery+Barrett變體+Garner's algorithm：
