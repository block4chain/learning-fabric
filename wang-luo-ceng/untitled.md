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
	//第一次连接，需要进行认证
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

### 其它方法

```go
func (c *commImpl) SetDialOpts(opts ...grpc.DialOption)
func (c *commImpl) Send(msg *proto.SignedGossipMessage, peers ...*RemotePeer)
func (c *commImpl) Probe(remotePeer *RemotePeer) error
func (c *commImpl) Handshake(remotePeer *RemotePeer) (api.PeerIdentityType, error)
func (c *commImpl) Accept(acceptor common.MessageAcceptor) <-chan proto.ReceivedMessage
func (c *commImpl) PresumedDead() <-chan common.PKIidType
func (c *commImpl) CloseConn(peer *RemotePeer)
func (c *commImpl) Stop() 
func (c *commImpl) GetPKIid() common.PKIidType
func (c *commImpl) SendWithAck(msg *proto.SignedGossipMessage, timeout time.Duration, minAck int, peers ...*RemotePeer) AggregatedSendResult
```

## 连接握手

Fabric网络中的节点间可以建立点对点连接，在建立连接前需要完成一次节点身份认证。

![](../.gitbook/assets/gossip_authentication.png)

握手时一个节点与另外一个节点建立GRPC连接，在双方交换身份信息前，发起方会通过Ping命令检测对方是否在线。

{% code-tabs %}
{% code-tabs-item title="gossip/comm/comm\_impl.go" %}
```go
func (c *commImpl) Handshake(remotePeer *RemotePeer) (api.PeerIdentityType, error) {
	var dialOpts []grpc.DialOption
	dialOpts = append(dialOpts, c.secureDialOpts()...)
	dialOpts = append(dialOpts, grpc.WithBlock())
	dialOpts = append(dialOpts, c.opts...)
	ctx := context.Background()
	ctx, cancel := context.WithTimeout(ctx, c.dialTimeout)
	defer cancel()
	cc, err := grpc.DialContext(ctx, remotePeer.Endpoint, dialOpts...)
	if err != nil {
		return nil, err
	}
	defer cc.Close()

    //建立GRPC连接
	cl := proto.NewGossipClient(cc)
	ctx, cancel = context.WithTimeout(context.Background(), DefConnTimeout)
	defer cancel()
	//探测节点是否存活
	if _, err = cl.Ping(ctx, &proto.Empty{}); err != nil {
		return nil, err
	}
	ctx, cancel = context.WithTimeout(context.Background(), handshakeTimeout)
	defer cancel()
	//发起连接建立请求
	stream, err := cl.GossipStream(ctx)
	if err != nil {
		return nil, err
	}
	//节点之间交换身份信息，并认证
	connInfo, err := c.authenticateRemotePeer(stream, true)
	if err != nil {
		c.logger.Warningf("Authentication failed: %v", err)
		return nil, err
	}
	if len(remotePeer.PKIID) > 0 && !bytes.Equal(connInfo.ID, remotePeer.PKIID) {
		return nil, fmt.Errorf("PKI-ID of remote peer doesn't match expected PKI-ID")
	}
	return connInfo.Identity, nil
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

在确认对方在线后，双方互相发送一个[连接建立消息](https://app.gitbook.com/@me020523/s/fabric/~/drafts/-LqoydU-0k79xp9FJWFw/primary/wang-luo-ceng/gossip-xie-yi#lian-jie-jian-li-xiao-xi)，消息中包含身份信息，在收到对方的身份信息后，节点会对身份信息进行检验。

{% code-tabs %}
{% code-tabs-item title="gossip/comm/comm\_impl.go" %}
```go
func (c *commImpl) authenticateRemotePeer(stream stream, initiator bool) (*proto.ConnectionInfo, error) {
	ctx := stream.Context()
	remoteAddress := extractRemoteAddress(stream)  //获取对方IP地址
	remoteCertHash := extractCertificateHashFromContext(ctx) //获取对方Tls证书的sha256 hash值
	var err error
	var cMsg *proto.SignedGossipMessage
	useTLS := c.tlsCerts != nil
	var selfCertHash []byte
	if useTLS {
		certReference := c.tlsCerts.TLSServerCert
		if initiator {
			certReference = c.tlsCerts.TLSClientCert
		}
		//计算自己的tls证书sha265 hash值
		selfCertHash = certHashFromRawCert(certReference.Load().(*tls.Certificate).Certificate[0])
	}
	signer := func(msg []byte) ([]byte, error) {
		return c.idMapper.Sign(msg)
	}
	// TLS enabled but not detected on other side
	if useTLS && len(remoteCertHash) == 0 {
		return nil, fmt.Errorf("No TLS certificate")
	}
	//创建连接建立消息
	cMsg, err = c.createConnectionMsg(c.PKIID, selfCertHash, c.peerIdentity, signer)
	if err != nil {
		return nil, err
	}
	stream.Send(cMsg.Envelope) //将消息发送给对方
	m, err := readWithTimeout(stream, c.connTimeout, remoteAddress) //获取对方的连接建立消息
	if err != nil {
		return nil, err
	}
	receivedMsg := m.GetConn()
	if receivedMsg == nil {
		return nil, fmt.Errorf("Wrong type")
	}
	//对方的PKIID必须不为空
	if receivedMsg.PkiId == nil {
		c.logger.Warningf("%s didn't send a pkiID", remoteAddress)
		return nil, fmt.Errorf("No PKI-ID")
	}
	err = c.idMapper.Put(receivedMsg.PkiId, receivedMsg.Identity)
	if err != nil {
		return nil, err
	}

	connInfo := &proto.ConnectionInfo{
		ID:       receivedMsg.PkiId,
		Identity: receivedMsg.Identity,
		Endpoint: remoteAddress,
		Auth: &proto.AuthInfo{
			Signature:  m.Signature,
			SignedData: m.Payload,
		},
	}
	if useTLS {
	    //保证对方的发送TLS证书与建立TLS连接使用的证书一致
		if !bytes.Equal(remoteCertHash, receivedMsg.TlsCertHash) {
			return nil, errors.Errorf("Expected %v in remote hash of TLS cert, but got %v", remoteCertHash, receivedMsg.TlsCertHash)
		}
	}
	// 确保发送消息来源没有被中间人拦截
	verifier := func(peerIdentity []byte, signature, message []byte) error {
		pkiID := c.idMapper.GetPKIidOfCert(api.PeerIdentityType(peerIdentity))
		return c.idMapper.Verify(pkiID, signature, message)
	}
	err = m.Verify(receivedMsg.Identity, verifier)
	if err != nil {
		return nil, err
	}
	return connInfo, nil
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

节点身份验证通过后，双方握手成功。连接握手的目的是为了对远端节点的身份进行确认。

## 身份管理

### PKIID

Fabric网络中的每一个节点都有一个唯一标识: _**PKIID**_.

{% code-tabs %}
{% code-tabs-item title="peer/gossip/mcs.go" %}
```go
func (s *MSPMessageCryptoService) GetPKIidOfCert(peerIdentity api.PeerIdentityType) common.PKIidType {
	// Validate arguments
	if len(peerIdentity) == 0 {
		mcsLogger.Error("Invalid Peer Identity. It must be different from nil.")
		return nil
	}
	sid, err := s.deserializer.Deserialize(peerIdentity)
	if err != nil {
		return nil
	}
	// concatenate msp-id and idbytes
	// idbytes is the low-level representation of an identity.
	// it is supposed to be already in its minimal representation

	mspIdRaw := []byte(sid.Mspid)
	raw := append(mspIdRaw, sid.IdBytes...)

	// Hash
	digest, err := factory.GetDefault().Hash(raw, &bccsp.SHA256Opts{})
	if err != nil {
		return nil
	}
	return digest
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

PKIID是节点身份和节点MSP ID的sha256 hash值。

### 身份存储

两个节点握手成功后，节点会将对方PKIID和对方身份存放在`identity.Mapper`中，方便后续使用。`identity.Mapper`是一个接口，定义如下:

{% code-tabs %}
{% code-tabs-item title="gossip/identity/identity.go" %}
```go
type Mapper interface {
	Put(pkiID common.PKIidType, identity api.PeerIdentityType) error
	Get(pkiID common.PKIidType) (api.PeerIdentityType, error)
	Sign(msg []byte) ([]byte, error)
	Verify(vkID, signature, message []byte) error
	GetPKIidOfCert(api.PeerIdentityType) common.PKIidType
	SuspectPeers(isSuspected api.PeerSuspector)
	IdentityInfo() api.PeerIdentitySet
	Stop()
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

该接口存在一个唯一实现`identity.identityMapperImpl`

{% code-tabs %}
{% code-tabs-item title="gossip/identity/identity.go" %}
```go
type identityMapperImpl struct {
	onPurge    purgeTrigger
	mcs        api.MessageCryptoService
	sa         api.SecurityAdvisor
	pkiID2Cert map[string]*storedIdentity  //节点PKIID与节点身份映射
	sync.RWMutex
	stopChan chan struct{}
	sync.Once
	selfPKIID string
}
//节点身份存储结构
type storedIdentity struct {
	pkiID           common.PKIidType
	lastAccessTime  int64
	peerIdentity    api.PeerIdentityType
	orgId           api.OrgIdentityType  //节点所在的组织
	expirationTimer *time.Timer  //身份过期计时器
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

### 身份清理

`identity.identityMapperImpl`提供了一些机制用于清理无用或过期的身份信息。

新建`identity.identityMapperImpl`实例时，peer节点会启动一个goroutine定期清理身份

{% code-tabs %}
{% code-tabs-item title="gossip/identity/identity.go" %}
```go
func NewIdentityMapper(mcs api.MessageCryptoService, selfIdentity api.PeerIdentityType, onPurge purgeTrigger, sa api.SecurityAdvisor) Mapper {
	//....
	go idMapper.periodicalPurgeUnusedIdentities()
	//....
}

func (is *identityMapperImpl) periodicalPurgeUnusedIdentities() {
	//超过usageTh时间没有使用用的身份会被清理，默认是一个小时
	usageTh := GetIdentityUsageThreshold()
	for {
		select {
		case <-is.stopChan:
			return
		case <-time.After(usageTh / 10):
			is.SuspectPeers(func(_ api.PeerIdentityType) bool {
				return false
			})
		}
	}
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

清理身份的goroutine会定期按规则选择出待清理的身份:

```go
func (is *identityMapperImpl) SuspectPeers(isSuspected api.PeerSuspector) {
	for _, identity := range is.validateIdentities(isSuspected) {
		identity.cancelExpirationTimer()
		is.delete(identity.pkiID, identity.peerIdentity)
	}
}
func (is *identityMapperImpl) validateIdentities(isSuspected api.PeerSuspector) []*storedIdentity {
	now := time.Now()
	usageTh := GetIdentityUsageThreshold()
	is.RLock()
	defer is.RUnlock()
	var revokedIdentities []*storedIdentity
	for pkiID, storedIdentity := range is.pkiID2Cert {
		//最近一段时间没有访问过的身份
		if pkiID != is.selfPKIID && storedIdentity.fetchLastAccessTime().Add(usageTh).Before(now) {
			revokedIdentities = append(revokedIdentities, storedIdentity)
			continue
		}
		//满足一定规则的身份
		if !isSuspected(storedIdentity.peerIdentity) {
			continue
		}
		//校验不通过的身份
		if err := is.mcs.ValidateIdentity(storedIdentity.fetchIdentity()); err != nil {
			revokedIdentities = append(revokedIdentities, storedIdentity)
		}
	}
	return revokedIdentities
}
```

节点身份也会过期，所以在添加节点身份时，会自动注册一个定时器，当节点身份过期时自己删除已经过期的身份

```go
func (is *identityMapperImpl) Put(pkiID common.PKIidType, identity api.PeerIdentityType) error {
    //省略一些代码
	var expirationTimer *time.Timer
	if !expirationDate.IsZero() {
		if time.Now().After(expirationDate) {
			return errors.New("identity expired")
		}
		//注册过期自动删除计时器
		timeToLive := expirationDate.Add(time.Millisecond).Sub(time.Now())
		expirationTimer = time.AfterFunc(timeToLive, func() {
			is.delete(pkiID, identity)
		})
	}
	is.pkiID2Cert[string(id)] = newStoredIdentity(pkiID, identity, expirationTimer, is.sa.OrgByPeerIdentity(identity))
	return nil
}
```

`identity.identityMapperImpl`也可以通过调用`SuspectPeers`方法手动清理身份



