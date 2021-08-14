#  以太坊多节点部署

​		在实际应用中，以太坊是按照分布式网络的形式来工作的。不同于实验一中的单点虚拟环境，此次实验的目标是搭建一个多节点的以太坊工作环境，该环境由三个可挖矿的节点组成。

##### 实验学时:  3课时

**实验目的：**

* 掌握以太坊私链环境搭建的基本原理和步骤；

**预备知识:** 

###### 以太坊网络架构

​		在搭建单点以太坊网络试验中，曾经提到以太坊网络是一个点对点网络。事实上，以太坊网络是一组运行相同网络通信协议的节点，彼此之间相互平等，可以相互发现、通信，进而有区块的广播、同步等一系列操作

###### Docker和容器

​		简单点说，Docker容器相当于轻量级的虚拟机，可以用来虚拟出一个网络节点，该网络节点拥有自己的文件系统、网络接口等。一个镜像相当于模板，可以批量产生多个与镜像相同的容器。

###### geth
​		geth是以太坊是用go语言编写，使用ethereum协议的客户端软件，也是现在以太坊上使用最广泛的客户端，可以作为节点加入到任何现有的以太坊网络。

**实验内容：**

本次实验

###### 安装Docker

```bash
apt install docker.io
```

​		换源

```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": [
   	"http://ovfftd6p.mirror.aliyuncs.com",
    "http://registry.docker-cn.com",
    "http://docker.mirrors.ustc.edu.cn",
    "http://hub-mirror.c.163.com"
    ]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```

###### 拉取geth镜像及配置虚拟网络

```bash
sudo docker pull ethereum/client-go
```

​		接下来创建一个Docker网络，后面将把三个节点都放在这个网络内

````bash
sudo docker network create -d bridge --subnet=172.18.0.0/16 ethnet
docker network ls
````

​		这里bridge类似于VMware中的NAT模式，建立了一个虚拟网络，网段是172.18.0.0/16

###### 配置首个节点

​		建立一个工作目录

```bash
mkdir -p ~/ethereum-private-chain
cd ~/ethereum-private-chain
mkdir n1 n2 n3
```

​		创建一个账户

```bash
cd n1
geth -datadir ./data account new
```

​		会提示输入一个口令，该口令用于加密账户的私钥，而私钥是以后用于该账户签署转账，以及部署合约所使用的。重复两次口令之后，操作成功。

​		一定要记住这个口令，口令遗失该账户也就遗失了。可以看到第一行Public address of the key后面的是该账户的地址，复制下来，待会儿要用

​		接下来，创建初始区块的配置

````bash
cd ..
tee genesis.json <<-'EOF'
{
  "config": {
    "chainId": 9876,
    "homesteadBlock": 0,
    "eip150Block": 0,
    "eip150Hash": "0x0000000000000000000000000000000000000000000000000000000000000000",
    "eip155Block": 0,
    "eip158Block": 0,
    "byzantiumBlock": 0,
    "constantinopleBlock": 0,
    "petersburgBlock": 0,
    "istanbulBlock": 0,
    "ethash": {}
  },
  "nonce": "0x0",
  "timestamp": "0x5e69e9c0",
  "extraData": "0x0000000000000000000000000000000000000000000000000000000000000000",
  "gasLimit": "0x47b760",
  "difficulty": "0x00002",
  "mixHash": "0x0000000000000000000000000000000000000000000000000000000000000000",
  "coinbase": "0x0000000000000000000000000000000000000000",
  "alloc": { },
  "number": "0x0",
  "gasUsed": "0x0",
  "parentHash": "0x0000000000000000000000000000000000000000000000000000000000000000"
}
EOF
cp genesis.json n1
cp genesis.json n2
````

​		这里，需要关注的是ChainId，目前，公网以太坊ETH的ChainId为1，私有链设置为其他数字即可。

​		接下来，配置n1节点的设置脚本。以下的语句会在n1目录下创建一个文件miner.sh，并且在其中添加内容。**注意**，在执行下面的语句之前，**务必**将其中两处的"0x9a7AAe190cc0e43c95458f5E4B92EAA965C334C3"修改为n1下账户的地址

````bash
cd n1
tee miner.sh <<-'EOF'
#!/bin/sh
cp -r /workspace/data/keystore/* ~/data/keystore/
geth -datadir ~/data --networkid 9876 --rpc --rpcaddr "172.18.0.50" --rpccorsdomain "*" --rpcapi admin,eth,miner,web3,personal,net,txpool --allow-insecure-unlock --unlock "！！！！！！！！！！" --etherbase "！！！！！！！！！！！"  --password "/workspace/pswd" --nodiscover console
EOF
````

​		修改脚本权限

````bash
sudo chmod 0755 miner.sh
````

​		口令也放在文件里，将下面语句中的_[口令]_ 字段，替换成刚才n1下的口令。下面第二句为了让pswd文件里多一个空行

````bash
echo [口令] >> pswd
echo >> pswd
````

​		接下来新建一个启动脚本

````bash
tee init.sh <<-'EOF'
#!/bin/sh
geth -datadir ~/data/ init /workspace/genesis.json

if [  $# -lt 1 ]; then
  exec "/bin/sh"
else
  exec /bin/sh -c "$@"
fi
EOF
````

​		修改脚本权限

````bash
sudo chmod 0755 init.sh		
````

​		启动该节点

````bash
cd ..
sudo docker run -it --name=node1 --network ethnet --ip 172.18.0.50 -p 8545:8545 -p 8000:8000 --hostname node1 -v `echo $PWD`/n1:/workspace --entrypoint /workspace/init.sh ethereum/client-go /workspace/miner.sh
````

​		此时，第一个节点启动成功，可以看到geth的控制台，第一个节点搭建成功！此时，不要关闭这个终端，它可以用来接下来操作第一个节点。

###### 第二个节点

​		新打开一个终端，创建n2节点的设置脚本，同样执行下面的命令之前，将其中的"0x9a7AAe190cc0e43c95458f5E4B92EAA965C334C3"部分替换为刚才记录的n2节点的地址.注意，这里的地址变为了172.18.0.51，区别于前一个172.18.0.50。

创建n2的秘钥对
````bash
cd ~/ethereum-private-chain
cd n2
geth -datadir ./data account new
````
把秘钥写进文件里
````bash
echo [秘钥] >> pswd
echo >> pswd
````
创建相关脚本
````bash
tee miner.sh <<-'EOF'
#!/bin/sh
cp -r /workspace/data/keystore/* ~/data/keystore/
geth -datadir ~/data --networkid 9876 --rpc --rpcaddr "172.18.0.51" --rpcapi admin,eth,miner,web3,personal,net,txpool --allow-insecure-unlock --unlock "！！！！！！！！" --etherbase "！！！！！！！"  --password "/workspace/pswd" --nodiscover console
EOF
````

​		注意到这里没有n1节点中的--rpccorsdomain "*"参数。该参数是在实验三中为了让以太坊区块链浏览器接入n1节点设置的。	

​		接下来新建一个启动脚本

````bash
tee init.sh <<-'EOF'
#!/bin/sh
geth -datadir ~/data/ init /workspace/genesis.json

if [  $# -lt 1 ]; then
  exec "/bin/sh"
else
  exec /bin/sh -c "$@"
fi
EOF
````

​		修改脚本权限

````bash
sudo chmod 0755 init.sh miner.sh
````

​		启动该节点

````bash
cd ..
sudo docker run -it --name=node2 --network ethnet --ip 172.18.0.51 -p 8546:8545 --hostname node2 -v `echo $PWD`/n2:/workspace --entrypoint /workspace/init.sh ethereum/client-go /workspace/miner.sh
````

​		启动成功！同样不要关闭这个终端。

###### 连接两个节点

​		目前，只是在网内建立了两个节点，记得在上面有设置了--nodiscover参数，表示不自动发现对等节点，接下来手动设置节点，在n1终端执行

```bash
admin.nodeInfo.enode
```

​		得到n1节点的enode值，此时通过n2节点的控制台分别与n1连接。记得配置的n1节点的ip为172.18.0.50，所以在n2的控制台添加节点n1，**注意**，下面命令中的enode要修改为查看到n1节点的值，不要照抄，注意将末尾的?discport=0删除，并且把@后改为n1的ip，即172.18.0.50

````bash
admin.addPeer("enode://cad42f0ff35f51b51cb8d02bd99e5489162ba9b7e944ca5d26a497bd135456f43e1eb59aab8ef3e625a63d9c81876a0b93206d27274a1d6d2a9a683604252884@172.18.0.50:30303")
````

​		此时，三个节点之间互相建立了连接，在n1上执行

````bash
admin.peers
````

​		查看相邻节点，连接建立成功。

###### 测试挖矿

​		在n1节点上，执行

```bash
miner.start(1)
```

​		经过一段较长时间的DAG建立过程，n1节点就开始挖矿了
​		从n2和n3节点可以看到接受n1挖出并广播的块
​		此时，实验二已经结束。实验三将在实验二的基础上继续。

##### 注意

1. 此次实验需要虚拟机配置比较高，建议Ubuntu内存有至少4G。

2. 每次退出docker容器后，需要在新终端

   ````bash 
   sudo docker ps -a
   ````

   查看哪些已经退出但是未删除的容器，利用

   ````bash
   sudo docker rm <容器id>
   ````

   删除容器

   

**常见问题**

###### 1.启动容器遇到提示docker: Error response from daemon: Conflict. The container name "/node1" is already in use by container

​		是因为先前启动过一个相同的容器，需要执行

````bash
sudo docker ps -a
````

​		找到这个容器的哈希值，然后

````bash
sudo docker rm [容器哈希]
````

​		删除后再重新启动该容器即可

### 练习

1. 此次实验中如何将建立的私有链和ETH公链相隔开的？

2. 请仔细阅读实验手册中的脚本代码，指出它们的作用。

3. 为什么生成账户时要确定一个口令？该口令、私钥和公钥之间是什么关系？

4. 在这次实验中，我们配置每一个节点拥有一个账户，节点数和账户数一定一样吗？n1节点作为唯一配置的矿工节点，每个挖出区块的奖励将会给谁？是通过如何实现的？如果要让其他账户也获得相应的奖励，有什么办法？

5. 尝试将两个节点设置为矿工节点，并且观察新区块在网络中的传递情况。

6. 总结一下以太坊私有链建立的相关步骤。

   <!-- 1. 通过设置网络ID来隔开，这里ChainId用来防止私有挖矿攻击，也被有些人称为重放攻击。

   2. docker启动命令：设置ip、设置节点所处网段、主机名称、设置挂载目录、默认容器首先执行

      ````bash
      /workspace/init.sh /workspace/miner.sh
      ````

      命令。miner启动脚本，设置了网络ID、数据目录、允许rpc接入、允许http接入（非https）、关闭节点自动发现、设置本节点挖矿受益人、设置本节点签名交易所用的私钥及其口令

      3. 这个口令用于加密用户私钥。私钥用于用户签发交易及合约，私钥对应的公钥通过keccak-256哈希运算后取末尾40位得到地址。
      4. 不一定一样，可以每个节点配置多个账户，这些账户一旦出现交易就会被全区块链内获知。奖励给--etherbase后设置的账户，这样在新产生的块之后，coinbase会转到该账户。在启动节点或者是在geth的javascript控制台中，设置挖矿的受益人。
      5. 方法类似于实验手册1的矿工节点配置。

   -->

