# 如何快速计算某个 10000000(一千万)以内的质数是第几个位置上

---
> https://stackoverflow.com/questions/14126959/find-position-of-prime-number

首先让我们先来计算出那些少于 10000000 的平凡根的质数, 这些质数会用于建立函数 pi(n)的table的值,以及执行计算并筛选出问题的答案.
一千万的平方根是 `3162.3`. 我们不会将`2`纳入筛选质数 -- 我们会只筛选奇数, 而2是一个特例, 会另一对待 -- 但是我们需要下一个质数
会大于 一千万的平方根, 所以, 用于筛选的质数其实是永不满足的 (这在将来也许会导致某个问题). 我们将使用下面这个非常简单版本的 `Sieve of Eratosthenes`
来计算筛选质数:

```python
    def primes(n):
        b,  p,  ps  =   [True]  *   (n+1),  2,  []
        for p   in  xrange(2,   n+1):
            if  b[p]:
                ps.append(p)
                for i   in  xrange(p,   n+1,    p):
                    b[i]    =   False
        return  ps
```
`Sieve of Eratosthenes` 分为两步. 第一步, 创建一个以2为起点, 到 目标值的数集.接着,重复遍历该集合, 从第一个未交际的数开始, 直到遍历完
所有的数. 一开始, 2 是第一个未交集的数, 这就会拿到 `4, 6, 8, 10...`. 接着是3, 拿到 `6, 9, 12, 15...`. 接着是4, 拿到的是 2 的两倍的数集, 
下一个数是5, 拿到 `10, 15, 20, 25...`. 继续直到所有没有交集的数都被处理掉; 所有剩下没有被交集到的数字就都是 **质数**, 以 `p`为待处理数,
如果他是不可交集的数, 循环i就会输出它的倍数.

`primes`函数会返回一个含有 447 个质数的 集合: 2, 3, 5, 7, 11, 13, ..., 3121, 3137, 3163. 我们将2从集合中取出, 将剩下的 446 个筛选质数
赋给全局变量 `ps`.
```python
    ps  =   primes(3163)[1:]
```
主函数是我们用来计算在某个范围内质数的个数. 为了避免每次执行这个计数函数时重复计算, 我们会将使用的筛选集合也赋给到一个全局的集合变量中:
```python
    sieve = [True] * 500
```

`count` 函数使用特殊的`a segmented Sieve of Eratosthenes`来计算一段从低到高(低点和高点也在范围内)的范围内的质数个数. 这个 `count` 函数中
有4 个`for` 循环: 第一个会清空 `sieve`, 最后一个计算质数的个数, 其他两个则用来执行筛选操作, 这两个操作的简化版本在上面已经呈现过:

```python
    def count(lo, hi):
        for i in xrange(500):
            sieve[i] = True
        for p in ps:
            if p* p > hi: break
            q = (lo + p + 1) / -2 % p
            if lo + q + q + 1 < p * p: q += p
            for j in xrange(q, 500, p):
                sieve[j] = False
        k = 0
        for i in xrange((hi - lo) // 2) :
            if sieve[i]: k += 1
        return k
```

这个函数的核心就在于`for p in ps` 这个执行筛选的循环, 轮询处理每一个筛选后的质数`p`.当 p 的平方大于要求的范围时就会跳出整个循环, 因为所有的质数都会
这样处理(我们要求收集的质数大于一千万的平方根, 这样可以保证整个循环会被中止). 神秘的变量`q`是当前筛选数`p`在范围[lo-hi]中的最小倍数的位置. 下一个
`if` 会当 `q` 是平方的根时候增加`q`的值. 接着的循环会将`q`的倍数从`seive` 中移除.

`count`函数会被用于两个地方. 第一个是用于创建当`pi(n)` n 为 1000 时的一个 `table`;第二个是用来 `interpolates within the table`. 这个
table放入 全局的 `piTable`:
```python
    piTable = [0] * 10000
```

我们选择的参数`1000` 和`10000`是基于初始的诉求, 这会保持内存的用量在 50kb 以内. (是的, 我知道原po主 放宽了这个要求. 但是这个我们可以做的更好).
 一万个 32-bit 的数字需要 40000 bytes来保存, 筛选一段范围在 1000 以内的数字将会只需要 500 bytes, 并且会运行的非常的快. 你也许会想试一试其他
 的参数来看看程序究竟会怎样影响空间和时间. 创建 `piTable`只需要执行 `count` 一万次:
 ```python
    for i in xrange(1, 10000):
        piTable[i] = piTable[i-1] + \
            count(1000 * (i - 1), 1000 * i)
 ```
这部分所有的计算都可以在编译时期完成, 避免 执行时期的损耗. 当我在 [ideone.com](ideone.com)执行这些计算的时候, 这花了大概5秒钟的时间, 但是这个
时间不需要计算, 因为当程序员写完这些代码一次执行后, 可以重复使用的. 通用的规则是, 你应该寻找这些将代码从执行期移动的编译器的机会, 这可以大幅提高你的
程序的执行速度.

现在还剩下的就是写出计算小于等于n时有多少个质数:
