[toc]

## 1. 基本使用

### 1.1 创建一个IAM实例

```go
import "github.com/TencentBlueKing/iam-go-sdk"
i := iam.NewIAM("bk_paas", "bk_paas", "{app_secret}", "http://{iam_backend_addr}", "http://{paas_domain}")
```

如果是使用 APIGateway 的方式(如果有, 推荐使用这种方式)

```go
import "github.com/TencentBlueKing/iam-go-sdk"
// if your TencentBlueking has a APIGateway, use NewAPIGatewayIAM, the url suffix is /stage/(for testing) and /prod/(for production)
i := iam.NewAPIGatewayIAM("bk_paas", "bk_paas", "{app_secret}", "http://bk-iam.{APIGATEWAY_DOMAIN}/stage/")
```

未来 APIGateway 高性能网关会作为一个基础的蓝鲸服务, 权限中心将会把后台及 SaaS 的所有开放 API 接入到网关中`bk-iam`

此时, 对于接入方, 不管是鉴权/申请权限还是其他接口, 都可以通过同一个网关访问到.

理解成本更低, 且相关的调用日志/文档/流控/监控等都可以在 APIGateway 统一管控.

网关地址类似: `http://bk-iam.{APIGATEWAY_DOMAIN}/{env}`, 其中 `env`值 `prod(生产)/stage(预发布)`

### 1.2 设置logger

开发时, 可以将log level设置为debug, 这样能在日志中查看到请求/响应/求值过程的详细数据;

注意: 生产环境请将日志级别设为 ERROR

如果生产环境开启debug带来的问题:
- 日志量过大
- 影响请求速度(性能大幅降低)
- 敏感信息泄漏(权限相关)

```go
import (
	"fmt"
	"os"
	"time"

	"github.com/sirupsen/logrus"

	"github.com/TencentBlueKing/iam-go-sdk"
	"github.com/TencentBlueKing/iam-go-sdk/logger"
)

func main() {
	// create a logger
	log := &logrus.Logger{
		Out:          os.Stderr,
		Formatter:    new(logrus.TextFormatter),
		Hooks:        make(logrus.LevelHooks),
		Level:        logrus.DebugLevel,
		ExitFunc:     os.Exit,
		ReportCaller: false,
	}
	// do set logger
	logger.SetLogger(log)
}
```

### 1.3 开启api debug

某些情况下, 例如在排查为什么有权限/无权限问题时, 需要开启api debug, 此时, 通过调用接口参数加入 debug 来获取更多信息

开启api debug:

```
设置环境变量: 
IAM_API_DEBUG=true 或 BKAPP_IAM_API_DEBUG=true
```

开启force强制服务端不走缓存:

```
设置环境变量: 
IAM_API_FORCE=true 或 BKAPP_IAM_API_FORCE=true
```

注意, 开启后性能非常低, 不应该在生产环境中使用


## 2. 鉴权

### 2.1 IsAllowed

> 查询是否有某个操作权限(没有资源实例), 例如访问开发者中心

```go
req := iam.NewRequest(
    "bk_paas",
    iam.NewSubject("user", "admin"),
    iam.NewAction("access_developer_center"),
    []iam.ResourceNode{},
)

allowed, err := i.IsAllowed(req)
fmt.Println("isAllowed:", allowed, err)
```

> 查询是否有某个资源的某个操作权限(有资源实例), 例如管理某个应用

```go
req := iam.NewRequest(
    "bk_paas",
    iam.NewSubject("user", "admin"),
    iam.NewAction("develop_app"),
    []iam.ResourceNode{
        iam.NewResourceNode("bk_paas", "app", "1", map[string]interface{}{}),
    },
)

allowed, err := i.IsAllowed(req)
fmt.Println("isAllowed:", allowed, err)
```

### 2.3 BatchIsAllowed

> 对一批资源同时进行鉴权

可以调用`batch_is_allowed` (注意这个封装不支持跨系统资源依赖, 只支持接入系统自己的本地资源)

```go
req := iam.NewRequest(
    "bk_paas",
    iam.NewSubject("user", "admin"),
    iam.NewAction("develop_app"),
    // NOTE: here is empty resource
    []iam.ResourceNode{},
)

resourcesList := []iam.Resources{
    []iam.ResourceNode{
        {
            System: "bk_paas",
            Type:   "app",
            ID:     "test1",
        },
    },
}

result, err := i.BatchIsAllowed(req, resourcesList)
fmt.Println("BatchIsAllowed:", result, err)
```

### 2.4 ResourceMultiActionsAllowed

> 对一个资源的多个操作同时进行鉴权

可以调用`resource_multi_actions_allowed`进行批量操作权限的鉴权

```go
multiReq := iam.NewMultiActionRequest(
    "bk_sops",
    iam.NewSubject("user", "admin"),
    []iam.Action{
        iam.NewAction("task_delete"),
        iam.NewAction("task_edit"),
        iam.NewAction("task_view"),
    },
    []iam.ResourceNode{
        iam.NewResourceNode("bk_sops", "task", "1", map[string]interface{}{"iam_resource_owner": "admin"}),
    },
)
result, err := i.ResourceMultiActionsAllowed(multiReq)
fmt.Println("ResourceMultiActionsAllowed: ", result, err)
```

### 2.5 BatchResourceMultiActionsAllowed

> 对于批量资源的多个操作同时进行鉴权, 例如进入资源列表也，可能需要在前端展示当前用户关于列表中的资源的一批操作的权限信息

可以调用`batch_resource_multi_actions_allowed`

```go
multiReq := iam.NewMultiActionRequest(
    "bk_sops",
    iam.NewSubject("user", "admin"),
    []iam.Action{
        iam.NewAction("task_delete"),
        iam.NewAction("task_edit"),
        iam.NewAction("task_view"),
    },
    // NOTE: here is empty resource
    []iam.ResourceNode{},
)
resourcesList := []iam.Resources{
    []iam.ResourceNode{
        iam.NewResourceNode("bk_sops", "task", "1", map[string]interface{}{"iam_resource_owner": "admin"}),
    },
    []iam.ResourceNode{
        iam.NewResourceNode("bk_sops", "task", "2", map[string]interface{}{"iam_resource_owner": "admin2"}),
    },
    []iam.ResourceNode{
        iam.NewResourceNode("bk_sops", "task", "3", map[string]interface{}{"iam_resource_owner": "admin3"}),
    },
}
results, err := i.BatchResourceMultiActionsAllowed(multiReq, resourcesList)
fmt.Println("BatchResourceMultiActionsAllowed: ", results, err)
```

### 2.2 IsAllowedWithCache

> 对于非敏感权限

可以调用`is_allowed_with_cache(request)`, 缓存10s. (注意: 不要用于新建关联权限的资源is_allowed判定, 否则可能新建一个资源新建关联生效之后跳转依旧没权限; 更多用于管理权限/未依赖资源的权限权限判断)

```go
// check 3 times but only call iam backend once
allowed, err = i.IsAllowedWithCache(req, 10*time.Second)
allowed, err = i.IsAllowedWithCache(req, 10*time.Second)
i2 := iam.NewIAM("bk_paas", "bk_paas", "{app_secret}", "http://{iam_backend_addr}", "http://{paas_domain}")
allowed, err = i2.IsAllowedWithCache(req, 10*time.Second)
fmt.Println("isAllowedWithCache:", allowed, err)
```

## 3. 非鉴权

### 3.1 获取无权限申请跳转url

> 没有权限时, 在前端展示要申请的权限列表, 需要访问IAM接口, 拿到申请权限url; 用户点击跳转到IAM SaaS对应页面申请权限

```go
actions := []iam.ApplicationAction{
    iam.NewApplicationAction("access_developer_center", []iam.ApplicationRelatedResourceType{}),
}
application := iam.NewApplication("bk_paas", actions)

url, err := i.GetApplyURL(application, "", "admin")
fmt.Println("GetApplyURL:", url, err)
```

### 3.2 生成无权限描述协议数据

可以生成生成 [无权限描述协议数据](https://bk.tencent.com/docs/document/6.0/160/8463)

```go
d := ApplicationActionListForApply{}
data, err := i.GenPermissionApplyData(d)
```

### 3.3 回调接口basic auth校验

```go
// get and parse the basic auth from http header
err := c.IsBasicAuthAllowed("bk_iam", "theToken")
fmt.Println("IsBasicAuthAllowed:", err)
```

### 3.4 查询系统的Token

```go
token, err := i.GetToken()
fmt.Println("GetToken:", token, err)
```

## 4. SDK 增强

### 注册metrics

enable prometheus metrics

```go
import "github.com/TencentBlueKing/iam-go-sdk/metric"
metric.RegisterMetrics()
```

### 实现回调dispatcher/provider接口

Implement resource callback api via dispatcher/provider interface

see [examples/dispatcher_provider/main.go](../examples/dispatcher_provider/main.go)

```go
func main() {
    d := resource.NewDispatcher()
    dummyProvider := DummyProvider{}

    // type=dummy will use the dummyProvider
    d.RegisterProvider("dummy", dummyProvider)

    handler := resource.NewDispatchHandler(d)

    // we just register only one api to handle all resource types - callback api
    http.HandleFunc("/api/v1/resource", handler)
}
```
