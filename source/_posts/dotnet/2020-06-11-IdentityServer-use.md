---
title : "IdentityServer使用指南"
---

DotHass.Lobby.Domain\IdentityServer\IdentityServerDataSeedContributor.cs 中 CreateClientsAsync()

会在dataseed的时候生成默认数据



When I try to send a HTTPS POST request from a desktop (Servers are in production environment) the following message is displayed inside the console :

```
Error: unable to verify the first certificate
```

After: `Postman -> Preferences -> General -> SSL certificate validation -> OFF` **it works**

https://localhost:5000/.well-known/openid-configuration



1. http://localhost:5000/connect/token

   ![image-20200613165200371](../../assets/images/2020-06-11-IdentityServer-use/image-20200613165200371.png)

2.  http://localhost:5000/connect/userinfo 将type设置成bearer token,token填入上面获得的access_token

![image-20200613165246959](../../assets/images/2020-06-11-IdentityServer-use/image-20200613165246959.png)

3.注意发布release后.配置表中的  ..如果配置错误将会认证失败

appsettings.json

```
{
  "App": {
    "SelfUrl": "http://localhost:5000"
  },
  "ConnectionStrings": {
    "Default": "Server=localhost;User Id=root;Password=123456;Database=dothass.blog"
  },
  "AuthServer": {
    "Authority": "http://localhost:5000"
  },
  "IdentityServer": {
    "Clients": {
      "Blog_App": {
        "ClientId": "Blog_App"
      }
    }
  }
}
```

appsettings.Development.json

```
{
  "App": {
    "SelfUrl": "https://localhost:44377"
  },
  "AuthServer": {
    "Authority": "https://localhost:44377"
  }
}
```

还要注意请求的域名是否一样,127.0.0.1或者localhost...可能返回结果即使一样.但是不能授权.

使用http://jwt.calebb.net/解析看下access_token

```
{
 alg: "RS256",
 kid: "1oauLjO2TtmvAH-4A7CCLg",
 typ: "at+jwt"
}.
{
 nbf: 1592054993,
 exp: 1623590993,
 iss: "http://127.0.0.1:5000",
 aud: "Blog",
 client_id: "Blog_App",
 sub: "fa9626f7-0f6f-6158-2afd-39f5a7f6d03f",
 auth_time: 1592054993,
 idp: "local",
 role: "admin",
 name: "admin",
 email: "admin@abp.io",
 email_verified: false,
 scope: [
  "address",
  "email",
  "openid",
  "phone",
  "profile",
  "role",
  "Blog",
  "offline_access"
 ],
 amr: [
  "pwd"
 ]
}.
```



```
	{
 alg: "RS256",
 kid: "1oauLjO2TtmvAH-4A7CCLg",
 typ: "at+jwt"
}.
{
 nbf: 1592055396,
 exp: 1623591396,
 iss: "http://localhost:5000",
 aud: "Blog",
 client_id: "Blog_App",
 sub: "fa9626f7-0f6f-6158-2afd-39f5a7f6d03f",
 auth_time: 1592055396,
 idp: "local",
 role: "admin",
 name: "admin",
 email: "admin@abp.io",
 email_verified: false,
 scope: [
  "address",
  "email",
  "openid",
  "phone",
  "profile",
  "role",
  "Blog",
  "offline_access"
 ],
 amr: [
  "pwd"
 ]
}.
```



