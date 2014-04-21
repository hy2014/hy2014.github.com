---
layout: post
title: "hbase explain"
description: ""
category: 
tags: []
---
{% include JB/setup %}

华为的二级索引Hindex是个好东西，但是在开发的阶段，却遇到了一个问题。

	开发人员想知道自己的Filter到底触发还是没触发index。

就像我们使用MySQL的时候， 我们是可以使用explain这个命令清楚的知道，我们的SQL查询到底触发了那些index。
通过查看源码，我们发现ScanFilterEvaluator这个类，只有一个方法evaluate。 我们可以写一个Java类来模拟实现一个非常简单的explain功能。

首先我们需要获取查询Table的Descriptor, 这个Descriptory应该是IndexedHTableDescriptor的类型， 这段代码就不写了。

接下来我们的思路是本地创建一个HRegion，通过调用ScanFilterEvaluator.evaluate这个方法，模拟出来到底哪些index起作用了。
```java
private static HRegion initHRegion(byte[] tableName, byte[] startKey,
			byte[] stopKey, String callingMethod, Configuration conf,
			byte[]... families) throws IOException {
	HRegionInfo info = new HRegionInfo(tableDescriptor.getName(), startKey,
			stopKey, false);
	Path path = new Path(DIR + callingMethod);
	FileSystem fs = FileSystem.get(conf);
	if (fs.exists(path)) {
		if (!fs.delete(path, true)) {
			throw new IOException("Failed delete of " + path);
		}
	}
	return HRegion.createHRegion(info, path, conf, tableDescriptor);
}

static void getIndexRegionScanner(Filter filter) throws Exception {
	LOG.info("begin to explain filter....");
	ScanFilterEvaluator mapper = new ScanFilterEvaluator();
	// set local filesystem
	Scan scan = new Scan();
	scan.setFilter(filter);
	// Will throw null pointer here.
	List<IndexSpecification> indices = ((IndexedHTableDescriptor) tableDescriptor).getIndices();
	IndexManager.getInstance().addIndexForTable(tableDescriptor.getNameAsString(), indices);
	//这里的region是我们本地创建的region
	IndexRegionScanner indexRegionScanner = mapper.evaluate(scan, indices, new byte[0], region, tableDescriptor.getNameAsString());
	explainIndexRegionScanner(indexRegionScanner);
}

```
上文中， 如果和Build Filter的代码也省略了。 这里最终是获取到了IndexRegionScanner对象， 我们可以简单的认为这个对象就是存储的起作用的Index。 关于IndexRegionScanner的结构类型。 它有三个实现类， 分别是LeafIndexRegionScanner，IndexRegionScannerForAND，IndexRegionScannerForOR。
从名字可以看出来， 这是一个树状的结构型， 实际上IndexRegionScannerForAND，IndexRegionScannerForOR只是中间节点， 他们内部存储了IndexRegionScanner， 而LeafIndexRegionScanner里面才存储了一个起作用的Index。
```java
if (indexRegionScanner == null) {
	System.out.println("No index invoke");
	return;
}
if (indexRegionScanner instanceof IndexRegionScannerForAND) {
	System.out.println(explainIndexRegionScannerForAND((IndexRegionScannerForAND) indexRegionScanner));
} else if (indexRegionScanner instanceof IndexRegionScannerForOR) {
	System.out.println(explainIndexRegionScannerForOR((IndexRegionScannerForOR) indexRegionScanner));
} else if (indexRegionScanner instanceof LeafIndexRegionScanner) {
	System.out.println(explainIndexRegionScannerForLeaf((LeafIndexRegionScanner) indexRegionScanner));
}
```
实际上explainIndexRegionScannerForAND和explainIndexRegionScannerForOR是一个递归的过程， 可以看看explainIndexRegionScannerForLeaf的简单实现。
```java
static IndexSpecification readIndexFromLeafIndexRegionScanner(
			LeafIndexRegionScanner scanner) throws Exception {
	Field field = scanner.getClass().getDeclaredField("index");
	field.setAccessible(true);
	return (IndexSpecification) field.get(scanner);
}
```
这里使用了反射，将不可见的index取了出来。

本文介绍了如何实现简单的Hindex Explain， 该Explain的功能可能略简单了些， 有兴趣的同学可以在此基础上作一些拓展。