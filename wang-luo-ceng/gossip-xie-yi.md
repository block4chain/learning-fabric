---
description: Gossip节点间完成数据通信的数据协议
---

# 协议

## 消息协议

Gossip节点间通过一个消息协议完成数据交换，根据消息是否被签名，消息协议被分为两类

* **原始消息协议**

{% tabs %}
{% tab title="protos/gossip/message.pb.go" %}
```go
type GossipMessage struct {
	//暂未使用
	Nonce uint64 `protobuf:"varint,1,opt,name=nonce,proto3" json:"nonce,omitempty"`
	Channel []byte `protobuf:"bytes,2,opt,name=channel,proto3" json:"channel,omitempty"`
	//对消息进行标记，方便对消息进行差异化处理
	Tag GossipMessage_Tag `protobuf:"varint,3,opt,name=tag,proto3,enum=gossip.GossipMessage_Tag" json:"tag,omitempty"`
	//根据不同的使用场景，消息的负载内容可能不一样
	Content              isGossipMessage_Content `protobuf_oneof:"content"`
	XXX_NoUnkeyedLiteral struct{}                `json:"-"`
	XXX_unrecognized     []byte                  `json:"-"`
	XXX_sizecache        int32                   `json:"-"`
}
```
{% endtab %}

{% tab title="protos/gossip/message.pb" %}
```c
message GossipMessage {
    uint64 nonce  = 1;
    bytes channel = 2;

    enum Tag {
        UNDEFINED    = 0;
        EMPTY        = 1;
        ORG_ONLY     = 2;
        CHAN_ONLY    = 3;
        CHAN_AND_ORG = 4;
        CHAN_OR_ORG  = 5;
    }
    
    Tag tag = 3;

    oneof content {
        // Membership
        AliveMessage alive_msg = 5;
        MembershipRequest mem_req = 6;
        MembershipResponse mem_res = 7;

        // Contains a ledger block
        DataMessage data_msg = 8;

        // Used for push&pull
        GossipHello hello = 9;
        DataDigest  data_dig = 10;
        DataRequest data_req = 11;
        DataUpdate  data_update = 12;

        // Empty message, used for pinging
        Empty empty = 13;

        // ConnEstablish, used for establishing a connection
        ConnEstablish conn = 14;

        // Used for relaying information
        // about state
        StateInfo state_info = 15;

        // Used for sending sets of StateInfo messages
        StateInfoSnapshot state_snapshot = 16;

        // Used for asking for StateInfoSnapshots
        StateInfoPullRequest state_info_pull_req = 17;

        //  Used to ask from a remote peer a set of blocks
        RemoteStateRequest state_request = 18;

        // Used to send a set of blocks to a remote peer
        RemoteStateResponse state_response = 19;

        // Used to indicate intent of peer to become leader
        LeadershipMessage leadership_msg = 20;

        // Used to learn of a peer's certificate
        PeerIdentity peer_identity = 21;

        Acknowledgement ack = 22;

        // Used to request private data
        RemotePvtDataRequest privateReq = 23;

        // Used to respond to private data requests
        RemotePvtDataResponse privateRes = 24;

        // Encapsulates private data used to distribute
        // private rwset after the endorsement
        PrivateDataMessage private_data = 25;
    }
}
```
{% endtab %}
{% endtabs %}

* **已签名消息协议**

{% tabs %}
{% tab title="protos/gossip/extensions.go" %}
```go
type SignedGossipMessage struct {
	*Envelope
	*GossipMessage
}
```
{% endtab %}

{% tab title="protos/gossip/message.pb.go" %}
```go
type Envelope struct {
	Payload              []byte          `protobuf:"bytes,1,opt,name=payload,proto3" json:"payload,omitempty"`
	Signature            []byte          `protobuf:"bytes,2,opt,name=signature,proto3" json:"signature,omitempty"`
	SecretEnvelope       *SecretEnvelope `protobuf:"bytes,3,opt,name=secret_envelope,json=secretEnvelope,proto3" json:"secret_envelope,omitempty"`
	XXX_NoUnkeyedLiteral struct{}        `json:"-"`
	XXX_unrecognized     []byte          `json:"-"`
	XXX_sizecache        int32           `json:"-"`
}

type SecretEnvelope struct {
	Payload              []byte   `protobuf:"bytes,1,opt,name=payload,proto3" json:"payload,omitempty"`
	Signature            []byte   `protobuf:"bytes,2,opt,name=signature,proto3" json:"signature,omitempty"`
	XXX_NoUnkeyedLiteral struct{} `json:"-"`
	XXX_unrecognized     []byte   `json:"-"`
	XXX_sizecache        int32    `json:"-"`
}
```
{% endtab %}
{% endtabs %}

## 消息签名

为了保证消息没有被篡改需要对消息进行签名，签名实现方法如下:

{% code title="protos/gossip/extensions.go" %}
```go
func (m *SignedGossipMessage) Sign(signer Signer) (*Envelope, error) {
	var secretEnvelope *SecretEnvelope
	if m.Envelope != nil {
		//待签名消息存在一个安全负载
		secretEnvelope = m.Envelope.SecretEnvelope
	}
	m.Envelope = nil
	payload, err := proto.Marshal(m.GossipMessage)  //对原始Gossip消息进行序列化
	if err != nil {
		return nil, err
	}
	sig, err := signer(payload)  //对序列化后的消息进行签名
	if err != nil {
		return nil, err
	}
	e := &Envelope{
		Payload:        payload,
		Signature:      sig,
		SecretEnvelope: secretEnvelope,
	}
	m.Envelope = e
	return e, nil
}
```
{% endcode %}

## 消息内容

消息协议提供了一个通用的协议结构，为满足不同的应用场景，消息的负载数据Content类型是不定的。

### ACK消息

ACK消息用于对接收到的消息进行确认回复

{% code title="protos/gossip/message.pb.go" %}
```go
type GossipMessage_Ack struct {
	Ack *Acknowledgement `protobuf:"bytes,22,opt,name=ack,proto3,oneof"`
}

type Acknowledgement struct {
	Error                string   `protobuf:"bytes,1,opt,name=error,proto3" json:"error,omitempty"`
	XXX_NoUnkeyedLiteral struct{} `json:"-"`
	XXX_unrecognized     []byte   `json:"-"`
	XXX_sizecache        int32    `json:"-"`
}
```
{% endcode %}

### 连接建立消息

连接建立消息用于在节点间建立Gossip点对点连接

{% code title="protos/gossip/message.pb.go" %}
```go
type GossipMessage_Conn struct {
	Conn *ConnEstablish `protobuf:"bytes,14,opt,name=conn,proto3,oneof"`
}
type ConnEstablish struct {
	PkiId                []byte   `protobuf:"bytes,1,opt,name=pki_id,json=pkiId,proto3" json:"pki_id,omitempty"`
	Identity             []byte   `protobuf:"bytes,2,opt,name=identity,proto3" json:"identity,omitempty"`
	TlsCertHash          []byte   `protobuf:"bytes,3,opt,name=tls_cert_hash,json=tlsCertHash,proto3" json:"tls_cert_hash,omitempty"`
	XXX_NoUnkeyedLiteral struct{} `json:"-"`
	XXX_unrecognized     []byte   `json:"-"`
	XXX_sizecache        int32    `json:"-"`
}
```
{% endcode %}

### 节点状态消息

节点状态消息用于向网络其它节点发布自己的状态:

* 帐本区块高度
* 是否离开通道
* 安装的链码集合

{% code title="protos/gossip/message.pb.go" %}
```go
type GossipMessage_StateInfo struct {
	StateInfo *StateInfo `protobuf:"bytes,15,opt,name=state_info,json=stateInfo,proto3,oneof"`
}
type StateInfo struct {
	Timestamp *PeerTime `protobuf:"bytes,2,opt,name=timestamp,proto3" json:"timestamp,omitempty"`
	PkiId     []byte    `protobuf:"bytes,3,opt,name=pki_id,json=pkiId,proto3" json:"pki_id,omitempty"`
	Channel_MAC          []byte      `protobuf:"bytes,4,opt,name=channel_MAC,json=channelMAC,proto3" json:"channel_MAC,omitempty"`
	Properties           *Properties `protobuf:"bytes,5,opt,name=properties,proto3" json:"properties,omitempty"`
	XXX_NoUnkeyedLiteral struct{}    `json:"-"`
	XXX_unrecognized     []byte      `json:"-"`
	XXX_sizecache        int32       `json:"-"`
}
//状态属性
type Properties struct {
	LedgerHeight         uint64       `protobuf:"varint,1,opt,name=ledger_height,json=ledgerHeight,proto3" json:"ledger_height,omitempty"`
	LeftChannel          bool         `protobuf:"varint,2,opt,name=left_channel,json=leftChannel,proto3" json:"left_channel,omitempty"`
	Chaincodes           []*Chaincode `protobuf:"bytes,3,rep,name=chaincodes,proto3" json:"chaincodes,omitempty"`
	XXX_NoUnkeyedLiteral struct{}     `json:"-"`
	XXX_unrecognized     []byte       `json:"-"`
	XXX_sizecache        int32        `json:"-"`
}
```
{% endcode %}

### 节点状态信息集合消息

节点通过这个消息向通道内其它节点广播自己从网络上学习到的其它节点的状态信息集合

{% code title="protos/gossip/message.pb.go" %}
```go
type GossipMessage_StateSnapshot struct {
	StateSnapshot *StateInfoSnapshot `protobuf:"bytes,16,opt,name=state_snapshot,json=stateSnapshot,proto3,oneof"`
}

type StateInfoSnapshot struct {
	Elements             []*Envelope `protobuf:"bytes,1,rep,name=elements,proto3" json:"elements,omitempty"`
	XXX_NoUnkeyedLiteral struct{}    `json:"-"`
	XXX_unrecognized     []byte      `json:"-"`
	XXX_sizecache        int32       `json:"-"`
}
```
{% endcode %}

### 节点状态信息集合请求消息

节点通过这个消息向通道内其它节点请求对方的状态消息集合

{% code title="protos/gossip/message.pb.go" %}
```go
type GossipMessage_StateInfoPullReq struct {
	StateInfoPullReq *StateInfoPullRequest `protobuf:"bytes,17,opt,name=state_info_pull_req,json=stateInfoPullReq,proto3,oneof"`
}

type StateInfoPullRequest struct {
	// channel_MAC is an authentication code that proves
	// that the peer that sent this message knows
	// the name of the channel.
	Channel_MAC          []byte   `protobuf:"bytes,1,opt,name=channel_MAC,json=channelMAC,proto3" json:"channel_MAC,omitempty"`
	XXX_NoUnkeyedLiteral struct{} `json:"-"`
	XXX_unrecognized     []byte   `json:"-"`
	XXX_sizecache        int32    `json:"-"`
}
```
{% endcode %}

