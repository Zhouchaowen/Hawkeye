https://zhuanlan.zhihu.com/p/46201859

# 0.1.2

## 启动流程

```go
flag.Parse()
runtime.GOMAXPROCS(*goMaxProcs) // 设置核心
workerId // 生成workerId
cpuProfile // cpu相关
func NewNSQd(workerId int64, tcpAddr, httpAddr *net.TCPAddr .....) *NSQd // 构建 NSQd
	func (n *NSQd) idPump() // 启动go协程 id生成器
func (n *NSQd) Main() // 启动主线程Main
	func LookupRouter(lookupHosts []string, exitChan chan int)
		case <-ticker: // 检查lookup心跳
		case newChannel := <-notifyChannelChan: // 通知lookup创建了新Channel 
		case newTopic := <-notifyTopicChan: // 通知lookup创建了新Topic
			// 封装协议
			func (c *LookupPeer) Announce(topic string, channel string, port int) *ProtocolCommand
			func (w *LookupPeerWrapper) Command(cmd *nsq.ProtocolCommand) // 调用执行
				if !peer.IsConnected() // 判断是否连接到集群其它机器
					syncTopicChan <- w // 没有连接写入消息到 syncTopicChan
				func (c *ProtocolClient) WriteCommand(cmd *ProtocolCommand) error // 写入cmd到conn连接
				func (c *ProtocolClient) ReadResponse() ([]byte, error) // 读取返回结果
		case lookupPeer := <-syncTopicChan: // 发现有未连接的机器，重新全部通知一遍topicMap
		case <-exitChan:
	go util.TcpServer(tcpListener, tcpClientHandler)
		clientConn, err := listener.Accept()	// 循环接收消息
		go handler(clientConn)  // 处理连接
      func tcpClientHandler(clientConn net.Conn) // tcpClientHandler == handler
        func (c *ServerClient) Handle(protocols map[int32]Protocol) // 拦截方法
          err = binary.Read(c.conn, binary.BigEndian, &protocolVersion) // 获取协议版本
					// 对应协议的IOLoop
          func (p *ServerProtocolV2) IOLoop(client *nsq.ServerClient) error 
						// 赋值函数，用来调对应协议的对应方法
            func ProtocolExecute(p interface{}, client *ServerClient, params ...string)  
              func (p *ServerProtocolV2) SUB(client *nsq.ServerClient, params []string)
								func (n *NSQd) GetTopic(topicName string) *Topic // 通过请求参数获取Topic
								// 通过请求参数获取Channel
								func (t *Topic) GetChannel(channelName string) *Channel
									// 查看t.channelMap有没有channelName的channel，没有就new一个
									func NewChannel(t.name, channelName,....)
										// Channel.bacnkend 创建一个可持久化的存储对象
										NewDiskQueue(topicName+":"+channelName, dataPath, maxBytesPerFile)
										go channel.router() // 创建channel时，开启go协程监听该channel的请求消息。
											// 监听incomingMessageChan来的消息写入
											case msg = <-t.incomingMessageChan: 
                        case t.memoryMsgChan <- msg: // 有消息写入后转交给 memoryMsgChan
                        //如果memoryMsgChan满了，就写到Backend磁盘
                        default: WriteMessageToBackend(msg, t) 
                          msg.Encode() // 先将消息编码
                          t.BackendQueue().Put(data) // 调用对应的BackendQueue实现的Put
                            // 如DiskQueue实现的BackendQueue
                            func (d *DiskQueue) writeOne(data []byte) 
                    	case <-t.exitChan: // 退出
										// 从内存或后端读取消息并写入客户端输出通道,
										go channel.messagePump() 
											case msg = <-c.memoryMsgChan: // 获取memoryMsgChan[内存]的msg
											case buf = <-c.backend.ReadChan(): // 获取backend[磁盘]的buf
												msg, err = nsq.DecodeMessage(buf) // 经buf编码为msg
											case <-c.exitChan:// 退出
											c.pushInFlightMessage(msg) // 将消息加入inFlightMessages Map
											c.clientMessageChan <- msg // 将消息写入clientMessageChan
											go c.waitToAutoRequeue(msg) // ? 等等重排
										notify.Post("new_channel", c) // 写个通知给notifyChannelChan，有新chan创建
									messagePumpStarter.Do(func() { go t.MessagePump() }) // 执行一次MessagePump
										case msg = <-t.memoryMsgChan: // 获取memoryMsgChan[内存]的msg
										case buf = <-t.backend.ReadChan(): // 获取backend[磁盘]的buf,经buf编码为msg
										case <-t.exitChan:// 退出
										// 将msg发送到channelMap中的每一个channel
										for _, channel := range t.channelMap{} 
											nsq.NewMessage(msg.Id, msg.Body)
											go channel.PutMessage(chanMsg)
												c.incomingMessageChan <- msg
								// 添加client到channel.clients切片
								func (c *Channel) AddClient(client *nsq.ServerClient)
								client.SetState .... // 设置client.state
								go p.PushMessages(client)
									client.GetState(....) // 获取client的传输信息
									for
										if readyCount > 0 // 判断client端能接收几个消息
											case count := <-readyStateChan:
												client.SetState("ready_count", count) // 更新客服端可接收消息个数
											case msg := <-channel.clientMessageChan: // 接收服务文发送的消息
												client.SetState("ready_count", readyCount-1) // 设置客服端可接收消息个数
												data, err := msg.Encode() // 编码消息
												clientData, err := p.Frame(nsq.FrameTypeMessage, data) // 封装resp
												_, err = client.Write(clientData) //  发送
											case <-clientExitChan: // 退出
										else
											case count := <-readyStateChan:
												client.SetState("ready_count", count) // 更新客服端可接收消息个数
											case <-clientExitChan: // 退出
									channel.RemoveClient(client) // 从Channel的客户端列表中删除 ServerClient
              func (p *ServerProtocolV2) RDY(client *nsq.ServerClient, params []string)
              func (p *ServerProtocolV2) FIN(client *nsq.ServerClient, params []string) 
              func (p *ServerProtocolV2) REQ(client *nsq.ServerClient, params []string)
                    // ............
						//封装返回消息
            func (p *ServerProtocolV2) Frame(frameType int32, data []byte) ([]byte, error) 
	go HttpServer(httpListener)
		handler.HandleFunc("/ping", pingHandler) // 注册handleFunc
		handler.HandleFunc("/put", putHandler)
			func NewReqParams(req *http.Request) (*ReqParams, error) // 解析请求参数
			func (n *NSQd) GetTopic(topicName string) *Topic // 通过请求参数获取topic
				// 么有就创建一个Topic
				topic = NewTopic(topicName, n.memQueueSize, n.dataPath, n.maxBytesPerFile) 
					go topic.Router() // 创建Topic时，开启go协程监听该topic的请求消息。
						case msg = <-t.incomingMessageChan: // 监听incomingMessageChan来的消息写入
							case t.memoryMsgChan <- msg: // 有消息写入后转交给 memoryMsgChan
							default: WriteMessageToBackend(msg, t) //如果memoryMsgChan满了，就写到Backend磁盘
								msg.Encode() // 先将消息编码
								t.BackendQueue().Put(data) // 调用对应的BackendQueue实现的Put
									func (d *DiskQueue) writeOne(data []byte) // 如DiskQueue实现的BackendQueue
						case <-t.exitChan:
				notify.Post("new_topic", topic) // 并且写个通知给notifyTopicChan，有新topic创建
			msg := nsq.NewMessage(<-nsqd.idChan, reqParams.Body) // 封装message
			func (t *Topic) PutMessage(msg *nsq.Message) // 将message放入incomingMessageChan
				
		// ......
		handler.HandleFunc("/mput", mputHandler)
    handler.HandleFunc("/stats", statsHandler)
    handler.HandleFunc("/empty", emptyHandler)
func (n *NSQd) Exit() // 安全关闭服务
	tcpListener.Close()
	httpListener.Close()
	close(NSQd.exitChan) // 发送NSQd关闭channel，退出nsqd.idPump
	for 
		topic.Close()
			close(topic.exitChan) // 发送topic关闭channel，退出topic.Router和topic.MessagePump
			for 
				channel.Close()
					close(channel.exitChan)// 发送channel关闭channel，退出chan.Router和chan.MessagePump
					FlushQueue(channel) 
						for 
							case msg := <-q.MemoryChan(): // 将内存中的消息写到磁盘
								err := WriteMessageToBackend(msg, q)
						inFlight := q.InFlight() // 将InFlight中的消息写到磁盘
							for _, msg := range inFlight 
								err := WriteMessageToBackend(msg, q)	
					channel.backend.Close()
						// 通知diskqueue的channel关闭，退出diskQueue.readAheadPump
						close(diskQueue.exitChan) 
						func (d *DiskQueue) persistMetaData() error // 记录系统状态
			FlushQueue(topic)
			topic.backend.Close()
```

## handleFunc API

```go
HttpServer
	func putHandler(w http.ResponseWriter, req *http.Request) // http路由
		func NewReqParams(req *http.Request) (*ReqParams, error) // 解析请求参数
		func (r *ReqParams) Query(key string) (string, error) // topic查询参数
		func (n *NSQd) GetTopic(topicName string) *Topic // 通过topic名称，获取Topic对象
		func NewMessage(id []byte, body []byte) *Message // 封装传递message
		func (t *Topic) PutMessage(msg *nsq.Message) // 发送消息
			t.incomingMessageChan <- msg
		w.Header().Set("Content-Length", "2") // 返回
		io.WriteString(w, "OK")
```



# 0.2.17

- 带超时的请求

```go
func ApiRequest(endpoint string) (*simplejson.Json, error)
```

- command 表示从客户端到 NSQ 守护进程的命令

```go
IDENTIFY
REGISTER
UNREGISTER
PING
PUB
....
```

- LookupPeer 延迟连接到 nsqlookupd 并优雅地重新连接

```go
type LookupPeer struct {
	addr            string			
	conn            net.Conn
	state           int32
	connectCallback func(*LookupPeer)	// 连接回调 
	Info            PeerInfo		// 元数据
}

// 执行command请求
// 建立连接，交换请求协议MagicV1
// 发送cmd，读取返回响应
func (lp *LookupPeer) Command(cmd *Command) ([]byte, error) 
```

- Message 消息是包含 id、正文和元数据的基本数据类型

```go
type Message struct {
	Id        MessageID		// type MessageID [16]byte
	Body      []byte
	Timestamp int64
	Attempts  uint16
}
```

- Protocol 消息协议，包含编解码

```go
// 各版本协议都要实现 IOLoop()
type Protocol interface {
	IOLoop(conn net.Conn) error
}
```

- Channel 
  - Channel 表示 NSQ 通道的具体类型（并且还实现了 Queue 接口）
  - 每个主题可以有多个频道，每个频道都有自己独特的订阅者（客户端）集。
  - 通道维护所有客户端和消息元数据，编排传输中的消息、超时、重新排队等。

```go
type Channel struct

// 创建一个channel
func NewChannel(topicName string, channelName string, options *nsqdOptions,
	notifier Notifier, deleteCallback func(*Channel)) *Channel
  c.initPQ()			// 初始化优先队列
  c.backend = NewDiskQueue   // 初始化持久化存储
  go c.messagePump()			// 启动消息泵
  c.waitGroup.Wrap(func() { c.router() })		// 启动消息路由，写入内存通道或持久化
  c.waitGroup.Wrap(func() { c.deferredWorker() })
  c.waitGroup.Wrap(func() { c.inFlightWorker() })
  go notifier.Notify(c) // 监听消息通知 
```











https://xiaomi-info.github.io/2019/12/06/nsq-src/

https://jiacyer.com/2020/07/31/youzan-nsq/





```bash
with \
		dictGetUInt32('geoip_city_blocks_ipv4', 'geoname_id', tuple(IPv4StringToNum(ip))) AS geoname_id \
	select \
		ip, \
		count, \
		dictGetString('geoip_city_locations_cn', 'country_iso_code', toUInt64(geoname_id)) AS country_iso_code, \
		dictGetString('geoip_city_locations_cn', 'country_name', toUInt64(geoname_id)) AS country_name, \
		dictGetString('geoip_city_locations_cn', 'city_name', toUInt64(geoname_id)) AS city_name, \
		dictGetFloat32('geoip_city_blocks_ipv4', 'latitude', tuple(IPv4StringToNum(ip))) AS latitude, \
		dictGetFloat32('geoip_city_blocks_ipv4', 'longitude', tuple(IPv4StringToNum(ip))) AS longitude \
	from ( \
SELECT \
    rdata AS ip, \
    sum(count) AS count \
FROM mv_attack_situation \
WHERE (user_id = '479284b1401da21f80b61bf60ce08ad6') AND (IPv4StringToNum(rdata) > 0) \
GROUP BY rdata \
ORDER BY count DESC \
LIMIT 5 )
```

```sql
`user_id` String, `client_id` String, `qtime` DateTime, `client_ip` String, `view` String, `domain` String, `rtype` UInt16, `rdata` Array(String), `rcode` UInt8, `query_delay` Int32, `score` UInt8, `policy` UInt8, `category` UInt8, `snapshot` String

INSERT INTO query_logs values('4c4f76a2408fc9e080b1728e21d521fc','56557a614072a55c809968cd806a03c3',1641437917,'12.55.67.10','default','1xbetgov.com',1,'[]',0,390089,45,1,4,'')

4c4f76a2408fc9e080b1728e21d521fc,56557a614072a55c809968cd806a03c3,1641437917,12.55.67.10,default,1xbetgov.com,1,"['']",0,390089,45,1,4,06-Jan-2022 10:58:37.661 client 12.55.67.10 47798: view default: 1xbetgov.com IN A NOERROR + NS NE NT ND NC NH 45 1 4 45.000000 W 390089

c9daf597405f09f2802bde474b855176,69b0fcae407412b580a7cbdbe61b8d58,1641437917,12.55.67.10,default,1xbetgov.com,1,"['']",0,390089,45,1,4,06-Jan-2022 10:58:37.661 client 12.55.67.10 47798: view default: 1xbetgov.com IN A NOERROR + NS NE NT ND NC NH 45 1 4 45.000000 W 390089



```

```go

```



















