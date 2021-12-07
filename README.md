# 负载均衡

先进入阿里云服务器（我的服务器）

## 环境配置

* 安装docker

* 拉取镜像

  ~~~python
  # triton server image
  docker pull nvcr.io/nvidia/tritonserver:21.07-py3
  # client image
  docker pull nvcr.io/nvidia/tritonserver:21.07-py3-sdk
  ~~~

* 拉取triton服务器的代码

  ~~~python
  # server
  git clone https://github.com/triton-inference-server/server.git
  # client
  git clone https://github.com/triton-inference-server/client.git
  ~~~



## 软件运行

* 运行服务器

  ~~~python
  # start triton_1 docker
  # 启动服务器前运行fetch_model.sh 那个脚本
  docker run --ipc=host --rm -p8000:8000 -p8001:8001 -p8002:8002 -v/root/Project/server/docs/examples/model_repository:/models nvcr.io/nvidia/tritonserver:21.07-py3 tritonserver --model-repository=/models
            
  # start triton_2 docker      主机端口：docker端口
  docker run --ipc=host --rm -p8003:8000 -p8004:8001 -p8005:8002 -v/root/Project/server/docs/examples/model_repository:/models nvcr.io/nvidia/tritonserver:21.07-py3 tritonserver --model-repository=/models
      
  
  ~~~

* 运行客户端

  ~~~python
  # start client docker
  docker run -it --rm --ipc=host --net=host nvcr.io/nvidia/tritonserver:21.07-py3-sdk
    
  docker run -it -v  --ipc=host --net=host nvcr.io/nvidia/tritonserver:21.07-py3-sdk
  
  #docker和宿主机映射，方向是从宿主机覆盖docker
  docker run -it -v /root/Project/client:/workspace/client  --ipc=host --net=host nvcr.io/nvidia/tritonserver:21.07-py3-sdk
    
  #运行perf_analyzer
  perf_analyzer -a -m densenet_onnx -u localhost:8001 --shared-memory system --request-distribution constant --request-rate-range 200  -i grpc -v
  ~~~

* 其他命令

  ~~~python
  #进入容器的命令
  docker exec -it 2bf743faed84  bash   
  # exec 表示执行，-it后面接docker ID ，bash是指命令
  
  #查看正在运行的docker
  docker ps
  #查看所有的docker
  docker ps -a
  ~~~



对文件进行编译

在workspace目录下

~~~python
$ mkdir build
$ cd build
$ cmake -DCMAKE_INSTALL_PREFIX=/workspace/client/build/install -DTRITON_ENABLE_CC_HTTP=ON -DTRITON_ENABLE_CC_GRPC=ON -DTRITON_ENABLE_PERF_ANALYZER=ON -DTRITON_ENABLE_PYTHON_HTTP=ON -DTRITON_ENABLE_PYTHON_GRPC=ON -DTRITON_ENABLE_JAVA_HTTP=ON -DTRITON_ENABLE_GPU=ON -DTRITON_ENABLE_EXAMPLES=ON -DTRITON_ENABLE_TESTS=ON ..
$ make cc-clients
~~~



linux的命令行解析参数之getopt_long函数:https://blog.csdn.net/qq_33850438/article/details/80172275



求违规率：

~~~c++
在 inference_profiler.cc 下面的 Measure 函数可以求出请求总数和违规数量
~~~

~~~c++
Main Profile<double>
  |--- inference_profiler.h Profile
  				|--- inference_profiler.cc Profile(double)
  								|--- request_rate_manager.cc ChangeRequestRate // 启动线程
  												|--- PauseWorkers  //准备工作，启动线程，线程内的函数暂时不会执行
  																|--- Infer  // 线程执行的函数，里面的Request方法放在了while循环
  																			|--- Request // 更新请求的开始时间等一些信息，发请求
  												|--- GenerateSchedule  // 创建一个schedule
  												|--- ResumeWorkers // 线程内部的函数开始执行
  								|--- ProfileHelper // 收集线程的数据，while循环执行Measure，打印中间输出
  												|--- Measure  // 收集数据，必须要间隔指定的时间才会执行，默认5s
  																|--- load_manager.cc GetAccumulatedClientStat  // 收集每一个线程的Stat数据
  																|--- load_manager.cc SwapTimestamps  // 收集每一个线程的时间戳数据
  																|--- Summarize
  																				|--- MeasurementTimestamp //计算valid_range
  																				|--- ValidLatencyMeasurement 
  																				//计算所有请求的返回时延，更新valid_sequence_count，delayed_request_count
  																				|--- SummarizeLatency // 如果empty会返回err
  
~~~





## 修改

* 创建三个全局变量

~~~c++
volatile const uint64_t max_time_delay_ns = 60002832726;//时延阀值
volatile uint64_t sum_request;//请求总数
volatile uint64_t bad_request;//违规请求数量
~~~

![image-20211206133249997](/Users/echozhou/Library/Application Support/typora-user-images/image-20211206133249997.png)

![image-20211206133332985](/Users/echozhou/Library/Application Support/typora-user-images/image-20211206133332985.png)

初始化：

![image-20211206134825017](/Users/echozhou/Library/Application Support/typora-user-images/image-20211206134825017.png)

已测试

* 把修改时延阀值作为一个可选参数传入 

![image-20211206153021681](/Users/echozhou/Library/Application Support/typora-user-images/image-20211206153021681.png)

![image-20211206153055675](/Users/echozhou/Library/Application Support/typora-user-images/image-20211206153055675.png)

已测试

* 设置违规率

![image-20211206153402512](/Users/echozhou/Library/Application Support/typora-user-images/image-20211206153402512.png)

![image-20211206153431527](/Users/echozhou/Library/Application Support/typora-user-images/image-20211206153431527.png)

* 设置change_server作为切换服务器的标志，设置为bool，并且按照early_exit进行修改
  * ProfileHelper * 1
  * Infer * 2
  * Profile 

![设置change_server](/Users/echozhou/Library/Application Support/typora-user-images/image-20211206154654502.png)

![设置change_server为全局](/Users/echozhou/Library/Application Support/typora-user-images/image-20211206154739638.png)

![ProfileHelper](/Users/echozhou/Library/Application Support/typora-user-images/image-20211206160050468.png)

![Infer](/Users/echozhou/Library/Application Support/typora-user-images/image-20211206160232960.png)

![Infer](/Users/echozhou/Library/Application Support/typora-user-images/image-20211206160349412.png)

![Profile](/Users/echozhou/Library/Application Support/typora-user-images/image-20211206190849837.png)

* 在Measure写计算违规率的函数，然后如果违规，则把change_server置为true

更新bad_request 和 sum_request

![计算](/Users/echozhou/Library/Application Support/typora-user-images/image-20211206192113133.png)

* 暴力创建一个factory等其他所需要的所有资源



## 附加：github 加速

可以直接在服务器上的`/etc/hosts`文件进行修改

https://zhuanlan.zhihu.com/p/147745547

~~~
#github
140.82.112.4 github.com
140.82.113.3 gist.github.com
199.232.69.194 github.global.ssl.fastly.net
185.199.111.153 assets-cdn.github.com
185.199.108.133 raw.githubusercontent.com

199.232.68.133 cloud.githubusercontent.com
199.232.68.133 camo.githubusercontent.com
199.232.68.133 avatars0.githubusercontent.com
199.232.68.133 avatars1.githubusercontent.com
199.232.68.133 avatars2.githubusercontent.com
199.232.68.133 avatars3.githubusercontent.com
199.232.68.133 avatars4.githubusercontent.com
199.232.68.133 avatars5.githubusercontent.com
199.232.68.133 avatars6.githubusercontent.com
199.232.68.133 avatars7.githubusercontent.com
199.232.68.133 avatars8.githubusercontent.com

185.199.108.154               github.githubassets.com
140.82.112.21                 central.github.com
185.199.108.133               desktop.githubusercontent.com
185.199.108.153               assets-cdn.github.com
185.199.108.133               camo.githubusercontent.com
185.199.108.133               github.map.fastly.net
199.232.69.194                github.global.ssl.fastly.net
140.82.112.3                  gist.github.com
185.199.108.153               github.io
140.82.113.4                  github.com
140.82.114.5                  api.github.com
185.199.108.133               raw.githubusercontent.com
185.199.108.133               user-images.githubusercontent.com
185.199.108.133               favicons.githubusercontent.com
185.199.108.133               avatars5.githubusercontent.com
185.199.108.133               avatars4.githubusercontent.com
185.199.108.133               avatars3.githubusercontent.com
185.199.108.133               avatars2.githubusercontent.com
185.199.108.133               avatars1.githubusercontent.com
185.199.108.133               avatars0.githubusercontent.com
185.199.108.133               avatars.githubusercontent.com
140.82.114.9                  codeload.github.com
52.216.106.75                 github-cloud.s3.amazonaws.com
52.217.65.52                  github-com.s3.amazonaws.com
52.216.177.203                github-production-release-asset-2e65be.s3.amazonaws.com
52.217.81.140                 github-production-user-asset-6210df.s3.amazonaws.com
52.217.80.164                 github-production-repository-file-5c1aeb.s3.amazonaws.com
185.199.108.153               githubstatus.com
64.71.168.201                 github.community
185.199.108.133               media.githubusercontent.com
~~~

~~~
ubuntu 安装ping
$ apt-get install inetutils-ping
~~~

https://www.cnblogs.com/betgar/p/14522454.html

