# PCAP processing service

Consists of services: receiving_service, pcap_statistics, saving_service

## Receiving Service (receiving_service)

Receiving Service uses HTTP server to accept data and send them further by GRPC. HTTP server only accepts POST requests with form name "uploadFile" and value as a file. Since the system as a whole is made for working with pcap files, you need to send them to this only pcap files.
It needs HTTP, GRPC addresses and adress for the server that collects metrics given by Prometheus. For that it needs environmetnal varibales set and instructions for that are in the step-by-step instructions section below. 

## Pcap Statistics (pcap_statistics)

Pcap Statistics accepts data and sends them over both by GRPC server and client. Before sending data further it needs to process it and make some statistics. This functionality is made explicitly for pcap files. So it is required to send only pcap files to this service.
Similar to Receiving Service needs to get GRPC server and client addresses and Metrics address from environmental variables.

## Saving Service (saving_service)

It accepts the data through GRPC server and saves them in the Database. Since all of the services are connected, this service gets data from Pcap Statistics, which are statistics and the files itself (exactly in this order). It saves data into MySQL db.
It also needs GRPC server address and Metrics address from environmental variables. Additionally, directory and dsn for saving data.

## Step-by-step Instructions

Step 1. Cloning repositories

Clone files in your preferred local repositories (each of them needs separate  repository) by entering this commands in Terminal:

```
git clone https://github.com/whym9/receiving_service.git

git clone https://github.com/whym9/pcap_statistics.git

git clone https://github.com/whym9/saving_service.git

```

Step 2. Building Docker Images

Start building docker images subsequentially. Go to each services' repository to locate the Dockerfile and run these commands:

```
.../receiving_service:~ sudo docker build -t my-app . 

```

Same with other services. my-app parameter is the name of the image. You can call images how you want to.

Step 4. Creating .env files

Create a file in each of your cloned repositories with .env extension. 
In this file you need to declare environmental variables by writing to it in this format:
```
VAR_NAME1=VAR_VALUE1
VAR_NAME=VAR_VALUE2
...
```
(Have to be in all caps!)

List of environmental valriable names for server adresses and additional values. (all of them are strings)
List of environmental variables for receiving_service: HTTP_RECEIVER, GRPC_SENDER, PROMETHEUS_ADDRESS. (all are addresses)
List of environmental variables for pcap_statistics: GRPC_RECEIVER, GRPC_SENDER, PROMETHEUS_ADDRESS (addresses) and DIR (directory).
List of environmental variables for saving_service: GRPC_SENDER, PROMETHEUS_ADDRESS (addresses) and DIR, DSN (directory and DSN for MySQL).


Step 5. Running docker images as containers 

We use the command -pd to run the image in a detached mode and give it some parameters of host_port:docker_port to connect your host port to docker's. For example:

```
sudo docker run -pd 8000:80 image-name
```

8000:80 part means that the host port 8000 should be connected to docker port 80.

Step 6. Testing

To test it you can use localhost with different ports for addresses.

Now all services are running in a detached mode and are connected with each other. To test the whole system we need to make a request to the receiving_service. 
You can do it by:
1. making a POST request through POSTMAN to the port you specified; the request should have a form with key - uploadFile and a value of some pcap file. 
![image](https://user-images.githubusercontent.com/104463020/192141599-58df7c58-0b59-4d7d-8a9c-11b820ad9d9c.png)
3. Similarly, make a curl request. For  example 
```
curl -v -F uploadFile=lo.pcapng -F upload=@lo.pcapng http://localhost:8080
```

where there is a lo.pcapng value you need to give the name of your file (or the directory for the second case). and at the end the address you are running the receiving_service on.

5. Or runnin a client service like client.go that was given in this repository in client folder by doing:
```
go run client.go
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
could not makw statistics
```

4) If there are error messages like:
```
INVALID_FILE
```
It means that input data isn't correct. You should consider sending right file through right methods and the file size should not exceed 300mb.

5) If you get some other errors it means that there is a problem in receiving_service.
