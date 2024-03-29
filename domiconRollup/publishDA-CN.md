# batcher发布数据
## batcher发布数据的流程如下  
### 1. 读取部署在L1上的智能合约，选择最优的广播节点。  
    1.1 读取合约上所有广播节点信息  
    ```go
    // get all node address
	bcNodeAddrs := new([]interface{})
	err := l1DomiconNodesContract.Call(&bind.CallOpts{}, bcNodeAddrs, "BROADCAST_NODES")
    ```
    1.2 验证广播节点的网络状况，选择网络状态最好的节点
    ```go
    // select best node
    ch := make(chan struct {
		duration time.Duration
		url      string
	}, len(nodesAddrRpc))
	for rpcUrl, _ := range nodesAddrRpc {
		go testRPCLatency(rpcUrl, ch)
	}
	var fastestUrl string
	for i := 0; i < len(nodesAddrRpc); i++ {
		item := <-ch
		if item.duration == time.Duration(0) {
			continue
		}
		if item.duration < minDuration {
			minDuration = item.duration
			fastestUrl = item.url
		}
	}
    ```
### 2. 组装batcher需要发送给domicon的data  

    2.1 获取当前用户的index值  
    ```go
    l1DomiconCommitmentContract.Call(&bind.CallOpts{}, results, "indices", userAddr)
    ```
    2.2 调用kzg-sdk的多项式算法，为batcher的tx.data生成CM(data commitment)
    ```go
    digest, err := kzgsdk.GenerateDataCommit(candidate.TxData)
	if err != nil {
		return nil, fmt.Errorf("failed to generate data commit: %w", err)
	}
	rawCD.CM = digest.Bytes()
    ```
    2.3 调用kzg-sdk的签名算法，对（tx.sender, tx.to, gasPrice, index， tx.data的长度， CM数据）进行签名  
    ```go
    singer := kzgsdk.NewEIP155FdSigner(big.NewInt(chainID))
	sigHash, sigData, err := kzgsdk.SignFd(m.cfg.From, rawCD.To, gasPrice, *m.index, uint64(length), rawCD.CM[:], singer, m.cfg.PrivateKey)
	if err != nil {
		return nil, fmt.Errorf("failed to sign commitment data : %w", err)
	}
	log.Info("craftCD", "sigHash", sigHash)
    ```
### 3. 数据发布
    通过rollupClient的sendDA接口，发布数据
    ```go
    cCtx, cancel := context.WithTimeout(ctx, m.cfg.NetworkTimeout)
	hash, err := rollupClient.SendDA(cCtx, rawCD.Index, rawCD.Len, rawCD.To, rawCD.From, hexutil.Bytes(rawCD.CM[:]),
			hexutil.Bytes(rawCD.Sig), hexutil.Bytes(rawCD.Data))
    ```