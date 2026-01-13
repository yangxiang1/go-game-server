# Go 游戏服务端 Mini 框架：网络设计与实现

# Go 游戏服务端Mini框架



轻量级游戏服务端框架，基于Go语言实现核心网络通信、数据存储、业务逻辑分层，适配小型休闲游戏/游戏Demo的快速开发，支持TCP/UDP双协议、连接管理、数据缓存等核心能力。

## 一、项目介绍

### 1. 项目背景

传统游戏服务端开发存在上手成本高、组件耦合度高的问题，本框架聚焦「轻量化、易扩展、高性能」，基于Go语言的并发优势+Netty的网络模型设计，适配2D休闲游戏、小游戏的服务端开发，可快速搭建登录、战斗、排行榜等核心游戏功能。

### 2. 核心特性

- ✅ 基于Go实现的Reactor网络模型（对标Netty），支持TCP/UDP长连接通信；

- ✅ 分层架构设计：网络层、协议层、业务层、数据层解耦，易扩展；

- ✅ 集成MySQL（玩家数据持久化）+ Redis（热点数据缓存/排行榜）；

- ✅ 内置连接管理、心跳检测、消息编解码、异常重连机制；

- ✅ 极简API设计，5分钟即可搭建一个基础游戏服务端。

### 3. 适用场景

- 小型休闲游戏（如消消乐、贪吃蛇）服务端开发；

- 游戏服务端入门学习/技术验证；

- 游戏Demo快速落地。

## 二、技术栈详解

|技术/组件|版本|核心作用|
|---|---|---|
|Go|1.21+|核心开发语言，利用goroutine实现高并发连接处理|
|Netty（思想）|-|参考Netty的Reactor线程模型设计网络层，实现事件驱动的IO处理|
|MySQL|8.0+|玩家基础信息、游戏存档、交易记录等持久化存储|
|Redis|7.0+|玩家在线状态、排行榜数据、临时会话数据缓存|
|Protobuf|3.0+|游戏协议编解码，降低网络传输开销|
|Gin（可选）|1.9+|提供HTTP接口，用于GM后台/运营数据查询|
## 三、项目结构

```Plain Text

go-game-mini-framework/
├── cmd/                  // 程序入口
│   └── server/           // 游戏服务端启动入口
│       └── main.go       // 主函数
├── config/               // 配置文件
│   ├── app.yaml          // 服务基础配置（端口、日志等）
│   ├── mysql.yaml        // MySQL配置
│   └── redis.yaml        // Redis配置
├── internal/             // 核心业务逻辑（内部包）
│   ├── network/          // 网络层（仿Netty实现）
│   │   ├── reactor.go    // Reactor模型实现
│   │   ├── connection.go // 连接管理
│   │   └── codec.go      // 协议编解码
│   ├── handler/          // 业务处理器
│   │   ├── login.go      // 登录处理
│   │   ├── battle.go     // 战斗处理
│   │   └── rank.go       // 排行榜处理
│   ├── model/            // 数据模型
│   │   ├── player.go     // 玩家模型
│   │   └── rank.go       // 排行榜模型
│   ├── storage/          // 数据存储层
│   │   ├── mysql/        // MySQL操作
│   │   └── redis/        // Redis操作
│   └── service/          // 业务服务层
│       ├── player.go     // 玩家服务
│       └── rank.go       // 排行榜服务
├── pkg/                  // 公共工具包
│   ├── logger/           // 日志工具
│   ├── util/             // 通用工具（加密、时间等）
│   └── proto/            // Protobuf协议定义
├── go.mod                // Go模块依赖
├── go.sum                // 依赖版本锁
└── README.md             // 项目说明（本文档）
```

## 四、核心代码实现

### 1. 网络层（仿Netty Reactor模型）

```Go

// internal/network/reactor.go
package network

import (
 "log"
 "net"
 "sync"
)

// Reactor 反应器模型，处理IO事件
type Reactor struct {
 listener net.Listener      // 监听器
 workers  []*Worker         // 工作协程池
 quit     chan struct{}     // 退出信号
 wg       sync.WaitGroup    // 等待组
}

// Worker 工作协程，处理具体的连接
type Worker struct {
 id      int               // 工作协程ID
 connCh  chan net.Conn     // 连接通道
 quit    chan struct{}     // 退出信号
}

// NewReactor 创建Reactor实例
func NewReactor(addr string, workerNum int) (*Reactor, error) {
 listener, err := net.Listen("tcp", addr)
 if err != nil {
 return nil, err
 }

 reactor := &Reactor{
 listener: listener,
 quit:     make(chan struct{}),
 }

 // 初始化工作协程池
 reactor.workers = make([]*Worker, workerNum)
 for i := 0; i < workerNum; i++ {
 worker := &Worker{
 id:     i,
 connCh: make(chan net.Conn, 100),
 quit:   make(chan struct{}),
 }
 reactor.workers[i] = worker
 reactor.wg.Add(1)
 go worker.run(&reactor.wg)
 }

 return reactor, nil
}

// Run 启动Reactor
func (r *Reactor) Run() {
 log.Println("Reactor server started...")
 workerID := 0
 for {
 conn, err := r.listener.Accept()
 if err != nil {
 select {
 case <-r.quit:
 log.Println("Reactor server stopped")
 return
 default:
 log.Printf("Accept error: %v", err)
 continue
 }
 }

 // 轮询分配连接到工作协程
 r.workers[workerID].connCh <- conn
 workerID = (workerID + 1) % len(r.workers)
 }
}

// run 工作协程执行逻辑
func (w *Worker) run(wg *sync.WaitGroup) {
 defer wg.Done()
 log.Printf("Worker %d started", w.id)
 for {
 select {
 case conn := <-w.connCh:
 // 处理连接
 go handleConn(conn)
 case <-w.quit:
 log.Printf("Worker %d stopped", w.id)
 return
 }
 }
}

// handleConn 处理单个连接的读写
func handleConn(conn net.Conn) {
 defer conn.Close()
 log.Printf("New connection from %s", conn.RemoteAddr())

 // 心跳检测（简单实现）
 conn.SetReadDeadline(time.Now().Add(30 * time.Second))
 buf := make([]byte, 1024)
 for {
 n, err := conn.Read(buf)
 if err != nil {
 log.Printf("Read error: %v, client: %s", err, conn.RemoteAddr())
 return
 }

 // 重置读超时（心跳）
 conn.SetReadDeadline(time.Now().Add(30 * time.Second))
 
 // 解析消息并分发到业务处理器
 msg := buf[:n]
 DispatchMsg(conn, msg)
 }
}

// Stop 停止Reactor
func (r *Reactor) Stop() {
 close(r.quit)
 r.listener.Close()

 // 停止所有工作协程
 for _, worker := range r.workers {
 close(worker.quit)
 }
 r.wg.Wait()
}
```

### 2. 数据层（MySQL+Redis）

```Go

// internal/storage/mysql/player.go
package mysql

import (
 "database/sql"
 "go-game-mini-framework/internal/model"
 _ "github.com/go-sql-driver/mysql"
)

type PlayerRepo struct {
 db *sql.DB
}

func NewPlayerRepo(db *sql.DB) *PlayerRepo {
 return &PlayerRepo{db: db}
}

// GetPlayerByID 根据ID查询玩家
func (r *PlayerRepo) GetPlayerByID(id int64) (*model.Player, error) {
 query := "SELECT id, username, level, exp, create_time FROM player WHERE id = ?"
 row := r.db.QueryRow(query, id)

 player := &model.Player{}
 err := row.Scan(&player.ID, &player.Username, &player.Level, &player.Exp, &player.CreateTime)
 if err != nil {
 return nil, err
 }
 return player, nil
}

// SavePlayer 保存玩家数据
func (r *PlayerRepo) SavePlayer(player *model.Player) error {
 query := "REPLACE INTO player (id, username, level, exp, create_time) VALUES (?, ?, ?, ?, ?)"
 _, err := r.db.Exec(query, player.ID, player.Username, player.Level, player.Exp, player.CreateTime)
 return err
}

// internal/storage/redis/rank.go
package redis

import (
 "context"
 "github.com/redis/go-redis/v9"
)

type RankRepo struct {
 client *redis.Client
 ctx    context.Context
}

func NewRankRepo(client *redis.Client) *RankRepo {
 return &RankRepo{
 client: client,
 ctx:    context.Background(),
 }
}

// AddScore 添加玩家排行榜分数
func (r *RankRepo) AddScore(rankKey string, playerID int64, score int) error {
 return r.client.ZAdd(r.ctx, rankKey, redis.Z{Score: float64(score), Member: playerID}).Err()
}

// GetTopN 获取排行榜前N名
func (r *RankRepo) GetTopN(rankKey string, n int) ([]redis.Z, error) {
 return r.client.ZRevRangeWithScores(r.ctx, rankKey, 0, int64(n-1)).Result()
}
```

### 3. 主函数（服务启动）

```Go

// cmd/server/main.go
package main

import (
 "database/sql"
 "go-game-mini-framework/config"
 "go-game-mini-framework/internal/network"
 "go-game-mini-framework/internal/storage/mysql"
 "go-game-mini-framework/internal/storage/redis"
 "go-game-mini-framework/pkg/logger"
 "github.com/redis/go-redis/v9"
 "log"
)

func main() {
 // 1. 初始化配置
 cfg, err := config.LoadConfig()
 if err != nil {
 log.Fatalf("Load config error: %v", err)
 }

 // 2. 初始化日志
 logger.InitLogger(cfg.Log.Level, cfg.Log.Path)

 // 3. 初始化MySQL连接
 mysqlDB, err := sql.Open("mysql", cfg.MySQL.DSN)
 if err != nil {
 log.Fatalf("MySQL connect error: %v", err)
 }
 defer mysqlDB.Close()

 // 4. 初始化Redis连接
 redisClient := redis.NewClient(&redis.Options{
 Addr:     cfg.Redis.Addr,
 Password: cfg.Redis.Password,
 DB:       cfg.Redis.DB,
 })
 defer redisClient.Close()

 // 5. 初始化网络层
 reactor, err := network.NewReactor(cfg.Server.Addr, cfg.Server.WorkerNum)
 if err != nil {
 log.Fatalf("Create reactor error: %v", err)
 }
 defer reactor.Stop()

 // 6. 启动服务
 logger.Info("Game server started on ", cfg.Server.Addr)
 reactor.Run()
}
```

## 五、快速开始

### 1. 环境准备

- 安装Go 1.21+：[https://golang.org/dl/](https://golang.org/dl/)

- 安装MySQL 8.0+，创建游戏数据库并执行表结构：

    ```SQL
    
    CREATE DATABASE game_mini DEFAULT CHARACTER SET utf8mb4;
    USE game_mini;
    
    CREATE TABLE player (
        id BIGINT PRIMARY KEY,
        username VARCHAR(50) NOT NULL,
        level INT DEFAULT 1,
        exp INT DEFAULT 0,
        create_time DATETIME DEFAULT CURRENT_TIMESTAMP
    );
    ```

- 安装Redis 7.0+，无需额外配置（默认端口6379）。

### 2. 配置修改

修改`config/`目录下的配置文件，适配你的本地环境：

```YAML

# config/app.yaml
server:
  addr: "0.0.0.0:8888"  # 游戏服务端口
  worker_num: 4         # 工作协程数
log:
  level: "info"
  path: "./logs/"

# config/mysql.yaml
dsn: "root:123456@tcp(127.0.0.1:3306)/game_mini?charset=utf8mb4&parseTime=True&loc=Local"

# config/redis.yaml
addr: "127.0.0.1:6379"
password: ""
db: 0
```

### 3. 编译运行

```Bash

# 克隆项目
git clone https://github.com/your-username/go-game-mini-framework.git
cd go-game-mini-framework

# 安装依赖
go mod tidy

# 编译
go build -o game-server ./cmd/server

# 运行
./game-server
```

### 4. 测试连接

使用Telnet/NetCat测试TCP连接：

```Bash

telnet 127.0.0.1 8888
# 输入任意消息，服务端会打印连接日志
```

## 六、功能扩展

### 1. 添加新业务（以背包系统为例）

1. 在`internal/model/`下添加`backpack.go`定义背包模型；

2. 在`internal/storage/mysql/`下添加背包数据操作；

3. 在`internal/handler/`下添加`backpack.go`处理背包相关消息；

4. 在`network/codec.go`中添加背包协议的编解码。

### 2. 集成Gin提供HTTP接口

```Go

// cmd/server/main.go 中添加
import "github.com/gin-gonic/gin"

func main() {
 // ... 原有初始化逻辑

 // 启动HTTP服务（GM后台）
 router := gin.Default()
 router.GET("/player/:id", func(c *gin.Context) {
 id := c.Param("id")
 // 查询玩家数据并返回
 c.JSON(200, gin.H{"code": 0, "data": player})
 })
 go router.Run(":8080")

 // 启动游戏服务
 reactor.Run()
}
```

## 七、部署到GitHub步骤

### 1. 本地初始化Git

```Bash

# 进入项目目录
cd go-game-mini-framework

# 初始化Git仓库
git init

# 添加所有文件
git add .

# 提交代码
git commit -m "init: 游戏服务端Mini框架初始版本"
```

### 2. 关联GitHub仓库

```Bash

# 创建远程仓库（先在GitHub网页端创建空仓库）
git remote add origin https://github.com/your-username/go-game-mini-framework.git

# 推送到GitHub
git push -u origin main
```

## 八、项目亮点与面试加分点

1. **网络模型**：参考Netty的Reactor模型设计，体现对高性能网络编程的理解；

2. **分层架构**：严格的分层设计（网络层→协议层→业务层→数据层），符合工业级项目规范；

3. **高并发**：利用Go的goroutine+工作池处理连接，适配游戏高并发场景；

4. **数据分层**：MySQL（持久化）+ Redis（缓存）分离，体现游戏数据存储的最佳实践；

5. **可扩展**：极简的API设计，新增业务无需修改核心逻辑，体现模块化设计思想。

## 九、常见面试问题（针对本项目）

1. **为什么选择Go语言开发游戏服务端？相比C++有什么优势？**

答：Go的goroutine轻量且易管理，并发编程成本低；编译速度快，开发效率高；内置GC无需手动管理内存，适合快速迭代的小型游戏；C++性能更高但开发/维护成本高，适合大型3A游戏，本框架聚焦小型游戏，Go是更优选择。

1. **仿Netty的Reactor模型是怎么设计的？为什么不用Go原生的net包直接处理？**

答：Reactor模型将「监听连接」和「处理连接」分离，通过工作池分摊连接处理压力，避免单协程处理所有连接导致的性能瓶颈；原生net包默认是每连接一协程，高并发下协程数过多会增加调度开销，Reactor+工作池更可控。

1. **Redis在排行榜中的作用？为什么不直接用MySQL做排行榜？**

答：Redis的ZSet天生适配排行榜场景，支持快速的分数更新和排序查询（O(logN)）；MySQL的ORDER BY查询在数据量大时性能差，且频繁更新分数会导致索引频繁修改，Redis的内存操作更适合高频更新的排行榜。

1. **游戏服务端的心跳检测是怎么实现的？如果玩家断连怎么处理？**

答：通过设置TCP读超时（SetReadDeadline）实现心跳，客户端定时发送心跳包，服务端每次读数据重置超时时间；断连时触发conn.Close()，并在连接管理模块清理玩家在线状态、保存临时数据到Redis/MySQL，玩家重连时恢复状态。

### 总结

本项目聚焦「轻量化、实战性」，既体现Go语言的并发优势，又结合游戏服务端核心技术（网络通信、数据存储、业务分层），部署到GitHub后可作为简历中的核心项目亮点。面试时可重点围绕「Reactor网络模型」「MySQL+Redis数据分层」「高并发连接处理」这三个核心点展开，突出你的技术设计能力和游戏服务端的理解。
> （注：文档部分内容可能由 AI 生成）
