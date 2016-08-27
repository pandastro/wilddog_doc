title:  错误码
---


## 错误码说明


错误码	|	解释	|  具体描述
---- | -------- | ---

WAuthenticationErrorDeniedByUser | 用户被阻止 | 用户未被授权。这个错误会在用户取消 OAuth 验证请求时出现

WAuthenticationErrorEmailTaken |	邮箱已存在 | 邮箱地址已存在(已注册)

WAuthenticationErrorInvalidConfiguration | 无效设置 | 提供商验证配置错误，请求未能完成。请确认提供商的客户端 ID 以及Secret是正确的

WAuthenticationErrorInvalidCredentials	| 证书无效 | 验证证书不可用。可能错误或者过期

WAuthenticationErrorInvalidEmail | 无效邮箱 | 该邮箱无效(不符合规范)

WAuthenticationErrorInvalidPassword	 | 密码错误 | 用户密码错误

WAuthenticationErrorNetworkError | 网络错误 | 尝试连接验证服务器时发生错误

WAuthenticationErrorPreempted | 网络错误 | 当前请求被一个新的认证请求中断
	
WAuthenticationErrorUnknown | 未知错误 | 发生了未知错误。请查阅详细错误信息	
WAuthenticationErrorUserDoesNotExist | 该用户不存在 | 该用户不存在

WAuthenticationErrorInvalidProvider | 不支持的验证提供商 | 验证提供商不存在，请查看 Wilddog Auth 文档中支持的提供商列表	
WAuthenticationErrorProviderError | 服务提供商错误 | 第三方验证提供商验证错误，请查看详细错误信息	

WAuthenticationErrorInvalidArguments |	冲突 | 资格证书错误或不全。请查看详细错误信息，并且从Wilddog 文档查找该服务商需要的验证参数

WAuthenticationErrorInvalidToken | 无效 Token	 | Wilddog ID Token 无效。可能是因为 token 错误、残缺、过期，或用于生成 Token 的 Secret 已经失效。


	

