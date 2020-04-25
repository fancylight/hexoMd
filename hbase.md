---
title: hbase
date: 2019-07-17 10:24:25
tags: 分布式
categories: 数据库
---
### 概述
本文要解决的问题
- hbase概念
- hbase和传统RDBMS数据库相比较的不同
- hbase连接池问题
##### 基础概念
- rowkey: 通过该列来表示一个行数据
- column family:表示一个族
- qualifier:表示族中的一个key
- version:每一个value可以有不同的版本

例如:建立一张用户表,ID,姓,名,密码
|ID|姓|名|密码|时间戳|
|--|--|--|--|--|
|1|张|三|111|20160102|
|2|李|四|222|20130214|
<center>关系型数据库</center>

|Row-Key|Value(CF,Qualifier,Version)|
|--|--|
|1|info{'姓':'张','名':'三'} pwd:{'密码':111}|
|2|info{'姓':'李','名':'四'} pwd:{'密码':222}|
<center>hbase(逻辑上)</center>

- 解释: 实际上在物理磁盘上hbase并不是如此储存数据的,只是根据rowkey将数据组织在一起,这里info,pwd就是cf, '姓','名'就是qualifier

##### java api和hbase的交互
- 通过Htable来交互,实际上最终交互的对象是HConnection


{%codeblock lang:java HTable%}
 public HTable(Configuration conf, final byte[] tableName)
  throws IOException {
    this(conf, TableName.valueOf(tableName));
  }
    public HTable(Configuration conf, final TableName tableName)
  throws IOException {
    this.tableName = tableName;
    this.cleanupPoolOnClose = this.cleanupConnectionOnClose = true;
    if (conf == null) {
      this.connection = null;
      return;
    }
    this.connection = HConnectionManager.getConnection(conf);
    this.configuration = conf;

    this.pool = getDefaultExecutor(conf);
    this.finishSetup();
  }
  //交互方式,由HTable封装的函数就能够于数据库交互
   public Result[] get(List<Get> gets) throws IOException {
    if (gets.size() == 1) {
      return new Result[]{get(gets.get(0))};
    }
    try {
      Object [] r1 = batch((List)gets); //调用HConnect进行处理

      // translate.
      Result [] results = new Result[r1.length];
      int i=0;
      for (Object o : r1) {
        // batch ensures if there is a failure we get an exception instead
        results[i++] = (Result) o;
      }

      return results;
    } catch (InterruptedException e) {
      throw (InterruptedIOException)new InterruptedIOException().initCause(e);
    }
  }
{%endcodeblock%}
- Scan的作用,用来处理返回数据类型,处理的逻辑在迭代过程中
{%codeblock lang:java%}
//一个查询的例子
public List<JSONObject> getAllDataFromTable(String tableName) {
		List<JSONObject> dataJsonList = new ArrayList<JSONObject>();
		//HTableInterface是HTable的父类接口
		HTableInterface hTableInterface = tablePool.getTable(tableName); //这里引入了表池的概念,实际该类被废弃了
		ResultScanner rs = null;
		try {
			rs = hTableInterface.getScanner(new Scan());//获取Scanner
			for (Result result : rs) {//迭代过程中发生了连接并获取数据
				JSONObject dataJsonObject = new JSONObject();
				String rowkey = new String(result.getRow());
				dataJsonObject.put("rowkey", rowkey);
				for (KeyValue keyValue : result.raw()) {
					dataJsonObject.put(new String(keyValue.getQualifier()),
							new String(keyValue.getValue()));
				}
				dataJsonList.add(dataJsonObject);
			}
		} catch (IOException e) {
			//log.error("getAllDataFromTable() IOException :: ", e);
			System.out.println("getAllDataFromTable() IOException");
		} finally {
			rs.close();//
		}
		return dataJsonList;
	}
{%endcodeblock%}