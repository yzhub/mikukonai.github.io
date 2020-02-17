
#!title:    JPEG图像压缩算法
#!date:     2019-07-07
#!authors:  Mikukonai
#!cover:    
#!type:     原创
#!tags:     

#!content

请尽量使用PC以获得最佳演示效果。

<iframe class="MikumarkIframe" src="./html/JPEG.html" height="450px"></iframe>

**说明**：演示了JPEG有损压缩算法的基本原理。熵编码部分尚未完成。DCT和IDCT采用Makhoul提出的快速算法[[1]](#参考资料)进行计算，其中FFT使用C-T算法实现。

# 参考资料

+ Makhoul J . A Fast Cosine Transform in One and Two Dimensions[J]. IEEE Transactions on Acoustics Speech and Signal Processing, 1980, 28(1):27-34.
+ [Fast cosine transform via FFT](https://dsp.stackexchange.com/questions/2807/fast-cosine-transform-via-fft)
+ [Fast discrete cosine transform algorithms](https://www.nayuki.io/page/fast-discrete-cosine-transform-algorithms)
+ [ITU T.81](https://www.w3.org/Graphics/JPEG/itu-t81.pdf)
