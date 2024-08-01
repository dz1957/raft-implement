# raft-java
Raft implementation library for Java.<br>
参考自[Raft论文](https://github.com/maemual/raft-zh_cn)和Raft作者的开源实现[LogCabin](https://github.com/logcabin/logcabin)。

# 支持的功能
* leader选举
* 日志复制
* snapshot
* 集群成员动态更变

## Quick Start
在本地单机上部署一套3实例的raft集群，执行如下脚本：<br>
cd raft-java-example && sh deploy.sh <br>
该脚本会在raft-java-example/env目录部署三个实例example1、example2、example3；<br>
同时会创建一个client目录，用于测试raft集群读写功能。<br>
部署成功后，测试写操作，通过如下脚本：
cd env/client <br>
./bin/run_client.sh "list://127.0.0.1:8051,127.0.0.1:8052,127.0.0.1:8053" hello world <br>
测试读操作命令：<br>
./bin/run_client.sh "list://127.0.0.1:8051,127.0.0.1:8052,127.0.0.1:8053" hello

