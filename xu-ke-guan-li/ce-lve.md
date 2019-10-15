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
示例: OR('SampleOrg.member', 'SampleOrg2.peer')
```

* ImplicitMeta策略

```c
<ANY|ALL|MAJORITY> <SubPolicyName>
示例:  MAJORITY Admins
```

### Signature策略



