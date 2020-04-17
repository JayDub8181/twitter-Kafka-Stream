# twitterKafkaStream

Setup Twitter client producing to Kafka Stream and written to Elasticsearch.

## Twitter Producer Class

### 1. Install Maven Dependencies (libraries required to compile code)
In `pom.xml`, add Maven dependencies:

_*Code is current as of April 13, 2020. For latest, click links._

1.1 Go to [Kafka Client](http://www.dropwizard.io/1.0.2/docs/) for latest Maven dependency code. Should look like this:
```
 <dependency>
            <groupId>org.apache.kafka</groupId>
            <artifactId>kafka-clients</artifactId>
            <version>2.4.1</version>
 </dependency>
```
1.2 Go to [slf4j (logger)](https://mvnrepository.com/artifact/com.walterjwhite.java.dependencies/slf4j-simple) for latest Maven dependency code. Should look like this:
```
<dependency>
            <groupId>com.walterjwhite.java.dependencies</groupId>
            <artifactId>slf4j-simple</artifactId>
            <version>0.0.17</version>
            <type>pom</type>
            <scope>test</scope>
</dependency>
```
1.3 Go to [twitter/hbc](https://github.com/twitter/hbc)
```
<dependency>
            <groupId>com.twitter</groupId>
            <artifactId>hbc-core</artifactId> <!-- or hbc-twitter4j -->
            <version>2.2.0</version> <!-- or whatever the latest version is -->
</dependency>
```
### 2. Create Twitter Client Function
```
public Client createTwitterClient(BlockingQueue<String> msgQueue){
}
```
**Include:**

2.1 Declare the host you want to connect to, the endpoint, and authentication (basic auth or oauth)
* Sets up a stream host
* Calls out variable by which endpoint will filter. Variable to be defined in Step 2.2.
* Authentication method will be OAuth for this example
```
Hosts hosebirdHosts = new HttpHosts(Constants.STREAM_HOST);
StatusesFilterEndpoint hosebirdEndpoint = new StatusesFilterEndpoint();
Authentication hosebirdAuth = new OAuth1(consumerKey, consumerSecret, token, secrets);
```
2.2 Setup Twitter Track Terms (which tweets will be pulled from API)
* Endpoint in 2.1 will pull all tweets with terms "kafka" in them. You may want to put something more frequently appearing like "Trump".
```
List<String> terms = Lists.newArrayList("kafka");
        hosebirdEndpoint.trackTerms(terms);
```
2.3 Setup client builder
* Creates client that connects to stream host in 2.1 line 1
* Use parameters set in  2.1 line 3 to authenticate
* Use status filter for endpoint. See 2.1 line 2
* Use String delimited processor to send strings to message queue built in 2.1

```
  ClientBuilder builder = new ClientBuilder()
                .name("Hosebird-Client-01")                              
                .hosts(hosebirdHosts)
                .authentication(hosebirdAuth)
                .endpoint(hosebirdEndpoint)
                .processor(new StringDelimitedProcessor(msgQueue));                         
```
2.4 Build client
```
Client hosebirdClient = builder.build();
        return hosebirdClient;
```

### 3. Run Method
3.1 Set up your message queues.
Parameter for LinkedBlockingQueue is max capacity for messages in queue at a time.
```
BlockingQueue<String> msgQueue = new LinkedBlockingQueue<String>(1000);
```
3.2 Create twitter client that outputs to your message queue and establish connection
```
Client client = createTwitterClient(msgQueue);
client.connect();
```
3.3 Create a loop that continually will send tweets to kafka based on terms
* There is also a try catch loop inserted in here to print any errors
* as well as our slf4j logger printing out messages being sent.
```
while (!client.isDone()) {
            String msg=null;
            try {
                msg = msgQueue.poll(5, TimeUnit.SECONDS);
            } catch (InterruptedException e){
                e.printStackTrace();
                client.stop();
            }
            if (msg!=null){
                logger.info(msg);
            }
        }
        logger.info("End of application");
```

### 4. Kafka Producer
4.1 Create Kafka Producer Function with minimum Properties
* Declare server IP to bootstrap (I used local host default)
* Set properties to point to this IP
* Set properties to convert key into a string
* Set properties to convert values into strings

*Actually create and return new producer
```
public KafkaProducer<String,String> createKafkaProducer(){
        String boostrapServers = "127.0.0.1:9092";

        //create Producer properties
        Properties properties = new Properties();
        properties.setProperty(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, boostrapServers);
        properties.setProperty(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        properties.setProperty(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());

        //create producer
        KafkaProducer<String,String> producer = new KafkaProducer<String, String>(properties);
        return producer;
```
4.2 Add create prdocuer method into existing run function
```
        KafkaProducer<String,String> producer = createKafkaProducer();
```
4.3 Also within run function, we are going to add a producer method with perameters of topic to send to, keys, and what to send. In addition, will add a new Callback function that essentially pipes out an error exception message should there be a problem
* You already have the top 2 lines from step 3.3; this is adding the remainder.
* The topic you input (in this example "twitter_tweets" must already be created"). You can do this via command line.
```
        if (msg!=null){
                logger.info(msg);
                producer.send(new ProducerRecord<>("twitter_tweets", null, msg), new Callback() {
                    @Override
                    public void onCompletion(RecordMetadata recordMetadata, Exception e) {
                        if (e != null){
                    logger.error("Something bad happened", e);
                        }
                    }
                });
            }
```
### 5. Declare Local Variables for
5.1 API keys and secrets
* Fill in per your Twitter Developer account
```
    String consumerKey=" ";
    String consumerSecret=" ";
    String token=" ";
    String secret=" ";
```
5.2 Declare logger variable
```
     Logger logger= LoggerFactory.getLogger(twitterProducer.class.getName());
```



### Setup Elasticsearch (WIP)




## Authors

* **John W**


## Acknowledgments

* https://www.linkedin.com/learning/learn-apache-kafka-for-beginners/intro-to-apache-kafka
