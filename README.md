# fluentd-docker-input-kafka
Example of Fluentd deployed as Docker container and the Kafka input plugin installed

We are using here the [Fluentd input Kafka plugin](https://github.com/fluent/fluent-plugin-kafka) and the [official
 Fluentd Docker image](https://hub.docker.com/r/fluent/fluentd/)
 
As output, we will just use the standard output.

# Content of the repo
- `Dockerfile`: Contains the base Alpine Fluentd Docker image, the installation of the Fluentd Input Kafka plugin and 
we add our custom `fluentd.conf` file, as is described in the 
[official documentation](https://hub.docker.com/r/fluent/fluentd/)
- `fluentd.conf`: The custom configuration file for Fluentd, where we have set the connection parameters and the topics
for the Kafka
- `entrypoint.sh`: Just the bash script to run our custom Fluentd Docker image. We took this from the 
[official documentation](https://hub.docker.com/r/fluent/fluentd/) too

# Configuring fluentd
In the `fluentd.conf` file, you can see that we are using, as input, the kafka plugin (type).

The most important part here is that, the topic name/s should match with the name set in the `<match>` tag. In this
example, the topic name is `mytopic`, so, in the match tag you see `<match mytopic>`

# Steps

##1. Run Kafka
We will use the Kafka version 2.11. Please download it from https://kafka.apache.org/ and follow the instructions of
the [quickstart](https://kafka.apache.org/quickstart) until the step of send some messages. Let's do it here anyway.

Move your shell to the directory where you downloaded and unziped the Kafka.

First, let's run the zookeeper 
```bash
bin/zookeeper-server-start.sh config/zookeeper.properties
```
In another shell, let's start the Kafka broker
```bash
bin/kafka-server-start.sh config/server.properties
```
Finally, let's create the topic `mytopic` where we will send message and from the Fluentd will take them.
```bash
bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic mytopic
```
By the moment, we are good with Kafka, let's continue with the Fluentd.

##2. Run Fluentd
We will use Fluentd version 1.7

Move your shell where you have clone`d this repository.

First, please edit the file `fluentd.conf` and replace `<yourlocalip>` by your IP.

Done that, let's create our custom Docker image based on Fluentd v1.7
```bash
docker build . -t my-custom-fluentd-kafka
```

Then, we will need a place to put the logs from the Fluentd
```bash
mkdir log && chmod 777 log && chmod 777 entrypoint.sh
```

Now, we are ready to run the Docker container from our custom Docker image
```bash
docker run -it --rm --name custom-docker-fluent-kafka -v $(pwd)/log:/fluentd/log my-custom-fluentd-kafka:latest
```
And you should see an output like
```bash
2020-03-18 17:26:34 +0000 [info]: parsing config file is succeeded path="/fluentd/etc/fluent.conf"
2020-03-18 17:26:34 +0000 [info]: using configuration file: <ROOT>
  <source>
    @type kafka
    brokers "here-your-ip:9092"
    topics "mytopic"
    format "text"
  </source>
  <match mytopic>
    @type stdout
  </match>
</ROOT>
2020-03-18 17:26:34 +0000 [info]: starting fluentd-1.7.4 pid=6 ruby="2.5.7"
2020-03-18 17:26:34 +0000 [info]: spawn command to main:  cmdline=["/usr/bin/ruby", "-Eascii-8bit:ascii-8bit", "/usr/bin/fluentd", "-c", "/fluentd/etc/fluent.conf", "-p", "/fluentd/plugins", "--under-supervisor"]
2020-03-18 17:26:35 +0000 [info]: gem 'fluent-plugin-kafka' version '0.13.0'
2020-03-18 17:26:35 +0000 [info]: gem 'fluentd' version '1.7.4'
2020-03-18 17:26:35 +0000 [info]: adding match pattern="mytopic" type="stdout"
2020-03-18 17:26:35 +0000 [info]: adding source type="kafka"
2020-03-18 17:26:35 +0000 [info]: #0 starting fluentd worker pid=14 ppid=6 worker=0
2020-03-18 17:26:35 +0000 [info]: #0 fluentd worker is now running worker=0
```
This is good, and it will stay there until we send some messages to Kafka. Let's do it then!

##3. Verify it's working!
Now, the expected behavior is that:
1. We send messages to Kafka and
2. We can see them in the container standard output

So, let's open a new shell from the directory where you downloaded and unziped the Kafka and run
```bash
bin/kafka-console-producer.sh --broker-list localhost:9092 --topic mytopic
```
The prompt will keep open for you to type plain messages, let's send something
```bash
>hello
>world
```
And you should see, in the standard output of our container, the messages from Kafka
```bash
...
2020-03-18 17:26:35 +0000 [info]: #0 starting fluentd worker pid=14 ppid=6 worker=0
2020-03-18 17:26:35 +0000 [info]: #0 fluentd worker is now running worker=0
2020-03-18 17:26:40.106900730 +0000 mytopic: {"message":"hello"}
2020-03-18 17:26:42.966026987 +0000 mytopic: {"message":"world"}
```
**Magic!**
# Conclusions
- The tricky part here is just the Fluentd configuration
- The custom docker image is quite thin ~50MB, which is good