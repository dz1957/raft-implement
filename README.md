# raft-java
Raft implementation library for Java.<br>
reference:
raft paper:(https://github.com/maemual/raft-zh_cn)
author's open source implement: [LogCabin](https://github.com/logcabin/logcabin)。

# Functions
* leader election
* log replication
* snapshot
* addpeer, removepeer

## Quick Start
To deploy a 3-instance Raft cluster on a local machine, execute the following script：<br>
```sh
cd raft-java-example && sh deploy.sh 
```
This script will deploy three instances, example1, example2, and example3, in the raft-java-example/env directory；<br>
It will also create a client directory for testing the read and write functionalities of the Raft cluster.<br>
After a successful deployment, test the write operation using the following script: <br>
```sh
cd env/client
./bin/run_client.sh "list://127.0.0.1:8051,127.0.0.1:8052,127.0.0.1:8053" hello world
```
To test the read operation, use the command：<br>
```sh
./bin/run_client.sh "list://127.0.0.1:8051,127.0.0.1:8052,127.0.0.1:8053" hello
```
