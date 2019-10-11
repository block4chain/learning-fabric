---
description: 服务层实现Gossip协议，并为上层提供节点发现、数据交换等服务
---

# 服务层

## 服务定义

{% code-tabs %}
{% code-tabs-item title="gossip/gossip/gossip.go" %}
```go
type Gossip interface {
	SelfMembershipInfo() discovery.NetworkMember
	SelfChannelInfo(common.ChainID) *proto.SignedGossipMessage
	Send(msg *proto.GossipMessage, peers ...*comm.RemotePeer)
	SendByCriteria(*proto.SignedGossipMessage, SendCriteria) error
	Peers() []discovery.NetworkMember
	PeersOfChannel(common.ChainID) []discovery.NetworkMember
	UpdateMetadata(metadata []byte)
	UpdateLedgerHeight(height uint64, chainID common.ChainID)
	UpdateChaincodes(chaincode []*proto.Chaincode, chainID common.ChainID)
	Gossip(msg *proto.GossipMessage)
	PeerFilter(channel common.ChainID, messagePredicate api.SubChannelSelectionCriteria) (filter.RoutingFilter, error)
	Accept(acceptor common.MessageAcceptor, passThrough bool) (<-chan *proto.GossipMessage, <-chan proto.ReceivedMessage)
	JoinChan(joinMsg api.JoinChannelMessage, chainID common.ChainID).
	LeaveChan(chainID common.ChainID)
	SuspectPeers(s api.PeerSuspector)
	IdentityInfo() api.PeerIdentitySet
	Stop()
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

