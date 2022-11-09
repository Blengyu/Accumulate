## JWT鉴权

jwt鉴权取头部信息 x-token 登录时回返回token信息 这里前端需要把token存储到cookie或者本地localStorage中 不过需要跟后端协商过期时间 可以约定刷新令牌或者重新登录

```go
token := c.Request.Header.Get("x-token")
if token == "" {
   response.FailWithTokenDetailed(gin.H{"reload": true}, "未登录或非法访问", c)
   c.Abort()
   return
}
if jwtService.IsBlacklist(token) {
   response.FailWithTokenDetailed(gin.H{"reload": true}, "您的帐户异地登陆或令牌失效", c)
   c.Abort()
   return
}
```

## Casbin拦截器

判断用户当前角色是否拥有操作该接口的权限

```go
waitUse, _ := utils.GetClaims(c)
// 获取请求的PATH
obj := c.Request.URL.Path
// 获取请求方法
act := c.Request.Method
// 获取用户的角色
sub := waitUse.AuthorityId
e := casbinService.Casbin()
// 判断策略中是否存在
success, _ := e.Enforce(sub, obj, act)
if global.GVA_CONFIG.System.Env == "develop" || success {
   c.Next()
} else {
   response.FailWithDetailed(gin.H{}, "权限不足", c)
   c.Abort()
   return
}
```

给每个用户分配角色该有的权限

```go
func (casbinService *CasbinService) UpdateCasbin(authorityId string, casbinInfos []request.CasbinInfo) error {
   casbinService.ClearCasbin(0, authorityId)
   rules := [][]string{}
   for _, v := range casbinInfos {
      cm := system.CasbinModel{
         Ptype:       "p",
         AuthorityId: authorityId,
         Path:        v.Path,
         Method:      v.Method,
      }
      rules = append(rules, []string{cm.AuthorityId, cm.Path, cm.Method})
   }
   e := casbinService.Casbin()
   success, _ := e.AddPolicies(rules)
   if !success {
      return errors.New("存在相同api,添加失败,请联系管理员")
   }
   return nil
}
```