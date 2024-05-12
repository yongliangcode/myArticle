```
title: Mesh9# 使用wasm上报envoy埋点信息
categories: Mesh
tags: Mesh
date: 2021-12-19 11:55:01
```



# 编写上报代码

**Step1 Java 服务接受回调请求** 

```java
@RestController
public class WasmCallBack {

    @PostMapping(value = "/wasmCallback")
    public String callback(@RequestParam Map<String,String> params,
                           @RequestHeader("melonMsg") String melonMsg,
                           @RequestBody String body){
        StringBuilder paramsBuilder = new StringBuilder();
        params.forEach((k,v)->{
            paramsBuilder.append(k).append("=").append(params.get(k)).append(",");
        });
        System.out.println("收到请求参数：" + paramsBuilder.toString() + " " + getCurrentDate());
        System.out.println("收到header：" + melonMsg );
        System.out.println("收到body：" + body );
        return "receive wasm callback request status ok";
    }

    public String getCurrentDate(){
        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        return sdf.format(new Date());
    }

}
```

**备注：服务为mesha模拟接受envoy的上报指标请求。** 



**Step2 wasm filter上报统计请求** 

```go
package main

import (
	"github.com/tetratelabs/proxy-wasm-go-sdk/proxywasm"
	"github.com/tetratelabs/proxy-wasm-go-sdk/proxywasm/types"
)

const tickMilliseconds uint32 = 5000
func main() {
	proxywasm.SetVMContext(&vmContext{})
}

type vmContext struct {
	// Embed the default VM context here,
	// so that we don't need to reimplement all the methods.
	types.DefaultVMContext
}

// Override types.DefaultVMContext.
func (*vmContext) NewPluginContext(contextID uint32) types.PluginContext {
	return &pluginContext{contextID: contextID}
}

type pluginContext struct {
	// Embed the default plugin context here,
	// so that we don't need to reimplement all the methods.
	types.DefaultPluginContext
	contextID uint32
	// callBack  func(numHeaders, bodySize, numTrailers int)
	cnt       int
}

func (*pluginContext) NewHttpContext(ctxId uint32) types.HttpContext {
	return &types.DefaultHttpContext{}
}

// Override types.DefaultPluginContext.
func (ctx *pluginContext) OnPluginStart(pluginConfigurationSize int) types.OnPluginStartStatus {
	if err := proxywasm.SetTickPeriodMilliSeconds(tickMilliseconds); err != nil {
		proxywasm.LogCriticalf("failed to set tick period: %v", err)
		return types.OnPluginStartStatusFailed
	}
	proxywasm.LogInfof("set tick period milliseconds: %d", tickMilliseconds)
	return types.OnPluginStartStatusOK
}

// Override types.DefaultPluginContext.
func (ctx *pluginContext) OnTick() {
	hs := [][2]string{
		{":method", "POST"},
		{":authority", "mesha.istio12.svc.cluster.local"},
		{"melonMsg", "my header from wasm envoy reporter"},
		{":path", "/wasmCallback"}, {"accept", "*/*"},
	}
	body := []byte("i'm wasm from wasm filter !!!")
	proxywasm.LogInfof("OnTick method is called before dispatchHttpCall ")
	if _, err := proxywasm.DispatchHttpCall("outbound|7171||mesha.istio12.svc.cluster.local", hs, body, nil, 5000, callback); err != nil {
		proxywasm.LogCriticalf("dispatch httpcall failed: %v", err)
	}
	proxywasm.LogInfof("OnTick method is called after dispatchHttpCall ")
}

func callback(numHeaders, bodySize, numTrailers int) {
	proxywasm.LogInfof("wasm callback...")
}
```

**备注：通过proxywasm.DispatchHttpCall每隔5秒上报埋点请求，示例中通过增加header、body的方式进行模拟。** 需要注意的是DispatchHttpCall第一个参数为cluster即outbound|7171||mesha.istio12.svc.cluster.local。可以通过以下命令查看

```
istioctl pc cluster mesha-b559fc4f4-qngq7 -n istio12 | grep mesha
mesha.istio12.svc.cluster.local                            7171      -          outbound      EDS
```

**Step3 使用域名访问** 

在实际中使用域名比较方便，需要通过serviceEntry，下面是使用https格式如下：

```
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: mesha
spec:
  hosts:
    - fat-mesh-metrics-inner.xxxx.cn
  location: MESH_EXTERNAL
  ports:
    - name: https
      number: 443
      protocol: TLS
  resolution: DNS

---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: tls-mesh-metrics
spec:
  host: "*-mesh-metrics-inner.xxxx.cn"
  trafficPolicy:
    tls:
      mode: SIMPLE
```







# # wasm构建方式

**Step1 构建wasm filter镜像** 

```
wasme build tinygo /Users/yongliang/GoLandProjects/melon-filter -t x.x.x.x/zzzz/melon-add-header:v0.6
Building with tinygo...go: downloading github.com/tetratelabs/proxy-wasm-go-sdk v0.14.0
INFO[0010] adding image to cache...                      filter file=/tmp/wasme730345840/filter.wasm tag="harbor.hellobike.cn/base/melon-add-header:v0.6"
INFO[0010] tagged image                                  digest="sha256:f793a19da4157cd1768ffb5c6b93acbbc10319c6b20081cb24d1000072d1c393" image="harbor.hellobike.cn/base/melon-add-header:v0.6"
```

**Step2 查看本地镜像列表** 

melon-add-header:v0.6为刚构建的

```
wasme list
NAME                                      TAG  SIZE     SHA      UPDATED
x.x.x.x/zzzz/melon-add-header v0.1 247.6 kB 750d6388 15 Nov 21 20:23 CST
x.x.x.x/zzzz/melon-add-header v0.2 247.6 kB 11c22c8e 16 Nov 21 14:12 CST
x.x.x.x/zzzz/melon-add-header v0.3 248.2 kB 34f739c2 24 Nov 21 11:44 CST
x.x.x.x/zzzz/melon-add-header v0.6 248.7 kB f793a19d 01 Dec 21 14:33 CST
```

**Step3 镜像推送** 

```
wasme push x.x.x.x/zzzz/melon-add-header:v0.6
INFO[0000] Pushing image x.x.x.x/zzzz/melon-add-header:v0.6 
INFO[0003] Pushed x.x.x.x/zzzz/melon-add-header:v0.6 
INFO[0003] Digest: sha256:165dd8fc593ed055d0a52e192700b38b225d0db6f09474d36667049750478a5e 
```

**Step4 镜像拉取** 

```
wasme pull x.x.x.x/zzzz/melon-add-header:v0.6

INFO[0000] Pulling image x.x.x.x/zzzz/melon-add-header:v0.6
INFO[0001] Image: x.x.x.x/zzzz/melon-add-header:v0.6
INFO[0001] Digest: sha256:f793a19da4157cd1768ffb5c6b93acbbc10319c6b20081cb24d1000072d1c393
```

**Step5 本地镜像存储路径** 

```
${user.home}.wasme/store/a32bb86b56fd75088071c2c348382a44
4 -rw-r--r-- 1 root root     46 Dec  1 15:39 image_ref
4 -rw-r--r-- 1 root root    225 Dec  1 15:39 descriptor.json
4 -rw-r--r-- 1 root root    126 Dec  1 15:39 runtime-config.json
244 -rw-r--r-- 1 root root 248664 Dec  1 15:39 filter.wasm
```

**Step6 启动HTTP服务方便下载** 

在该目录启动http-server作为测试

```
/root/.wasme/store/a32bb86b56fd75088071c2c348382a44]
# http-server
Starting up http-server, serving ./

http-server version: 14.0.0

http-server settings:
CORS: disabled
Cache: 3600 seconds
Connection Timeout: 120 seconds
Directory Listings: visible
AutoIndex: visible
Serve GZIP Files: false
Serve Brotli Files: false
Default File Extension: none

Available on:
  http://127.0.0.1:8080
  http://x.x.x.x:8080
  http://x.x.x.x:8080
```

完整的url为：http://x.x.x.x:8080/filter.wasm



# tinygo构建方式



**Step1 安装tinygo工具包** 

```
brew tap tinygo-org/tools
Updating Homebrew...

==> Tapping tinygo-org/tools
Cloning into '/usr/local/Homebrew/Library/Taps/tinygo-org/homebrew-tools'...
remote: Enumerating objects: 114, done.
remote: Counting objects: 100% (24/24), done.
remote: Compressing objects: 100% (17/17), done.
remote: Total 114 (delta 7), reused 19 (delta 7), pack-reused 90
Receiving objects: 100% (114/114), 18.80 KiB | 283.00 KiB/s, done.
Resolving deltas: 100% (35/35), done.
Tapped 1 formula (13 files, 27.4KB).
```

**Step2 安装tinygo** 

```
brew install tinygo
// ...
==> Installing dependencies for tinygo-org/tools/tinygo: binaryen
==> Installing tinygo-org/tools/tinygo dependency: binaryen
==> Pouring binaryen--102.catalina.bottle.tar.gz
🍺  /usr/local/Cellar/binaryen/102: 1,456 files, 38.4MB
==> Installing tinygo-org/tools/tinygo
🍺  /usr/local/Cellar/tinygo/0.21.0: 4,719 files, 362.4MB, built in 20 seconds
==> `brew cleanup` has not been run in 30 days, running now...
Removing: /Users/yongliang/Library/Logs/Homebrew/envoy... (64B)
```

**Step3 tinygo构建** 

生成melon-plugin.wasm文件

```
tinygo build -o target/melon-plugin.wasm -scheduler=none -target=wasi -wasm-abi=generic -no-debug
```

**Step4 构建wasm filter镜像** 

```
wasme build precompiled target/melon-plugin.wasm --tag  x.x.x.x/zzzz/wasm-filter-reporter:0.1 -v
```

**Step5 查看本地镜像** 

```
wasme list
NAME                                          TAG   SIZE     SHA      UPDATED
x.x.x.x/zzzz/wasm-filter-reporter 0.1   76.3 kB  76395184 02 Dec 21 14:35 CST
```

**Step6 镜像推送到harbor仓库** 

```
wasme push x.x.x.x/zzzz/wasm-filter-reporter:0.1
INFO[0000] Pushing image x.x.x.x/zzzz/wasm-filter-reporter:0.1 
INFO[0004] Pushed x.x.x.x/zzzz/wasm-filter-reporter:0.1 
INFO[0004] Digest: sha256:b1f8ee9c676802a8aa63de66963c492c0e76ce0e2f3377d1ef8c1b21b76ea0cd 
```

**Step7 在调试服务器上镜像拉取** 

```
wasme pull x.x.x.x/zzzz/wasm-filter-reporter:0.1

INFO[0000] Pulling image x.x.x.x/zzzz/wasm-filter-reporter:0.1
INFO[0001] Image: x.x.x.x/zzzz/wasm-filter-reporter:0.1
INFO[0001] Digest: sha256:76395184289a2ac7295a55b94f16765cc63a78d951d1277f27ad924ec7ee0187
```

**Step8 本地镜像存储路径** 

```
${user.home}/.wasme/store/a942f445d394384de38fef37e61fef13
-rw-r--r-- 1 root root   224 Dec  2 14:43 descriptor.json
-rw-r--r-- 1 root root 76318 Dec  2 14:43 filter.wasm
-rw-r--r-- 1 root root    49 Dec  2 14:43 image_ref
-rw-r--r-- 1 root root   126 Dec  2 14:43 runtime-config.json
```

**Step9 启动HTTP服务方便下载** 

下载地址为：http://10.69.57.24:8080/filter.wasm

```
cd ${user.home}/.wasme/store/a942f445d394384de38fef37e61fef13
http-server
Starting up http-server, serving ./

http-server version: 14.0.0

http-server settings:
CORS: disabled
Cache: 3600 seconds
Connection Timeout: 120 seconds
Directory Listings: visible
AutoIndex: visible
Serve GZIP Files: false
Serve Brotli Files: false
Default File Extension: none

Available on:
  http://127.0.0.1:8080
  http://10.69.57.24:8080
  http://10.74.1.129:8080
Hit CTRL-C to stop the server
```

完整的url为：http://x.x.x.x:8080/filter.wasm



# 加载wasm filter



**Step1 编写envoyfilter让其加载filter.wasm** 

```
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: wasm-filter-reporter
spec:
  configPatches:
  - applyTo: HTTP_FILTER
    match:
      context: ANY
      #listener:
      #  portNumber: 7171
    patch:
      operation: INSERT_BEFORE
      value:
        name: wasm-filter-reporter
        config_discovery:
          config_source:
            ads: {}
            initial_fetch_timeout: 0s 
          type_urls: [ "type.googleapis.com/envoy.extensions.filters.http.wasm.v3.Wasm"]
---
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: wasm-filter-reporter-config
spec:
  configPatches:
  - applyTo: EXTENSION_CONFIG
    match:
      context: ANY
      #listener:
      #  portNumber: 7171
    patch:
      operation: ADD
      value:
        name: wasm-filter-reporter
        typed_config:
          "@type": type.googleapis.com/udpa.type.v1.TypedStruct
          type_url: type.googleapis.com/envoy.extensions.filters.http.wasm.v3.Wasm
          value:
            config:
              vm_config:
                vm_id: wasm-filter-reporter_vm
                runtime: envoy.wasm.runtime.v8
                code:
                  remote:
                    http_uri:
                      uri: http://x.x.x.x:8080/filter.wasm
                      timeout: 10s
              configuration:
                '@type': type.googleapis.com/google.protobuf.StringValue
                value: |
                  {}
```

**Step2 envoyfilter执行生效** 

```
# kubectl apply -f wasm-filter-reporter.yaml
envoyfilter.networking.istio.io/melon-wasm-envoy-filter created
envoyfilter.networking.istio.io/melon-wasm-filter-config created
```



# 上报结果验证

**Step1 验证上报服务输出日志** 

```
# kubectl logs -f mesha-b559fc4f4-qngq7 -n istio12
收到请求参数： 2021-12-03 09:37:28
收到header：my header from wasm envoy reporter
收到body：i'm wasm from wasm filter !!!

收到请求参数： 2021-12-03 09:37:33
收到header：my header from wasm envoy reporter
收到body：i'm wasm from wasm filter !!!
// ...
```

**备注：接受上报请求的服务mesha每5秒钟输出一次日志。**



**Step2 验证Envoy输出日志** 

```
# kubectl logs -f mesha-b559fc4f4-rnfv7 -c istio-proxy -n istio12
2021-12-03T08:04:22.759629Z	info	envoy wasm	wasm log wasm-filter-reporter_vm: OnTick method is called before dispatchHttpCall
2021-12-03T08:04:22.759895Z	info	envoy wasm	wasm log wasm-filter-reporter_vm: OnTick method is called after dispatchHttpCall
2021-12-03T08:04:22.761450Z	info	envoy wasm	wasm log wasm-filter-reporter_vm: wasm callback...
//...
```

**备注：通过查看Envoy日志中找到了wasm filter中打印的日志信息。** 
