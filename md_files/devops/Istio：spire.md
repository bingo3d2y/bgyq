## Istio：spire

https://mp.weixin.qq.com/s/043yz1etTkJ1l4Eo6DG7WA



### 零信任

零信任的本质是以身份为中心的动态访问控制。动态证书轮换、动态证书下发、动态权限控制。

### SPIFFE

SPIFFE 解决的是标识工作负载的问题。

#### 相关术语

- **SPIFFE**（Secure Production Identity Framework For Everyone）是一套身份认证标准。
-  **SPIRE**（SPIFFE Runtime Environment） 是 SPIFFE 标准的一套生产就绪实现。
- **SVID**（SPIFFE Verifiable Identity Document）是工作负载向资源或调用者证明其身份的文件。SVID 包含一个 SPIFFE ID，代表了服务的身份。它将 SPIFFE ID 编码在一个可加密验证的文件中，目前支持两种格式：X.509 证书或 JWT 令牌。
- **SPIFFE ID** 是一个统一资源标识符（URI），其格式如下：`spiffe://trust_domain/workload_identifier`。

### SPIRE

身份是零信任网络的基础，SPIFFE 统一了异构环境下的身份标准。在 Istio 中不论我们是否使用 SPIRE，身份验证对于工作负载来说是不会有任何感知的。通过 SPIRE 来为工作负载提供身份验证，可以有效地管理工作负载的身份，为实现零信任网络打好基础。