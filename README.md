<h1 align="center">celestia验证节点教程</h1>

# 第 一 部分 环境安装

## 1. 系统依赖安装
```shell
# 系统为ubuntu20.04
sudo apt update                                                                                          # 更新源
sudo apt upgrade -y                                                                                      # 更新最新的包
sudo apt install curl tar wget clang pkg-config libssl-dev jq build-essential git make ncdu tmux -y  # 安装用到的依赖包
```

## 2. golang编译环境安装
```shell
cd ~                                                           # 回到家目录
wget "https://golang.org/dl/go1.18.2.linux-amd64.tar.gz"       # 下载go1.18.2压缩包
sudo rm -rf /usr/local/go                                      # 删除原有的go
sudo tar -C /usr/local -zxvf "go1.18.2.linux-amd64.tar.gz"     # 解压新的go
rm -rf go1.18.2.linux-amd64.tar.gz                             # 解压完成后可删除下载的go1.18.2

# 设置golang的环境变量
echo "export GOROOT=/usr/local/go" |  sudo tee -a /etc/profile
echo "export GOPATH=$HOME/go" |  sudo tee -a /etc/profile
echo "export PATH=$PATH:/usr/local/go/bin:$GOPATH/bin" |  sudo tee -a /etc/profile
echo "export GO111MODULE=on" |  sudo tee -a /etc/profile
echo "export GOPROXY=https://goproxy.cn" |  sudo tee -a /etc/profile

# 使用环境生效
source /etc/profile

# 检查golang版本, 有1.18.2版本输出表示环境设置成功
go version
```

## 3. 设置进程运行的文件句柄数量
```shell
# 设置文件句柄数为1048576(如果这个数不够大, 进程可能会莫名其秒的自动退出)
echo "ulimit -SHn 1048576" |  sudo tee -a /etc/profile
echo "* hard nofile 1048576" |  sudo tee -a /etc/security/limits.conf
echo "* soft nofile 1048576" |  sudo tee -a /etc/security/limits.conf
echo "DefaultLimitNOFILE=1048576" |  sudo tee -a /etc/systemd/user.conf
echo "DefaultLimitNOFILE=1048576" |  sudo tee -a /etc/systemd/system.conf
echo "session required pam_limits.so" |  sudo tee -a /etc/pam.d/common-session

# 使用环境生效
source /etc/profile
```


# 第 二 部分 部署celestia-app

## 1. 编译 celestia-appd 二进制应用
```shell
cd ~                                                          # 回到家目录
rm -rf celestia-app                                           # 删除旧的项目代码仓库目录
git clone https://github.com/celestiaorg/celestia-app.git     # 下载新的项目代码仓库目录
cd celestia-app/                                              # 进入项目代码仓库目录
git checkout tags/v0.6.0 -b v0.6.0                            # 切换到官方要求的项目仓库分支
make install                                                  # 编译，并自动安装到bin目录下
celestia-appd version                                         # 查看编译出来的版本(如果提示找不到二进制文件，则需要检查bin目录是否添加到$PATH环境变量中)
```

## 2. 准备网络创世文件
```shell
cd ~                                                                # 回到家目录
rm -rf networks                                                     # 删除旧的项目代码仓库目录
git clone https://github.com/celestiaorg/networks.git               # 下载新的项目代码仓库目录, 目的是为了获取到里面的genesis.json创世文件
MONIKER="moran666666"                                               # MONIKER为验证节点的名称, 会显示在浏览器上, 可以修改为自己的
celestia-appd init $MONIKER --chain-id mamaki                       # 会生成~/.celestia-app/配置目录
cp ~/celestia/networks/mamaki/genesis.json ~/.celestia-app/config   # 拷贝genesis.json创世文件到~/.celestia-app/配置目录中
BOOTSTRAP_PEERS=$(curl -sL https://raw.githubusercontent.com/celestiaorg/networks/master/mamaki/bootstrap-peers.txt | tr -d '\n')  # 获取到官方的启动节点列表
sed -i.bak -e "s/^bootstrap-peers *=.*/bootstrap-peers = \"$BOOTSTRAP_PEERS\"/" $HOME/.celestia-app/config/config.toml             # 修改配置文件, 将官方的启动节点添加到里面
```

## 3. 配置节点为修剪模式（节省存储空间）<br>
下面的命令主要是修改celestia-appd配置文件app.toml里面的部分内容, 修改为 pruning = "custom", pruning-keep-recent = 100, pruning-interval = 10
```shell
PRUNING="custom"
PRUNING_KEEP_RECENT="100"
PRUNING_INTERVAL="10"
sed -i -e "s/^pruning *=.*/pruning = \"$PRUNING\"/" $HOME/.celestia-app/config/app.toml
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"$PRUNING_KEEP_RECENT\"/" $HOME/.celestia-app/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"$PRUNING_INTERVAL\"/" $HOME/.celestia-app/config/app.toml
```

## 4. 配置节点为validator运行模式<br>
下面的命令主要是修改celestia配置文件config.toml里面的部分内容, 修改为 mode = "validator"
```shell
sed -i.bak -e "s/^mode *=.*/mode = \"validator\"/" $HOME/.celestia-app/config/config.toml
```

## 5. 复位网络区块数据，并下载新数据导入
```shell
cd ~                                                                                            # 回到家目录
celestia-appd tendermint unsafe-reset-all --home $HOME/.celestia-app                            # 复位网络区块数据, 将会清空$HOME/.celestia-app/data 目录下的区块数据文件
SNAP_NAME=$(curl -s https://snaps.qubelabs.io/celestia/ | egrep -o ">mamaki.*tar" | tr -d ">")  # 获取将要下载的区块数据文件名称
wget https://snaps.qubelabs.io/celestia/${SNAP_NAME}                                            # 下载50多G的快照文件, 下载视网络速率情况，可能需要很长时间
tar -xvf ${SNAP_NAME} -C $HOME/.celestia-app/data/                                              # 解压下载的区块数据到 $HOME/.celestia-app/data/ 目录中, 解压文件大也需一段时间
```

## 6. 启动celestia-appd应用：
```shell
tmux new -s celestia -d -n appd                                                       # 新建tmux名称为celestia的后台，并在里面建appd窗口
tmux send-keys -t celestia:appd "celestia-appd start 2>&1 | tee ~/appd.log " C-m      # 在常规命令行中, 发送命令到tmux celestia后台的appd窗口中启动应用                                      
curl -s localhost:26657/status | jq .result | jq .sync_info                           # 启动应用后就可以查看同步信息, 确保 "catching_up": false, 否则让它运行直到处于同步状态。
```
***注意: 新手等这里查询出来同步后再往下执行！！！***<br><br>
tmux命令知识: 
```shell
一、常用命令 --- 在常规命令终端下执行
1. tmux new -s session0 -d                                            # 新建会话session0, 并转入后台运行 ---（new 新建会话; -s 后带会话名称; -d 后台运行）
2. tmux new-window -t session0 -n window0                             # 在后台会话session0中, 新建窗口window0 ---（new-window 新建窗口; -t 代表tmux后台会话; -n 窗口名称）跟Ctrl+b+c一样，只不过可以用此命令做自动化
3. tmux send-keys -t session0:window0 "celestia bridge start " C-m    # 向后台会话session0的window0窗口中发送命令
4. tmux kill-session -t session0                                      # 关闭会话session0
5. tmux ls                                                            # 列出后台会话
6. tmux a                                                             # 打开tmux后台最近的会话

二、常用快捷键 --- 在tmux会话窗口中执行 (默认的前缀键是Ctrl+b，即先同时按下Ctrl+b，放开后再按快捷键)
1. Ctrl+b w                                                           # 在tmux中切换会话
2. Ctrl+b c                                                           # 在会话中新建窗口
3. Ctrl+b ,                                                           # 重命名窗口名称
4. Ctrl+b p                                                           # 切换到上一个窗口
5. Ctrl+b n                                                           # 切换到下一个窗口
6. Ctrl+b d                                                           # 将会话分离, 回到常规命令终端窗口
```

## 7. 创建节点钱包
```shell
celestia-appd config keyring-backend test      # 会在~/.celestia-app/下生成钱包信息
celestia-appd keys add validator               # 创建名为 validator 的钱包并生成地址(请保管好钱包的助记词, 私钥等信息)
celestia-appd keys list                        # 查看钱包地址
```
到discord mamaki-faucet频道请求水：$request celestia1dxxxxxxxxxxxxxxxxxxxxxx, 在浏览器(https://celestia.explorers.guru)查询钱包是否收到水

## 8. 质押token到验证节点
```shell
VALIDATOR_WALLET=celestia1dxxxxxxxxxxxxxxxxxxxxxx                                                  # 验证节点的钱包地址 (用上面创建的钱包地址)
AMOUNT=620000000utia                                                                               # 质押币数量(后面6个0是小数,这里实际上是620个币)
celestia-appd keys show $VALIDATOR_WALLET --bech val -a                                            # 返回一个操作员的钱包地址, 将其修改到下面 OP_WALLET 变量的等号后面
OP_WALLET=celestiavaloper1dxxxxxxxxxxxxxxxxxxxxxx                                                  # 操作员钱包地址变量
celestia-appd tx staking delegate $OP_WALLET $AMOUNT --from=$VALIDATOR_WALLET --chain-id=mamaki    # 质押多少币到操作员地址的验证节点上
```

# 第 三 部分 部署celestia-node

## 1. 编译 celestia 二进制应用
```shell
cd ~                                                          # 回到家目录
rm -rf celestia-node                                          # 删除旧的项目代码仓库目录
git clone https://github.com/celestiaorg/celestia-node.git    # 下载新的项目代码仓库目录
cd celestia-node/                                             # 进入项目代码仓库目录
git checkout tags/v0.3.0-rc2                                  # 切换到官方要求的项目仓库分支(v0.3.0-rc2分支要用go1.18.2编译，不能用go1.19.*版本，版本太高会编译出错)
make install                                                  # 编译，并自动安装到bin目录下
celestia version                                              # 查看编译出来的版本(如果提示找不到二进制文件，则需要检查bin目录是否添加到$PATH环境变量中)
```

## 2. 初始化、启动 bridge node<br>
启动网桥：让我们通过启动 Celestia 网桥来连接需要连接的应用程序：这个网桥将连接两层网络，严格来说是数据层和共识层。使用以下命令启动网桥
```shell
celestia bridge init --core.remote http://localhost:26657 --core.grpc http://localhost:26657       # 会生成~/.celestia-bridge/配置目录
tmux new-window -t celestia -n bridge                                                              # 在现有tmux名称为celestia的后台中新建bridge窗口
tmux send-keys -t celestia:appd "celestia bridge start 2>&1 | tee ~/bridge.log " C-m               # 在常规命令行中, 发送命令到tmux celestia后台的bridge窗口启动bridge node
```

## 3. 在链上创建验证节点
```shell
AMOUNT="620000000utia"                                      # 质押币数量(后面6个0是小数,这里实际上是620个币), 可以根据自己币的数量情况来填写
MONIKER="moran666666"                                       # MONIKER为验证节点的名称, 会显示在浏览器上, 可以修改为自己的
VALIDATOR_WALLET="celestia1dxxxxxxxxxxxxxxxxxxxxxx"         # 之前创建的钱包地址, 确保有足够的质押币
celestia-appd tx staking create-validator \
    --amount=$AMOUNT \
    --pubkey=$(celestia-appd tendermint show-validator) \
    --moniker=$MONIKER \
    --chain-id=mamaki \
    --commission-rate=0.1 \
    --commission-max-rate=0.2 \
    --commission-max-change-rate=0.01 \
    --min-self-delegation=1000000 \
    --from=$VALIDATOR_WALLET \
    --keyring-backend=test
```
上面命令执行成功后，可以从区块浏览器(https://celestia.explorers.guru)上查询到验证节点的情况

# 第 四 部分 其他有用的命令
```shell
curl -s localhost:26657/status | jq .result | jq .sync_info                                                                           # 查同步状态
celestia-appd q staking validator $OP_WALLET                                                                                          # 用操作员钱包地址检查验证器的状态
celestia-appd q bank balances $VALIDATOR_WALLET --node https://rpc-mamaki.pops.one                                                    # 查询钱包余额
celestia-appd tx bank send $from_adress $to_address 10000000utia --node https://rpc-mamaki.pops.one/ --chain-id mamaki --gas auto -y  # 钱包转帐
```
rpc地址:<br>
https://rpc-mamaki.pops.one<br>
https://rpc-1.celestia.nodes.guru<br>
https://grpc-1.celestia.nodes.guru:10790<br>
https://celestia-testnet-rpc.polkachu.com<br>
https://rpc.celestia.testnet.run<br>
https://rpc.mamaki.celestia.counterpoint.software<br>
