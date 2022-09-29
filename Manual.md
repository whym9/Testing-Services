# PCAP processing service

Consists of services: receiving_service, pcap_statistics, saving_service

## Receiving Service (receiving_service)

Receiving Service uses HTTP server to accept files and a GRPC client to send them over to Pcap Statistics. HTTP server only accepts POST requests with form name "uploadFile" and value as a file. Since the system as a whole is made for working with pcap files, you need to send only pcap files.
It needs HTTP, GRPC addresses and adress for the Prometheus metrics. For that it needs environmetnal varibales, instruction for which are given in a section below. 

## Pcap Statistics (pcap_statistics)

Pcap Statistics accepts pcap files by a GRPC server and sends them to Saving Service by a GRPC client. Before sending pcap files further it needs to process it and make statistics about: TCP, UDP, IPv4, IPv6 protocols. 
GRPC server of this service needs to get the same address as a GRPC client of previous service. GRPC client and address for metrics should be different. We provide those addresses through environmental variables.

## Saving Service (saving_service)

Saving Service accepts the data through GRPC server and saves them in the file system and the Database. It gets data from Pcap Statistics, which are statistics and the files itself (exactly in this order). It saves data into MySQL db.
It also needs GRPC server address (same as Pcap Statistics GRPC client) and Metrics address from environmental variables. Additionally, a directory and a mysql dsn for saving data.

## Step-by-step Instructions

Step 1. Cloning repositories

Clone files in your preferred local repository by entering this commands in Terminal:

```
...:~$ git clone https://github.com/whym9/receiving_service.git

...:~$ git clone https://github.com/whym9/pcap_statistics.git

...:~$ git clone https://github.com/whym9/saving_service.git

```

Step 2. Building Docker Images

Start building docker images subsequentially. Go to each services' repository to locate the Dockerfile and run it: 

```
...:~$ cd receiving_service

.../receiving_service:~$ sudo docker build -t receiver . 

.../receiving_service:~$ cd ../pcap_statistics

.../pcap_statistics:~$ sudo docker build -t statistics . 

.../pcap_statistics:~$ cd ../saving_service

.../saving_service:~$ sudo docker build -t saver . 
```

Step 3. Adding environment variables

In this repository there are .env files that have environment vatiables listed there. Download them and place in the repository you use.

Step 4. Setting up MySQL container

This service uses MySQL server to save data into database. Therefore, we will use the MySQL image and run it and use it as the db server for our service to use. Execute following commands:

```
...:-$ sudo docker run -e MYSQL_ROOT_PASSWORD=123 --name=mys -p 127.0.0.1:3307:3306 -d mysql
```
It will pull mysql for you if you don't have it locally. parameter --name names the container so we can use that name later to refer to the container. 
Then you need to create a database and table in it. 

```
...:-$ sudo docker exec -it mys mysql -uroot -p123
```
This command will open mysql terminal in it we need to write:

```
mysql> CREATE DATABASE pcap_files;

mysql> USE pcap_files;

mysql> CREATE TABLE File_Statistics (
    -> ID INT NOT NULL,
    -> FilePath VARCHAR(256),
    -> ProtocolTCP INT,
    -> UDP INT,
    -> IPv4 INT,
    -> IPv6 INT,
    -> PRIMARY KEY (ID)
    -> );
```



Step 5. Running docker images as containers 

We use the command -pd to run the image in a detached mode and give it some parameters of host_port:docker_port to connect your host port to docker's. --env-file parameter tells the container to take as environment the .env file. Go to the inital directory and run:

```
...:~$ cd receiving_service

.../receiving_service:~$ sudo docker run --name=rec --env-file .env -pd 8080:8080 receiver 

.../receiving_service:~$ cd ../pcap_statistics

.../pcap_statistics:~ sudo docker run --name=stat --env-file .env -pd 6006:6006 receiver

.../pcap_statistics:~$ cd ../saving_service

.../saving_service:~$ sudo docker run --name=save --env-file .env -pd 5005:5005 receiver
```

8080:8080 part means that the host port 8080 should be connected to docker port 8080.

Step 6. Testing

To test it you can use localhost with different ports for addresses.

Now all services are running in a detached mode and are connected with each other. To test the whole system we need to make a request to the receiving_service. 
You can do it by:
1. making a POST request through POSTMAN to the port you specified; the request should have a form with key - uploadFile and a value of some pcap file. 
![image](https://user-images.githubusercontent.com/104463020/192141599-58df7c58-0b59-4d7d-8a9c-11b820ad9d9c.png)
2. Similarly, make a curl request. For  example 
```
...:~$ curl -v -F uploadFile=lo.pcapng -F upload=@lo.pcapng http://localhost:8080
```

where there is a lo.pcapng value you need to give the name of your file (or the directory for the second case). and at the end the address you are running the receiving_service on.

3. Or runnin a client service like client.go that was given in this repository in client folder by doing:

```
...:~$ go run client.go
```

Step 7. Possible answers

After running all the services and trying out some tests there are several outcomes you might get.

1) If everything is done errorless the you should get the statistics in this format:
```
TCP: 0
UDP: 0
IPv4: 0
IPv6: 0
```
2) If there is a problem in saving_service:
```
could not save data
```

3) If  the problem in pcap_statistics:
```
could not make statistics
```

4) If there are error messages like:
```
INVALID_FILE
```
It means that input data isn't correct. You should consider sending right file through right methods and the file size should not exceed 300mb.

5) If you get some other errors it means that there is a problem in receiving_service.

To stop the containers we use the command:

```
...:-$ sudo docker stop [container-name]
```
