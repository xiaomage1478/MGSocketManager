# MGSocketManager

## 封装CocoaAsyncSocket进行文件上传.由于公司有大量的图片上传需求,要求使用socket上传,因此封装了一下方便使用.

***

1. 创建socket并连接服务器
````
    // 创建GCDSocket
    self.GCDSocket = [[GCDAsyncSocket alloc] initWithDelegate:self delegateQueue:dispatch_get_main_queue()];

    // 连接服务器
    NSError *error = nil;
    if (![self.GCDSocket connectToHost:self.socketHost onPort:self.socketPort error:&error])
    {
        NSString *err = [NSString stringWithFormat:@"socket连接失败,错误代码:%ld",(long)error.code];
        self.errorStr = err;
        [self socketUploadError];
    }
````
2. 连接成功之后 首先上传头协议 (协议跟服务器协商,作加密,断点续传等特殊处理)

````
    - (void)socket:(GCDAsyncSocket *)sock didConnectToHost:(NSString *)host port:(UInt16)port
    {
        NSLog(@"1 didConnectToHost:%@ port:%hu", host, port);
        self.isConnect = YES;
        // 开始上传
        [self uploadFileData];
    }
     

````
3. 头协议上传成功后服务器返回响应数据 并解析

````
    - (void)socket:(GCDAsyncSocket *)sock didReadData:(NSData *)data withTag:(long)tag
    {
        NSLog(@"3 didReadData:");
        if (tagReadData == tag)
        {
        // 对得到的data值进行解析
        [self fileInfoWithData:data];
        }
        [self.GCDSocket readDataWithTimeout:timeWithout tag:tagReadData];
    }

````
4. 根据解析数据进行文件上传

````
    //获取到第一个 1 之后开始上传
    if (self.fileHandle && self.beginUpload)
    {
        // 文件当前上传记录位置，及是否继续上传
        if (self.filelength > self.currentOffset)
        {
            //找到文件当前上传进度位置
            [self.fileHandle seekToFileOffset:self.currentOffset];

            unsigned long long newOffSet = OffSet;
            //判断剩余文件大小是否小于固定上传大小,不足固定大小直接上传完毕,防止进度条越界
            BOOL o = self.filelength - self.currentOffset < OffSet;
            newOffSet = o ? (self.filelength - self.currentOffset):OffSet;
            self.currentOffset = o ? self.filelength : self.currentOffset+OffSet;

            //上传进度 0.0 ~ 1.0
            CGFloat progress = (CGFloat)self.currentOffset/self.filelength;
            if (self.uploadingBlock) {
                self.uploadingBlock(progress);
            }

            // 根据文件偏移量，读取固定大小的文件信息
            NSData *bodyData = [self.fileHandle readDataOfLength:(NSUInteger)newOffSet];
            // 向服务器发送数据
            [self.GCDSocket writeData:bodyData withTimeout:timeWithout tag:tagWriteData];
            [self.GCDSocket readDataWithTimeout:timeWithout tag:tagReadData];
            bodyData = nil;
        }
    }

````

5. 文件上传完毕后关闭socket连接

## MGSocketManager使用

1. 首先要设置文件路径 ,文件名称,ip, 端口,这些都是必须设置属性
````
    #pragma mark - 先设置要上传的文件路径,文件名，然后再连接服务器进行上传

    /** 文件路径 */
    @property (nonatomic, copy) NSString *filePath;

    /** 文件上传名称 */
    @property (nonatomic, copy) NSString *fileName;

    #pragma mark - 登录之后在首页设置host 和 port. PS:主要针对测试更换ip
    /** ip */
    @property (nonatomic, copy) NSString *host;

    /** 端口 */
    @property (nonatomic, copy) NSString *port;
    
````

2. 设置完调用连接方法即可

````
    /**
    GCDSocket连接（先设置以上属性，然后再连接服务器进行上传）
    */
    - (void)GCDSocketConnectWithUploadingBlock:(SocketUploadingBlock)uploading
    withUploadCompletion:(SocketDidCompletion)completion
    withUploadError:(SocketUploadErrorBlock)errorBlock;
    
````
3. 需要手动关闭连接/取消上传 调用 ` - (void)GCDSocketDisconnect; ` 

