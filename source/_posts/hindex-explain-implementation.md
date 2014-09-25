title: hindex explain implementation
date: 2014-04-21 17:06:22
tags: hbase
category: hbase
description: "简单实现Hindex的explain功能"
---

华为的二级索引[Hindex][1]是个好东西，但是在开发的阶段，由于Hindex在客户端的绝对透明， 开发人员希望知道自己定义的Filter是否触发了index, 并且触发了那些index?

<!-- more -->

就像我们使用MySQL的时候， 我们是可以使用explain这个命令清楚的知道，我们的SQL查询到底触发了那些index。

通过查看Hindex源码，发现ScanFilterEvaluator这个类，只有一个方法evaluate。 我们可通过调通该方法explain客户端的Filter和Server端index的交互情况。

基本的思路是本地创建一个与Server段Table相同Schema的HRegion，通过调用ScanFilterEvaluator.evaluate这个方法，模拟出来到底哪些index起作用了。

下列代码描述了随机在本地创建一个HRegion对象。
```java
private static HRegion initHRegion(byte[] tableName, byte[] startKey,
			byte[] stopKey, String callingMethod, Configuration conf,
			byte[]... families) throws IOException {
	HRegionInfo info = new HRegionInfo(tableDescriptor.getName(), startKey, stopKey, false);
	Path path = new Path(DIR + callingMethod);
	FileSystem fs = FileSystem.get(conf);
	if (fs.exists(path)) {
		if (!fs.delete(path, true)) {
			throw new IOException("Failed delete of " + path);
		}
	}
	return HRegion.createHRegion(info, path, conf, tableDescriptor);
}
```

调用evaluate方法
```java
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

上文中，最终获取到了IndexRegionScanner对象， 这个对象存储了trigger的Index, 我们尝试与解析它。
下面是关于IndexRegionScanner结构的描述

	它有三个实现类， 分别是LeafIndexRegionScanner，IndexRegionScannerForAND，IndexRegionScannerForOR。从名字可以看出来， 这是一个树状的结构型， IndexRegionScannerForAND，IndexRegionScannerForOR只是中间节点，内部存储了IndexRegionScanner对象。而一个LeafIndexRegionScanner是叶子节点, 只存储了一个起作用的Index。


下面是解析代码:
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
explainIndexRegionScannerForAND和explainIndexRegionScannerForOR是一个不断递归的过程。

主要看explainIndexRegionScannerForLeaf的简单实现。
```java
static IndexSpecification readIndexFromLeafIndexRegionScanner(
			LeafIndexRegionScanner scanner) throws Exception {
	Field field = scanner.getClass().getDeclaredField("index");
	field.setAccessible(true);
	return (IndexSpecification) field.get(scanner);
}
```

通过Java的反射机制，将不可见的index成员变量取出来， 并且返回。

  [1]: https://github.com/Huawei-Hadoop/hindex