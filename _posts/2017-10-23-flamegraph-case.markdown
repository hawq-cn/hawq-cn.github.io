---
layout: post
title:  "一个火焰图case"
author: interma
date:   2017-10-23
published: true
---

### 背景
使用火焰图对hawq的TPCH-Q6进行性能分析，发现了一个奇怪的平顶山，打算解决之。

### 追查过程
发现一个疑点：ParquetColumnReader_readValue() 这个函数非常简单：
```
void
ParquetColumnReader_readValue(
		ParquetColumnReader *columnReader,
		Datum *value,
		bool *null,
		int hawqTypeID)
{
	/*
	 * for non-repeatable column, because of using shared `pageBuffer`,
	 * consume should be called after last value is truely returned.
	 */
	if (columnReader->columnMetadata->r == 0)
	{
		consume(columnReader);
	}

	/* current definition level is used to determine null value */
	if (CurrentDefinitionLevel(columnReader) < columnReader->columnMetadata->d)
	{
		*null = true;
	}
	else
	{
		*null = false;

		if (hawqTypeID == HAWQ_TYPE_BOOL)
		{
			*value = BoolGetDatum((bool) BitPack_ReadInt(columnReader->currentPage->bool_values_reader));
		}
		else
		{
			decodePlain(value, &(columnReader->currentPage->values_buffer), hawqTypeID);
		}
	}
	
	/*
	 * for repeatable column, need to preread next value's r
	 */
	if (columnReader->columnMetadata->r > 0)
	{
		consume(columnReader);
	}
}
```

预期consume()和decodePlain()这2个函数会占用大量时间（它们都只是解析内存），但是为何未占满全部时间，出现一个『差』

![image]({{ site.url }}/assets/images/q6-1.png)

然后又做了几次实验：
1. inline consume()

![image]({{ site.url }}/assets/images/q6-2.png)

2. 去掉执行不到的各个if条件，修改函数内容为：
```
void
ParquetColumnReader_readValue(
		ParquetColumnReader *columnReader,
		Datum *value,
		bool *null,
		int hawqTypeID)
{
		consume(columnReader);
		decodePlain(value, &(columnReader->currentPage->values_buffer), hawqTypeID);
}
```

![image]({{ site.url }}/assets/images/q6-3.png)

3. 数据：10.27%->9.06%->6.89%

### 结论
* 调用多的函数会放大简单操作的时间消耗=>产生差
* 这里这个函数没有明显问题（或简单就能改善的手段），如果需要优化，需要深入到指令级别而非算法级别

### study
* 牢记火焰图中基于采样，执行多or执行慢是都可能的
* 如果执行多（本case执行2000余万次），简单操作（函数调用，简单赋值等）也会导致大量时间消耗
* 出现时间差，可以对单个函数加计时器打日志

