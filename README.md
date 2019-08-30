# SpringBoot-Kafka-Storm
SpringBoot+Kafka+Storm
---
通过SpringBoot、Kafkah和Storm的整合，实现了项目KafkaProducerStorm中请求数据库的数据并将数据发送到Kafka,在项目KafkaConsumerStorm中消费Kafka的数据并将消费掉的数据传给Storm进行判断，然后将符合条件的数据信息插入到数据库(项目源码点击[这里](https://github.com/butalways1121/SpringBoot-Kafka-Storm)）。
<!-- more -->
### KafkaProducerStorm项目
该项目主要是向Kafka发送数据。
**1.KafkaProducerUtil类**
在项目中创建了KafkaProducerUtil类，用sendMessage(String msg,String url,String topicName)以向Kafka发送数据，其中kafka的相关配置都放在application.properties中，详细代码如下：
```bash
package com.KafakaProducer.producer;

import java.util.Properties;

import org.apache.kafka.clients.producer.ProducerRecord;
import org.apache.kafka.common.serialization.StringSerializer;
import org.springframework.stereotype.Component;
import org.apache.kafka.clients.producer.KafkaProducer;


@Component
public final class KafkaProducerUtil {
	//向kafka发送单条消息
	public static boolean sendMessage(String msg,String url,String topicName) {
		KafkaProducer<String, String> producer=null;
		boolean falg=false;
		try{
			Properties props=init(url);
			producer= new KafkaProducer<String, String>(props);
			producer.send(new ProducerRecord<String, String>(topicName,msg));
			falg=true;
		}catch(Exception e){
			e.printStackTrace();
		}finally{
			producer.close();
		}
		return falg;
	}

	//初始化配置
	private static Properties init(String url){
		Properties props = new Properties();
		props.put("bootstrap.servers", url);
		//acks=0：如果设置为0，生产者不会等待kafka的响应。
		//acks=1：这个配置意味着kafka会把这条消息写到本地日志文件中，但是不会等待集群中其他机器的成功响应。
		//acks=all：这个配置意味着leader会等待所有的follower同步完成。这个确保消息不会丢失，除非kafka集群中所有机器挂掉。这是最强的可用性保证。
		props.put("acks", "all");
		//配置为大于0的值的话，客户端会在消息发送失败时重新发送。
		props.put("retries", 0);
		//当多条消息需要发送到同一个分区时，生产者会尝试合并网络请求。这会提高client和生产者的效率
		props.put("batch.size", 16384);
		props.put("key.serializer", StringSerializer.class.getName());
		props.put("value.serializer", StringSerializer.class.getName());
		return props;
	}
}
```
**2.InfoController类**
在controller类里面主要负责处理请求返回数据，有两种方法，一是通过id查询到一条数据将该条数据发往Kafka，二是查询数据库的所有数据，然后将拿到的列表形式数据逐一的发往Kafka。在发往Kafka时需要将拿到的数据通过实体类里面定义的toString方法转换成字符串，如下：
```bash
@Override
	public String toString() {
		return JSON.toJSONString(this);
	}
```
另外，该toString方法会对数据的内容按照字母顺序排序，可以在实体类中通过注解@JSONType(orders={"id","name","age"})来按指定顺序排序，InfoController类的代码如下：
```bash
package com.KafakaProducer.controller;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

import com.KafakaProducer.config.ApplicationConfiguration;
import com.KafakaProducer.dao.InfoDao;
import com.KafakaProducer.entity.Info;
import com.KafakaProducer.producer.KafkaProducerUtil;
import com.KafakaProducer.service.InfoService;
import java.io.IOException;
import java.util.List;

@RestController
@RequestMapping(value = "/api")
public class InfoController {
	@Autowired
	private InfoService infoService;
	@Autowired
	private InfoDao infoDao;
	@Autowired
	ApplicationConfiguration app;

	//根据id查询信息
	@RequestMapping(value = "/findById",method = RequestMethod.GET)
	public Info findById(@RequestParam(value = "id",required = true) int id) throws IOException{
		System.out.println("开始根据id查询信息！");
		Info result = infoDao.findById(id);
		KafkaProducerUtil.sendMessage(result.toString(), app.getServers(), app.getTopicName());
		return infoService.findById(id);
	}
	
	//查询所有信息
	@PostMapping(value = "/findAll")
	public List<Info> findAll() {
		List<Info> result = infoService.findAll();//列表形式
		//逐条向Kafka发送消息
		for(int i=0;i<result.size();i++) {//.size()就是获取到ArrayList中存储的对象的个数
			//.get(i)当i=0时，取得是list集合中第一个元素
			KafkaProducerUtil.sendMessage(result.get(i).toString(), app.getServers(), app.getTopicName());
		}
		return infoService.findAll();
	}
}
```

### KafkaConsumerStorm项目
该项目用来消费Kafka的数据，并将消费掉的数据通过Storm处理后再插入到数据库。在项目中创建storm包，其下创建Spout、Bolt及Topology类来实现Storm对数据的处理。
**1.Spout类**
Spout是Storm获取数据的一个组件，主要实现nextTuple方法，在其中添加从Kafka消费获取数据的代码，就能将数据进一步交给Bolt进行处理，代码如下：
```bash
package com.KafkaConsumer.storm;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;
import java.util.Map;
import java.util.Properties;
import java.util.concurrent.TimeUnit;

import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.storm.spout.SpoutOutputCollector;
import org.apache.storm.task.TopologyContext;
import org.apache.storm.topology.OutputFieldsDeclarer;
import org.apache.storm.topology.base.BaseRichSpout;
import org.apache.storm.tuple.Fields;
import org.apache.storm.tuple.Values;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import com.KafkaConsumer.config.ApplicationConfiguration;
import com.KafkaConsumer.constant.Constants;
import com.KafkaConsumer.entity.InfoGet;
import com.KafkaConsumer.util.GetSpringBean;
import com.alibaba.fastjson.JSON;


public class Spout extends BaseRichSpout{

	private static final long serialVersionUID = -2548451744178936478L;
	private static final Logger logger = LoggerFactory.getLogger(Spout.class);
	private SpoutOutputCollector collector;
	private KafkaConsumer<String, String> consumer;
	private ConsumerRecords<String, String> msgList;
	private ApplicationConfiguration app;
	
	@SuppressWarnings("rawtypes")
	@Override
	public void open(Map map, TopologyContext arg1, SpoutOutputCollector collector) {
		app=GetSpringBean.getBean(ApplicationConfiguration.class);
		kafkaInit();
		this.collector = collector;
	}
	
	@Override
	public void nextTuple() {
		for (;;) {
			try {
				msgList = consumer.poll(100);
				if (null != msgList && !msgList.isEmpty()) {
					String msg = "";
					List<InfoGet> list=new ArrayList<InfoGet>();
					for (ConsumerRecord<String, String> record : msgList) {
						// 原始数据
						msg = record.value();
						if (null == msg || "".equals(msg.trim())) {
							continue;
						}
						try{
							list.add(JSON.parseObject(msg, InfoGet.class));
						}catch(Exception e){
							logger.error("数据格式不符!数据:{}",msg);
							continue;
						}
				     } 
					logger.info("Spout发射的数据:"+list);
					//发送到bolt中
					this.collector.emit(new Values(JSON.toJSONString(list)));
					 consumer.commitAsync();
				}else{
					TimeUnit.SECONDS.sleep(3);
					logger.info("未拉取到数据...");
				}
			} catch (Exception e) {
				logger.error("消息队列处理异常!", e);
				try {
					TimeUnit.SECONDS.sleep(10);
				} catch (InterruptedException e1) {
					logger.error("暂停失败!",e1);
				}
			}
		}
	}
	
	
	@Override
	public void declareOutputFields(OutputFieldsDeclarer declarer) {
		declarer.declare(new Fields(Constants.FIELD));
	}
	
	/**
	 * 初始化kafka配置
	 */
	private void kafkaInit(){
		Properties props = new Properties();
        props.put("bootstrap.servers", app.getServers());  
        props.put("max.poll.records", app.getMaxPollRecords());
        props.put("enable.auto.commit", app.getAutoCommit());
        props.put("group.id", app.getGroupId());
        props.put("auto.offset.reset", app.getCommitRule());
        props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        consumer = new KafkaConsumer<String, String>(props);
        String topic=app.getTopicName();
    	this.consumer.subscribe(Arrays.asList(topic));
    	logger.info("消息队列[" + topic + "] 开始初始化...");
	}
}

```
**2.Bolt类**
Bolt类主要是对数据进行处理，对数据做一个简单的判断，然后将符合条件的插入到数据库，Bolt类主要处理业务逻辑的方法是execute，主要实现的方法也是写在其中，这里主要是对数据的age属性做一判断，如果age>10,则可以进一步插入数据库，否则丢弃，代码如下：
```bash
package com.KafkaConsumer.storm;

import java.util.Iterator;
import java.util.List;
import java.util.Map;

import org.apache.storm.task.OutputCollector;
import org.apache.storm.task.TopologyContext;
import org.apache.storm.topology.OutputFieldsDeclarer;
import org.apache.storm.topology.base.BaseRichBolt;
import org.apache.storm.tuple.Tuple;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import com.KafkaConsumer.constant.Constants;
import com.KafkaConsumer.entity.InfoGet;
import com.KafkaConsumer.service.InfoGetService;
import com.KafkaConsumer.util.GetSpringBean;
import com.alibaba.fastjson.JSON;


public class Bolt extends BaseRichBolt{

		private static final long serialVersionUID = 6542256546124282695L;
		private static final Logger logger = LoggerFactory.getLogger(Bolt.class);
		private InfoGetService infoGetService;
		
		@SuppressWarnings("rawtypes")
		@Override
		public void prepare(Map map, TopologyContext arg1, OutputCollector collector) {
			infoGetService=GetSpringBean.getBean(InfoGetService.class);
		}
	  
		@Override
		public void execute(Tuple tuple) {
			String msg=tuple.getStringByField(Constants.FIELD);
			try{
				List<InfoGet> listUser =JSON.parseArray(msg,InfoGet.class);
				//移除age小于10的数据
				if(listUser!=null&&listUser.size()>0){
					Iterator<InfoGet> iterator = listUser.iterator();
					 while (iterator.hasNext()) {
						 InfoGet user = iterator.next();
						 if (user.getAge()<10) {
							 logger.warn("Bolt移除的数据:{}",user);
							 iterator.remove();
						 }
					 }
					if(listUser!=null&&listUser.size()>0){
						infoGetService.insert(listUser);
					}
				}
			}catch(Exception e){
				logger.error("Bolt的数据处理失败!数据:{}",msg,e);
			}
		}


		@Override
		public void cleanup() {
		}

		@Override
		public void declareOutputFields(OutputFieldsDeclarer arg0) {
				
		}
		
	
}

```
**3.Topology类**
Topology类是Storm的主类，主要是对Topology(拓步)进行提交，在其中需要对spout和bolt进行相应的设置，如下：
```bash
package com.KafkaConsumer.storm;

import org.apache.storm.Config;
import org.apache.storm.LocalCluster;
import org.apache.storm.StormSubmitter;
import org.apache.storm.topology.TopologyBuilder;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;

import com.KafkaConsumer.constant.Constants;

@Component
public class Topology {
	private  final Logger logger = LoggerFactory.getLogger(Topology.class);

	public  void runStorm(String[] args) {
		// 定义一个拓扑
		TopologyBuilder builder = new TopologyBuilder();
		// 设置1个Executeor(线程)，默认一个
		builder.setSpout(Constants.KAFKA_SPOUT, new Spout(), 1);
		// shuffleGrouping:表示是随机分组
		// 设置1个Executeor(线程)，和两个task
		builder.setBolt(Constants.INSERT_BOLT, new Bolt(), 1).setNumTasks(1).shuffleGrouping(Constants.KAFKA_SPOUT);
		Config conf = new Config();
		//设置一个应答者
		conf.setNumAckers(1);
		//设置一个work
		conf.setNumWorkers(1);
		try {
			// 有参数时，表示向集群提交作业，并把第一个参数当做topology名称
			// 没有参数时，本地提交
			if (args != null && args.length > 0) { 
				logger.info("运行远程模式");
				StormSubmitter.submitTopology(args[0], conf, builder.createTopology());
			} else {
				// 启动本地模式
				logger.info("运行本地模式");
				LocalCluster cluster = new LocalCluster();
				cluster.submitTopology("Topology", conf, builder.createTopology());
			}
		} catch (Exception e) {
			logger.error("storm启动失败!程序退出!",e);
			System.exit(1);
		}
		logger.info("storm启动成功...");
	}
}
```
**4.App.java**
该类是SpringBoot启动的主类，在其中添加Storm的Topology，Storm会随着SpringBoot启动而启动，如下：
```bash
package com.KafkaConsumer;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.ConfigurableApplicationContext;

import com.KafkaConsumer.storm.Topology;
import com.KafkaConsumer.util.GetSpringBean;


/**
 * Hello world!
 *
 */
@SpringBootApplication
public class App 
{
	public static void main(String[] args) {
		// 启动嵌入式的 Tomcat 并初始化 Spring 环境及其各 Spring 组件
		ConfigurableApplicationContext context = SpringApplication.run(App.class, args);
		GetSpringBean springBean=new GetSpringBean();
		springBean.setApplicationContext(context);
		Topology app = context.getBean(Topology.class);
		app.runStorm(args);
	}
}
```

### 测试
1. 启动KafkaProducerStorm项目，测试findAll方法，在postman浏览器中输入请求，send之后，会将数据库的所有数据返回，如下:
![](https://raw.githubusercontent.com/butalways1121/img-Blog/master/51.png)
打开Kafka Tool,在对应的topic下，可看到数据已写入Kafka:
![](https://raw.githubusercontent.com/butalways1121/img-Blog/master/52.png)
2. 启动KafkaConsumerStorm项目，控制台会显示如下处理结果：


Spout发射的数据:[{"age":11,"id":1,"name":"aa"}, {"age":22,"id":2,"name":"bb"}, {"age":33,"id":3,"name":"cc"}, {"age":4,"id":4,"name":"dd"}, {"age":8,"id":5,"name":"ee"}]
Bolt移除的数据:{"age":4,"id":4,"name":"dd"}
Bolt移除的数据:{"age":8,"id":5,"name":"ee"}
数据写入成功！


同时到对应数据库查看，发现符合age>10的数据已写入：
![](https://raw.githubusercontent.com/butalways1121/img-Blog/master/53.png)

***
## 至此，OVER！
