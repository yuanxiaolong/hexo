---
title: AWS S3 Java API
comments: true
share: true
toc: true
date: 2016-07-06 21:12:48
category: ['AWS']
tags: AWS
description: 用 aws 的 s3 来进行存储
---

本文介绍一下 aws s3 （simple storage service）的 api 操作

<!--more-->

利用 s3 我们可以将存储外挂在 aws 上，是一种扩展性的服务

## 环境准备

* 一个aws账号，目前只能申请国际账号，即非中国账号。申请的时候，需要一张信用卡，及电话一部
* java环境

aws账号的注册，需要信用卡，然后冻结1$ 以免滥注册，以及需要接听一个美国呼叫过来的电话，是一个语音验证码

![](/images/aws/20160706/1.png)

### 获取 accessKeyID 及 secretKey

首先登陆账户-> 「安全证书」

![](/images/aws/20160706/2.png)

创建「访问密钥」，这个密钥创建完后，只显示一次，请妥善保管

![](/images/aws/20160706/3.png)

由于我已经创建过了密钥，所以这里会存在一条记录

![](/images/aws/20160706/4.png)

### s3中的概念

简单说明一下，更详情的可以百度。
buckets
桶 ，在s3中最大粒度的 存储单位 是 buckets，一般情况，需要你先通过 aws 官网登陆 s3 服务，在网页里人工创建buckets ，而不是通过 代码 api 来创建。buckets的命名 aws 建议 用类似这种命名法，来保证唯一 `yxl.aws.bucket` ，从代码的角度上看，一个bucket 就是一个命名空间

文件系统
在同一个bucket里，文件组织形式，虽然跟linux一样采用树形结构，但实际上没有完全一致。在AWS里采用『前缀』+『实际文件名』的形式来存储，只不过前缀的表现形式是树形结构

注：在s3中没有类似linux中 `/` 这样的根目录，在s3中指定了bucket，就充当了根目录

例如下图中有3个文件夹，那test文件夹的『前缀』是什么呢？

![](/images/aws/20160706/5.png)
![](/images/aws/20160706/6.png)

答案是 test 而不是 /test，因为s3已经忽略了根路径。如果你创建了一个前缀为 /test/abc.txt 的文件在bucket里，
那么s3变解析为 『//test/abc.txt』，及2个斜杠，在路径中表现为一个空文件夹下，才有test文件夹，及test文件夹下的abc.txt

### 编写代码

1.添加 pom.xml 依赖

``` xml
<dependency>
       <groupId>com.amazonaws</groupId>
       <artifactId>aws-java-sdk</artifactId>
       <version>1.10.48</version>
</dependency>
```

2.建立认证，其中 accessKeyID和secretKey 就是你刚才重点保存的。（中国调用美国接口，可能会超时）

``` java
AWSCredentials credentials;
AmazonS3 s3Client;
credentials = new BasicAWSCredentials(accessKeyID, secretKey);
s3Client = new AmazonS3Client(credentials);
```

列出buckets

```java
private static void listBuckets(AmazonS3 s3Client){
        List<Bucket> list = s3Client.listBuckets();
        for (Bucket bucket : list){
            // java.net.SocketException: Connection reset
            // java.net.SocketTimeoutException: connect timed out
            // java.io.EOFException: SSL peer shut down incorrectly
            System.out.println("--------->" + bucket.getName() + " ----> " + bucket.getOwner());
        }
    }
```

遍历所有文件

```java
private static void listAll(AmazonS3 s3Client,String bucketName){
        ObjectListing objects = s3Client.listObjects(bucketName);
        do {
            for (S3ObjectSummary objectSummary : objects.getObjectSummaries()) {
                System.out.println("Object: " + objectSummary.getKey());
            }
            objects = s3Client.listNextBatchOfObjects(objects);
        } while (objects.isTruncated());
    }
```

查询指定文件信息

``` java
    //size : 86791631
    private static void info(AmazonS3 s3Client,String bucketName){
        ObjectMetadata metadata = s3Client.getObjectMetadata(bucketName,"test/apache-hive-1.1.0-bin.tar.gz");
        System.out.println("size : " + metadata.getContentLength());
//        System.out.println("md5 : " + metadata.getContentMD5());
    }
```

删除指定文件

``` java
    private static void delete(AmazonS3 s3Client,String bucketName){
        // 重复删除没有关系
        DeleteObjectRequest request = new DeleteObjectRequest(bucketName, "/test/apache-hive-1.1.0-bin.tar.gz");
        s3Client.deleteObject(request);
    }
```

上传单个文件（s3接口已封装成多块上传）
``` java
    // 同名覆盖
    private static void upload(AmazonS3 s3Client,String bucketName,String from,String to){
        TransferManager tm = new TransferManager(s3Client);
        TransferManagerConfiguration conf = tm.getConfiguration();

        Upload upload = tm.upload(bucketName,to,new File(from));
        TransferProgress p = upload.getProgress();
        while (upload.isDone() == false){
            int percent =  (int)(p.getPercentTransferred());
            System.out.print("\r" + from + " - " + "[ " + percent + "% ] "
                    + p.getBytesTransferred() + " / " + p.getTotalBytesToTransfer() );
            // Do work while we wait for our upload to complete...
            try {
                Thread.sleep(500);
            }catch (Exception e){

            }
        }
        try{
            upload.waitForCompletion();
            // 默认添加public权限
            s3Client.setObjectAcl(bucketName, to, CannedAccessControlList.PublicRead);
        }catch (Exception e){
            System.out.println(e.getMessage());
        }finally {
            tm.shutdownNow();
        }
        System.out.print("\r" + from + " - " + "[ 100% ] "
                + p.getBytesTransferred() + " / " + p.getTotalBytesToTransfer() );

    }
```

上传整个文件夹下的文件(不会创建本地文件前的路径)

``` java
private static void uploadDirectory(AmazonS3 s3Client,String bucketName){
        TransferManager tm = new TransferManager(s3Client);
        final String localpath = "/Users/yxl/Downloads/highlight";
        MultipleFileUpload multipleFileUpload = tm.uploadDirectory(bucketName, "test/", new File(localpath), false);
        final TransferProgress p = multipleFileUpload.getProgress();
        multipleFileUpload.addProgressListener(new ProgressListener() {
            @Override
            public void progressChanged(ProgressEvent progressEvent) {
                double percent = p.getPercentTransferred();
                System.out.print("\n" + localpath + " - " + "[ " + String.format("%.2f",percent) + "% ] "
                        + p.getBytesTransferred() + " / " + p.getTotalBytesToTransfer() );

                if (progressEvent.getEventType() == ProgressEventType.TRANSFER_COMPLETED_EVENT) {
                    System.out.println(" Upload complete!!!");
                }
            }
        });

        try{
            multipleFileUpload.waitForCompletion();
        }catch (Exception e){
            System.out.println(e.getMessage());
        }finally {
            tm.shutdownNow();
        }
    }
```

上传多个文件（会创建本地文件前的路径到s3）

``` java
//这个会创建相对路径的文件，例如本地路径 localfile1 和 localfile2，则上传到s3后，文件路径为test/Users/yxl/Documents/q.txt 和 test/Users/yxl/Downloads/ha.xml
    private static void uploadMultFiles(AmazonS3 s3Client,String bucketName){
        TransferManager tm = new TransferManager(s3Client);
        final String localfile1 = "/Users/yxl/Documents/q.txt";
        final String localfile2 = "/Users/yxl/Downloads/ha.xml";

        MultipleFileUpload multipleFileUpload = tm.uploadFileList(bucketName,"test/",new File("/"), Arrays.asList(new File(localfile1),new File(localfile2)));
//        MultipleFileUpload multipleFileUpload = tm.uploadDirectory(bucketName, "test/", new File(localpath), false);
        final TransferProgress p = multipleFileUpload.getProgress();
        multipleFileUpload.addProgressListener(new ProgressListener() {
            @Override
            public void progressChanged(ProgressEvent progressEvent) {
                double percent = p.getPercentTransferred();
                System.out.print("\n" + "file now " + " - " + "[ " + String.format("%.2f",percent) + "% ] "
                        + p.getBytesTransferred() + " / " + p.getTotalBytesToTransfer() );

                if (progressEvent.getEventType() == ProgressEventType.TRANSFER_COMPLETED_EVENT) {
                    System.out.println(" Upload complete!!!");
                }
            }
        });

        try{
            multipleFileUpload.waitForCompletion();
        }catch (Exception e){
            System.out.println(e.getMessage());
        }finally {
            tm.shutdownNow();
        }
    }
```

下载文件

``` java
private static void download(AmazonS3 s3Client,String bucketName){
        GetObjectRequest request = new GetObjectRequest(bucketName,"test/apache-hive-1.1.0-bin.tar.gz");
        TransferManager tm = new TransferManager(s3Client);

        final String localpath = "/Users/yxl/Downloads/aws-hive-bin.tar.gz";
        final Download download = tm.download(request,new File(localpath));
        final TransferProgress p = download.getProgress();
        download.addProgressListener(new ProgressListener() {
            @Override
            public void progressChanged(ProgressEvent progressEvent) {

                double percent = p.getPercentTransferred();
                System.out.print("\n" + localpath + " - " + "[ " + String.format("%.2f",percent) + "% ] "
                    + p.getBytesTransferred() + " / " + p.getTotalBytesToTransfer() );

                if (progressEvent.getEventType() == ProgressEventType.TRANSFER_COMPLETED_EVENT) {
                    System.out.println(" Download complete!!!");
                }
            }
        });
        try{
            download.waitForCompletion();
        }catch (Exception e){
            System.out.println(e.getMessage());
        }finally {
            tm.shutdownNow();
        }
    }
```

下载整个文件夹

``` java
//Unable to store object contents to disk: Read timed out
    //由于下载会超时导致不完整，需要校验元信息大小和实际下载大小
    private static void downloadDirectory(AmazonS3 s3Client,String bucketName){
        TransferManager tm = new TransferManager(s3Client);
        final String localpath = "/Users/yxl/Downloads/aws_download";
        final MultipleFileDownload download = tm.downloadDirectory(bucketName,"test/",new File(localpath));
        final TransferProgress p = download.getProgress();
        download.addProgressListener(new ProgressListener() {
            @Override
            public void progressChanged(ProgressEvent progressEvent) {

                double percent = p.getPercentTransferred();
                System.out.print("\n" + localpath + " - " + "[ " + String.format("%.2f",percent) + "% ] "
                        + p.getBytesTransferred() + " / " + p.getTotalBytesToTransfer() );

                if (progressEvent.getEventType() == ProgressEventType.TRANSFER_COMPLETED_EVENT) {
                    System.out.println(" Download complete!!!");
                }
            }
        });
        try{
            download.waitForCompletion();
        }catch (Exception e){
            System.out.println(e.getMessage());
        }finally {
            tm.shutdownNow();
        }
    }
```

## 链接

aws s3 登陆地址 ：https://console.aws.amazon.com/s3/home
aws s3 java api ：http://docs.aws.amazon.com/AWSJavaSDK/latest/javadoc/index.html
