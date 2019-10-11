---
description: Gossip节点间完成数据通信的数据协议
---

# Gossip协议

## 消息协议

Gossip节点间通过一个消息协议完成数据交换，根据消息是否被签名，消息协议被分为两类

* **原始消息协议**

{% code-tabs %}
{% code-tabs-item title="protos/gossip/message.pb.go" %}
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
{% endcode-tabs-item %}

{% code-tabs-item title="protos/gossip/message.pb" %}
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
{% endcode-tabs-item %}
{% endcode-tabs %}

* **已签名消息协议**

{% code-tabs %}
{% code-tabs-item title="protos/gossip/extensions.go" %}
```go
type SignedGossipMessage struct {
	*Envelope
	*GossipMessage
}
```
{% endcode-tabs-item %}

{% code-tabs-item title="protos/gossip/message.pb.go" %}
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
{% endcode-tabs-item %}
{% endcode-tabs %}

## 消息签名

为了保证消息没有被篡改需要对消息进行签名，签名实现方法如下:

{% code-tabs %}
{% code-tabs-item title="protos/gossip/extensions.go" %}
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
{% endcode-tabs-item %}
{% endcode-tabs %}

## 消息内容

消息协议提供了一个通用的协议结构，为满足不同的应用场景，消息的负载数据Content类型是不定的。

### ACK消息

ACK消息用于对接收到的消息进行确认回复

{% code-tabs %}
{% code-tabs-item title="protos/gossip/message.pb.go" %}
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
{% endcode-tabs-item %}
{% endcode-tabs %}

### 连接建立消息

连接建立消息用于在节点间建立Gossip点对点连接

{% code-tabs %}
{% code-tabs-item title="protos/gossip/message.pb.go" %}
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
{% endcode-tabs-item %}
{% endcode-tabs %}

