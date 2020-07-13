---
description: 服务层实现Gossip协议，并为上层提供节点发现、数据交换等服务
---

# 服务层

## 服务定义

{% code title="gossip/gossip/gossip.go" %}
```go
type Gossip interface {
	//channel相关
	JoinChan(joinMsg api.JoinChannelMessage, chainID common.ChainID).
	LeaveChan(chainID common.ChainID)
	SelfChannelInfo(common.ChainID) *proto.SignedGossipMessage
	UpdateLedgerHeight(height uint64, chainID common.ChainID)
	UpdateChaincodes(chaincode []*proto.Chaincode, chainID common.ChainID)
	PeerFilter(channel common.ChainID, messagePredicate api.SubChannelSelectionCriteria) (filter.RoutingFilter, error)
	
	//member相关
	SelfMembershipInfo() discovery.NetworkMember
	Peers() []discovery.NetworkMember
	PeersOfChannel(common.ChainID) []discovery.NetworkMember
	UpdateMetadata(metadata []byte)
	SuspectPeers(s api.PeerSuspector)
	IdentityInfo() api.PeerIdentitySet
	
	//消息
	Send(msg *proto.GossipMessage, peers ...*comm.RemotePeer)
	SendByCriteria(*proto.SignedGossipMessage, SendCriteria) error
	Gossip(msg *proto.GossipMessage)
	
	//其它
	Accept(acceptor common.MessageAcceptor, passThrough bool) (<-chan *proto.GossipMessage, <-chan proto.ReceivedMessage)
	Stop()
}
```
{% endcode %}

## 服务实现

## 通道管理

### 加入退出通道

### 节点发现

## 消息发送

## 领导选举

## 状态同步

## 区块同步

### 区块分发

### 区块拉取

## 帐本维护

### 公开帐本

### 隐私状态



