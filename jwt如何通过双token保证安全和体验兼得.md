---
tags:
  - ai总结
  - 程序员
  - 八股文
  - 场景题
---
# 视频文案综合分析报告（信息提炼型）

[jwt如何通过双token保证安全和体验兼得](https://www.bilibili.com/video/BV1qb6EB2EGi?p=2)

## 讨论角度

视频从“安全 vs 体验”的矛盾出发，讨论 JWT 登录体系中 Token 过期时间如何设计：过期太长提升体验但增加被盗用风险；过期太短更安全但导致频繁登录、体验差。给出的主解法是“双 Token（access + refresh）续期机制”。

## 核心论点与分论点

- 核心论点：用短效 access token + 长效 refresh token 的双 Token 机制，可以在保证安全的同时，做到更好的用户无感续期体验。
- 分论点 1：单 token 模式下，token 短效会导致用户频繁 401 并被迫跳转登录页，体验差。
- 分论点 2：引入 refresh token 后，access token 过期触发续期流程，由 refresh token 换取新的 access token，用户无需显式重新登录。
- 分论点 3：refresh token 需要额外的安全设计：放在 HttpOnly Cookie 降低被脚本窃取的风险，并配合后端可控的失效/风控手段（例如服务端存储、异地登录检测、主动注销）。

## 关键数据 / 案例 / 结论

- 典型有效期建议（举例）：
  - access token：短效，约 30 分钟～1 小时
  - refresh token：长效，约 7 天～半个月（按业务风险调整）
- 典型交互与状态码：
  - access token 过期后接口返回 401（未授权/登录超时）
  - 前端携带 refresh token 请求后端刷新接口，后端验证通过后下发新的 access token
- 存储与防护要点：
  - refresh token 放在 HttpOnly Cookie 中（降低 XSS 窃取风险）
  - refresh token 在后端也保存一份（Redis/数据库），便于主动吊销
  - 发现异常（例如不同 IP 异地登录）可在后端销毁 refresh token，阻断“偷来的 refresh token 换新 access token”
- 结论：
  - 双 Token 机制通过“短效 access 控安全、长效 refresh 保体验”，是 JWT 续期的主流实现方案之一。
  - 除双 Token 外，视频也提到还有滑动窗口、前端无感刷新、强制重新登录等续期思路。

## 思维导图（markmap）

```markmap
# JWT 刷新：双 Token 机制
## 痛点：安全 vs 体验
### Token 过期太长
#### 体验好
#### 风险：access token 被截获后可长期滥用
### Token 过期太短
#### 安全性好
#### 体验差：频繁登录/跳转
## 方案：Access Token + Refresh Token
### Access Token（短效）
#### 携带位置：请求头
#### 用途：每次请求鉴权
#### 过期：30min~1h（示例）
### Refresh Token（长效）
#### 用途：给 Access Token 续期
#### 过期：7d~15d（示例）
#### 存放：HttpOnly Cookie
#### 服务端存储：Redis/DB（便于吊销）
## 典型流程
### 1) 登录成功
#### 下发 access + refresh
### 2) 正常请求
#### 请求头带 access，后端过滤器/拦截器校验
### 3) access 过期
#### 接口返回 401
### 4) 续期
#### 前端带 refresh 调刷新接口
#### 后端验证 refresh，签发新 access
### 5) 无感继续
#### 下次请求用新 access
## 安全补强
### HttpOnly 降低脚本窃取
### 有效期限制降低窗口
### 风控：异地登录/IP 异常
### 主动吊销 refresh（服务端拒绝换 token）
```

