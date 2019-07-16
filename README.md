### Please refer the official website

https://university.mongodb.com/

### MongoDB-Certification

Repository for material and practice stuff for mongoDB certification

### Some important links used

- http://tuyenvuong.me/

### Connecting to univeristy.mongodb.com class atlas cluster from the mongo shell

- Please enter the following command in the command prompt. You can find the details of the command from the tutorial video

```
mongo "mongodb://cluster0-shard-00-00-jxeqq.mongodb.net:27017,cluster0-shard-00-01-jxeqq.mongodb.net:27017,cluster0-shard-00-02-jxeqq.mongodb.net:27017/100YWeatherSmall?replicaSet=Cluster0-shard-0" --authenticationDatabase admin --ssl --username m001-student --password m001-mongodb-basics
```

### Connecting to your mongodb atlas cluster from the mongo shell

- Please enter the following command in the command prompt. You can find the details of the command from your mongodb atlas account on clicking connect

```
mongo "mongodb+srv://cluster0-okti4.mongodb.net/test" --username database_username
```

### Loading JS files to your atlas cluster

- Connect to the mongo shell by going to the directory where the js file exists.
- Execute the following command after connecting to the mongo shell

```
load("file_name.js")
```