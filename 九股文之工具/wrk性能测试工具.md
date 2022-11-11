# wrk

## 简介

wrk 是一款针对 Http 协议的基准测试工具，它能够在单机多核 CPU 的条件下，使用系统自带的高性能 I/O 机制，如 epoll，kqueue 等，通过多线程和事件模式，对目标机器产生大量的负载。

## 使用

基本命令

```shell
Usage: wrk <options> <url>
  Options:
    -c, --connections <N>  总的http连接数
    -d, --duration    <T>  压测时间
    -t, --threads     <N>  使用多少个线程进行压测

    -s, --script      <S>  指定lua脚本路径
    -H, --header      <H>  为每一个http请求添加http头部
        --latency          在压测结束后，打印延迟统计信息
        --timeout     <T>  超时时间
    -v, --version          版本信息
    
  <N>代表数字参数，支持国际单位 (1k, 1M, 1G)
  <T>代表时间参数，支持时间单位 (2s, 2m, 2h)
```

测试报告

执行wrk，生成测试报告

```sh
wrk -t12 -c400 -d30s --latency http://www.baidu.com

Running 30s test @ http://www.baidu.com (压测时间 30s)
  12 threads and 400 connections (30个线程400个连接)
  Thread Stats   Avg（平均值）      Stdev（标准差）     Max（最大值）       +/- Stdev（正负标准差）
    Latency      420.66ms  				  448.67ms   					2.00s     			85.82%
    Req/Sec      12.28     					10.19    					  69.00     			77.28%
  Latency Distribution (延迟分布 应该是 p50 p90 p99)
     50%  284.48ms
     75%  549.36ms
     90%    1.03s
     99%    1.88s
  2562 requests in 30.07s, 26.73MB read （30s内总共发起了2562个请求）
  Socket errors: connect 158, read 0, write 0, timeout 256 (发生错误数)
Requests/sec:     85.20 (每秒请求数 即qps)
Transfer/sec:      0.89MB (每秒传输数据量)

```

