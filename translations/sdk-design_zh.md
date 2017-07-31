---
layout: page
title: "[翻译]Hyperledger Fabric SDK设计说明书 v1.0"
description: ""
---
{% include JB/setup %}

原文：<https://docs.google.com/document/d/1R5RtIBMW9fZpli37E5Li5_Q9ve3BnQ4q3gWmGZj6Sv4/edit>

翻译：郭立冬 <690748277@qq.com>

---

* 目录
{:toc}

---




































![](../images/sdk-image01.png)

### 4.3交易支持




![](../images/sdk-image02.jpg)

上图是对交易流程非常上层的描述。SDK中有详细的执行网络和环境配置，包括获取签名密钥来安全的管理成员，处理交易和事件流，（依赖应用的）多种共识通道。





## 6详细说明



| ---- | --- | --- |
| Client | 0 | 主要的入口模块。它必须允许用户创建需要的任何对象来执行所有支持的操作，例如直接连接网络，chaincode部署，交易执行，多种查询。另外，基于编码规范和普遍的社区练习，每一种语言的实现也能决定是否添加方便的方法，如sendTransaction(chain, tx)|

| --- | --- | --- |

为了帮助说明以上各模块的关系，我们给出了以下的UML图：

![](../images/sdk-image03.png)




	根据参数给出的名字创建一个链的实例。这实际上代表“通道”（正如上面解释的），这个调用返回一个空对象。初始化这个通道，这个返回的对象上必须配置一个参与方的背书者列表和排序者节点。
	
	
	
	
	

	Params
	* name (str): The name of the chain （链的名称）
	* (Chain instance or None): the chain instance for the name. （以入参命名的链的实例）
	* The state store has not been set（还没有设置存储状态）
	
	* name (str): The name of the chain （链名）
	Returns
	* None
	为client提供的获取状态存储的对象的便利方法
	* None
	* None
	* (User instance): The user object corresponding to the name, or null if the user does not exist or if the state store has not been set（返回匹配名称的用户对象，如果用户不存在或者状态存储未设置返回空）

	* peer (Peer): an instance of the Peer class that has been initialized with URL, TLC certificate, and enrollment certificate（入参：被URL，TLC证书，和背书证书初始化之后的Peer类的对象。）
	* peer (Peer): an instance of the Peer class（节点实例）
	* orderer (Orderer): an instance of the Orderer class （Orderer实例）
	* (Orderer list): The orderer list on the chain
	* None
	Returns （链的初始化动作是否执行成功）
	获取链的状态来查看底层通道是否已经被终止，创建一个只读链，可以查询信息（交易和状态）但是不可以提交新的交易。
	* Object containing the block
	* transactionID
	* (Proposal): The created Proposal instance or None.
	Returns
	* transaction_proposal (Transaction_Proposal): The transaction proposal data（交易提案的数据）
	* (Transaction_Proposal_Response response): The response to send proposal request.（发送提案请求后的响应）
	* (Transaction instance): The created transaction object instance.（创建的交易对象实例）
	

	




	获取成员名称。从对象实例请求参数。
	
	* The name of the user
		获取用户的角色。它可能是“客户端”“审计员”这类值的数组。成员服务多定义了两个的角色用来保存节点成员关系，如“节点”和“验证者”，这两个没有暴露给应用程序。
	* name (string[): The user name / id.
	设置用户的背书证书。




	* None
	备注：这个请求对节点上事件的生成者有强化作用。
	* eventName (string): required （必需的）
	
	 
	获取节点名称。需要对象实例的参数。
	* The name of the Peer
	* Name (string): The unique name / id of this Peer.
	* Roles (string[]): The list of roles for the user（用户角色列表）
	* Certificate in PEM format signed by the trusted CA
	* Certificate: Certificate in PEM format signed by the trusted CA


	Retrieves a value given a key
	* key (str): The name of the key
	* Result (Object): The value
	Sets the value
	*  value
	* Acknowledgement of successful storage of the value
	





	
	* opts (Object): an object that encapsulates two properties, “algorithm” and“ephemeral”.

	Derives a key from k using opts.
	* (Key) derived key
	
	* ski (byte[])
	* (Key) An instance of the Key class corresponding to the ski
	Hashes messages msg using options opts .
	* msg (byte[])
	Encrypt plain text.
	* (byte[]) Cipher text
	
	Decrypt cipher text.
	Sign the data.
	Params
	Params
	* key (Key) public verification key
	* (bool): verification successful or not








