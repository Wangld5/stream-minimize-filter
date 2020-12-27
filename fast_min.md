```
Input: a from 1 to n; wsize > 2
Set: U deque; mval array
append 1 to U
for i in {2,...,n} do:
	if i>=w+1 then
		output U(front) => mval[i-w]
	if a[i] < a[i-1] then
		pop U from back
		while a[i] < a[U(back)] do
			pop U from back
	append i to U
	if i = w + U(front) then
		pop U from front
```

这个算法的效率是与窗口大小无关的，为什么呢，因为这个算法会维护一个队列，而这个队列是一个单调递增的队列，队列的第一个值永远是最小值。然后窗口的大小只是限制这个最小值作用的范围是在[最小值索引-窗口值，最小值索引+窗口值]之间，理解这一思想就能理解这个算法的思路了：

首先设置一个保存位置的双向队列和一个保存结果的列表，把一行数据中的第一个值的位置放进队列中。然后开始遍历：

从第二个值开始：

第一步：如果遍历完了一个窗口长度，到了能够将第一个窗口的最小值放进结果列表里面了，就把双向队列的第一个值取出来作为下标从这一行数据中取值加入到结果列表中。如果没有遍历完一个窗口就不用管。

第二步：是对双向队列的操作。首先判断遍历到的值是不是小于上一个值，如果是，那说明队列中的最后一个值要让位给当前遍历到的数据的下标，不仅如此，还需要向前遍历队列中的每一个值，如果队列中的值作为下标从数据中找到的值大于遍历到的数据的值，那么会把队列中的最后一个值pop掉，直到如果队列中的值作为下标从数据中找到的值小于遍历到的数据的值。

第三步：将遍历到的行数据的值的位置加入队列中。

第四步：如果队列第一个值+窗口长度等于遍历到的位置了，那么就说明队列中第一个值的影响范围到头了，就把这个值从队列头中删除掉。

最后：遍历完所有值一遍之后因为遍历机制的缘故保存结果列表还需要插入最后一个值，这一个值还是取队列的第一个值作为下标去行数据那里找就可以了。

整个算法的计算复杂度是O（1），因为不需要对每一个窗口都进行检索，算法进行的操作只有取队列的第一个值的操作而已。

举个例子：

![1605181218156](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1605181218156.png)

```python
## 二维的实现，先在行数据上操作，再在列方向上操作
	def fast_minimum_filter(self, image, w):
        min_row = self.fast_minimum_filter_row(image, w)
        min_res = self.fast_minimum_filter_col(min_row, w)
        return np.transpose(min_res)

    def fast_minimum_filter_row(self, a, w):
        row = a.shape[0]
        col = a.shape[1]
        all_min = []
        for i in range(0, row):
            mininfo = deque()
            mininfo.append(0)
            minvalues = [None for _ in range(col-w+1)]
            for j in range(1, col):
                if j >= w:
                    minvalues[j-w] = a[i][mininfo[0]]
                if a[i][j] < a[i][j-1]:
                    mininfo.pop()
                    while mininfo:
                        if a[i][j] >= a[i][mininfo[-1]]:
                            break
                        mininfo.pop()
                mininfo.append(j)
                if j == (w+mininfo[0]):
                    mininfo.popleft()
                minvalues[col-w] = a[i][mininfo[0]]
            all_min.append(minvalues)
            del mininfo
            del minvalues
        return np.array(all_min)
    
    def fast_minimum_filter_col(self, a, w):
        row = a.shape[0]
        col = a.shape[1]
        all_min = []
        for j in range(0, col):
            mininfo = deque()
            mininfo.append(0)
            minvalues = [None] * (row-w+1)
            for i in range(1, row):
                if i >= w:
                    minvalues[i-w] = a[mininfo[0]][j]
                if a[i][j] < a[i-1][j]:
                    mininfo.pop()
                    while mininfo:
                        if a[i][j] >= a[mininfo[-1]][j]:
                            break
                        mininfo.pop()
                mininfo.append(i)
                if i == (w+mininfo[0]):
                    mininfo.popleft()
                minvalues[row-w] = a[mininfo[0]][j]
            all_min.append(minvalues)
            del mininfo
            del minvalues
        return np.array(all_min)
```

### softmatting

softmatting的做法是将原图像的问题抠出来生成单通道的图像作为输入图像，然后将原图像作为引导图像进行联合双边滤波，但是即使是这种做法最后产生的图像也会有光晕效应，而且带有梯度反转。

#### 梯度反转问题

先写一下双边滤波的公式：
$$
BF[I]_p=\frac{1}{W_p}\sum_{q\in S}G_{\sigma{_s}}(||p-q||)G_{\sigma{_r}}(||p-q||)I_q
$$
当我们对这个滤波函数进行求导的时候发现，当两个像素点之间的灰度值的方差很大，说明是处在图像边缘的时候，高斯滤波核的值却很小，这就导致滤波之后的图像边缘和原图的边缘梯度不一致，虽然都能找出图像的边缘，但是双边滤波结束之后的图形边缘会出现灰度值过渡不自然的现象。

而导向滤波则不同，因为 
$$
q_i=aI_i+b
$$
如果输入图像和引导图像相同，那么每个窗口的a就是输入图像的方差比上方差加一个参数。当方差很大的时候，a就约等于1，相当于输出图像与输入图像的梯度比例关系不变。