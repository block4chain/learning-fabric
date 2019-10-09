---
description: 通信层维护底层的GRPC连接，处理数据的接收和发送
---

# 通信层

## 服务定义

通信层提供一个GRPC服务，定义如下:

{% code-tabs %}
{% code-tabs-item title="protos/gossip/message.pb.go" %}
```go
// GossipServer is the server API for Gossip service.
type GossipServer interface {
	// GossipStream is the gRPC stream used for sending and receiving messages
	GossipStream(Gossip_GossipStreamServer) error
	// Ping is used to probe a remote peer's aliveness
	Ping(context.Context, *Empty) (*Empty, error)
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

## 服务实现

`comm.commImpl`是通信层服务的实现

{% code-tabs %}
{% code-tabs-item title="gossip/comm/comm\_impl.go" %}
```go
type commImpl struct {
	sa             api.SecurityAdvisor
	tlsCerts       *common.TLSCertificates
	pubSub         *util.PubSub
	peerIdentity   api.PeerIdentityType
	idMapper       identity.Mapper
	logger         util.Logger
	opts           []grpc.DialOption
	secureDialOpts func() []grpc.DialOption
	connStore      *connectionStore
	PKIID          []byte
	deadEndpoints  chan common.PKIidType
	msgPublisher   *ChannelDeMultiplexer
	lock           *sync.Mutex
	exitChan       chan struct{}
	stopWG         sync.WaitGroup
	subscriptions  []chan proto.ReceivedMessage
	stopping       int32
	metrics        *metrics.CommMetrics
	dialTimeout    time.Duration
	connTimeout    time.Duration
	recvBuffSize   int
	sendBuffSize   int
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

### `GossipStream`方法

{% code-tabs %}
{% code-tabs-item title="gossip/comm/comm\_impl.go" %}
```go
func (c *commImpl) GossipStream(stream proto.Gossip_GossipStreamServer) error {
	if c.isStopping() {
		return fmt.Errorf("Shutting down")
	}
	connInfo, err := c.authenticateRemotePeer(stream, false)
	if err != nil {
		c.logger.Errorf("Authentication failed: %v", err)
		return err
	}
	c.logger.Debug("Servicing", extractRemoteAddress(stream))

	conn := c.connStore.onConnected(stream, connInfo, c.metrics)

	h := func(m *proto.SignedGossipMessage) {
		c.msgPublisher.DeMultiplex(&ReceivedMessageImpl{
			conn:                conn,
			lock:                conn,
			SignedGossipMessage: m,
			connInfo:            connInfo,
		})
	}
	conn.handler = interceptAcks(h, connInfo.ID, c.pubSub)
	defer func() {
		c.logger.Debug("Client", extractRemoteAddress(stream), " disconnected")
		c.connStore.closeByPKIid(connInfo.ID)
		conn.close()
	}()
	return conn.serviceConnection()
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

### `Ping`方法

{% code-tabs %}
{% code-tabs-item title="gossip/comm/comm\_impl.go" %}
```go
func (c *commImpl) Ping(context.Context, *proto.Empty) (*proto.Empty, error) {
	return &proto.Empty{}, nil
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}



