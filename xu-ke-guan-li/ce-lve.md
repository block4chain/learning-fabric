# 策略

## 类型

Fabric有两种策略类型

* Signature策略

Signature策略通过运算符:

* `OR`
* `AND`
* `NOutOf`

对一组主体身份\(MSP Principle\)断言进行逻辑运算，断言语法如下:

```text
<MSPID>   //主体属于某个MSPID
<MSPID.ROLE{ADMIN|MEMBER|CLIENT|PEER}> //主体满足MSPID下的ROLE: 
    - ADMIN: 主体是MSPID下的admin证书
    - MEMBER: 主体是MSPID下签发的任何证书
    - PEER/CLIENT: 主体属于MSPID，且OU是PEER/CLIENT
<MSPID.OU> //主体身份属于某个MSPID下的OU, 暂未实现
```

```bash
示例: 
OR('SampleOrg.member', 'SampleOrg2.peer')
2 of [org1.Member, org1.Admin]
```

* ImplicitMeta策略

```c
<ANY|ALL|MAJORITY> <SubPolicyName>
示例:  MAJORITY Admins
```

### Signature策略

Signature策略定义如下:

{% code-tabs %}
{% code-tabs-item title="protos/common/policies.pb.go" %}
```go
type SignaturePolicyEnvelope struct {
	Version              int32               `protobuf:"varint,1,opt,name=version,proto3" json:"version,omitempty"`
	Rule                 *SignaturePolicy    `protobuf:"bytes,2,opt,name=rule,proto3" json:"rule,omitempty"`
	Identities           []*msp.MSPPrincipal `protobuf:"bytes,3,rep,name=identities,proto3" json:"identities,omitempty"`
	XXX_NoUnkeyedLiteral struct{}            `json:"-"`
	XXX_unrecognized     []byte              `json:"-"`
	XXX_sizecache        int32               `json:"-"`
}

type SignaturePolicy struct {
	// Types that are valid to be assigned to Type:
	//	*SignaturePolicy_SignedBy
	//	*SignaturePolicy_NOutOf_
	Type                 isSignaturePolicy_Type `protobuf_oneof:"Type"`
	XXX_NoUnkeyedLiteral struct{}               `json:"-"`
	XXX_unrecognized     []byte                 `json:"-"`
	XXX_sizecache        int32                  `json:"-"`
}
```
{% endcode-tabs-item %}

{% code-tabs-item title="SignaturePolicy\_NOutOf\_ struct" %}
```go
type SignaturePolicy_NOutOf_ struct {
	NOutOf *SignaturePolicy_NOutOf `protobuf:"bytes,2,opt,name=n_out_of,json=nOutOf,proto3,oneof"`
}
type SignaturePolicy_NOutOf struct {
	N                    int32              `protobuf:"varint,1,opt,name=n,proto3" json:"n,omitempty"`
	Rules                []*SignaturePolicy `protobuf:"bytes,2,rep,name=rules,proto3" json:"rules,omitempty"`
	XXX_NoUnkeyedLiteral struct{}           `json:"-"`
	XXX_unrecognized     []byte             `json:"-"`
	XXX_sizecache        int32              `json:"-"`
}
```
{% endcode-tabs-item %}

{% code-tabs-item title="SignaturePolicy\_SignedBy " %}
```go
type SignaturePolicy_SignedBy struct {
	SignedBy int32 `protobuf:"varint,1,opt,name=signed_by,json=signedBy,proto3,oneof"`
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}



