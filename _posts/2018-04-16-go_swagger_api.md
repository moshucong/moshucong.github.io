---
layout: post
title:  "用swagger为Go后端实现在线API文档"
date:   2018-04-16 17:12:13 +0800
categories: Golang
---

引入swagger主要是为了：1）作为文档。供客户端开发人员用来查阅后端API定义; 2）作为测试工具。供测试人员使用来发送HTTP请求，测试API。   

## 1. swagger-ui工作原理

swagger-ui的使用非常简单。  

可以clone其源码( https://github.com/swagger-api/swagger-ui )，其中``dist``目录就是完整可部署的swagger前端。
直接用浏览器打开``dist``目录中的``index.html``，页面上显示的是示例应用``pet store``的API。

查看``index.html``源码，其中有这样的代码：
```
<script>
window.onload = function() {
  
  // Build a system
  const ui = SwaggerUIBundle({
    url: "http://petstore.swagger.io/v2/swagger.json",
    dom_id: '#swagger-ui',
    deepLinking: true,
    presets: [
      SwaggerUIBundle.presets.apis,
      SwaggerUIStandalonePreset
    ],
    plugins: [
      SwaggerUIBundle.plugins.DownloadUrl
    ],
    layout: "StandaloneLayout"
  })
  window.ui = ui
}
</script>
```

其中``url: "http://petstore.swagger.io/v2/swagger.json"``一行是关键。   

SwaggerUIBundle()方法将从该URL读取并解析swagger配置文件来生成页面。因此，只要将其参数中的``url``替换成自己的``swagger.json``或``swagger.yml``，那么页面上显示的就是我们的API。    

明确swagger原理后，唯一的问题就是**如何生成我们自己的swagger.yml**

## 2. 生成swagger.yml

生成swagger.yml配置文件有两种方式：手动编写配置文件，或通过工具自动生成。  

我推荐采用手动编写配置文件的方式，首先是因为swagger配置文件并不复杂，其次也是为了让代码保持干净。



### 2.1 自动化生成swagger配置文件

有一些工具够为Go程序生成swagger配置文件，例如：

* [go-swagger/go-swagger](https://github.com/go-swagger/go-swagger)
* [yvasiyarov/swagger](https://github.com/yvasiyarov/swagger)

这些工具大同小异。

首先，要求在代码中手动添加特定的注释。例如，下面是``go-swagger``给一个http API添加的注释：

```
    // swagger:operation GET /pets getPet
    //
    // Returns all pets from the system that the user has access to
    //
    // Could be any pet
    //
    // ---
    // produces:
    // - application/json
    // - application/xml
    // - text/xml
    // - text/html
    // parameters:
    // - name: tags
    //   in: query
    //   description: tags to filter by
    //   required: false
    //   type: array
    //   items:
    //     type: string
    //   collectionFormat: csv
    // - name: limit
    //   in: query
    //   description: maximum number of results to return
    //   required: false
    //   type: integer
    //   format: int32
    // responses:
    //   '200':
    //     description: pet response
    //     schema:
    //       type: array
    //       items:
    //         "$ref": "#/definitions/pet"
    //   default:
    //     description: unexpected error
    //     schema:
    //       "$ref": "#/definitions/errorModel"
    mountItem("GET", basePath+"/pets", nil)
```

然后，通过工具提供的命令分析代码中的注释，生成swagger配置。例如，``go-swagger``提供的``swagger generate``命令可生成swagger配置：    
```
swagger generate spec -i ./swagger.yml -o ./swagger.json
```

自动化生成swagger.yml配置的好处是swagger配置与代码容易保持一致。但带来问题是**要在所有controller方法、model类上加入大量的swagger注释，代码会变得很长很丑**。       

### 2.2 手动编写swagger.yml

swagger配置文件很简单，尤其是yaml配置方式语义非常清晰明确。 

编写时可借助[在线的swagger配置编辑器](http://editor.swagger.io)，但需要注意的是：在线编辑器的域名是``editor.swagger.io``，后台必须要配置好后端的CORS策略才能在线调用后端API。    

## 3. swagger服务部署

swagger-ui是一个纯前端项目，部署到Nginx、Apache、Tomcat等服务器均可。    

由于我们后端采用Go编写，因此这里仅介绍``go run``部署方式。    

### 3.1 通过go run部署swagger

项目结构如下：    

```
.
├── main.go
├── 主项目其他包
│
└── swagger
    ├── swagger.go
    ├── swagger.yml
    └── swagger-ui
        ├── favicon-16x16.png
        ├── favicon-32x32.png
        ├── index.html
        ├── oauth2-redirect.html
        ├── swagger-ui-bundle.js
        ├── swagger-ui-bundle.js.map
        ├── swagger-ui.css
        ├── swagger-ui.css.map
        ├── swagger-ui.js
        ├── swagger-ui.js.map
        ├── swagger-ui-standalone-preset.js
        └── swagger-ui-standalone-preset.js.map
```

顶层目录下是后端项目代码，``main.go``是后端项目的主函数；   

在``swagger``目录下：

* ``swagger.yml``是自行编写的swagger配置；
* ``swagger-ui``目录是是从swagger-ui项目dist目录移过来，仅将配置文件的url改成了我们自行编写的yaml文件；
* ``swagger.go``是swagger文档的主函数，代码如下：

```
import (
	"flag"
	"log"
	"net/http"
)

func main() {
	port := flag.String("p", "8100", "port to serve on")
	directory := flag.String("d", ".", "the directory of static file to host")
	flag.Parse()

	http.Handle("/", http.FileServer(http.Dir(*directory)))

	log.Printf("Serving %s on HTTP port: %s\n", *directory, *port)
	log.Fatal(http.ListenAndServe(":"+*port, nil))
}
```
``swagger.go``实际上就是启动了一个文件服务器，让用户可通过浏览器访问``swagger``目录下的内容。    

通过``go run swagger.go``启动swagger项目，在浏览器中访问``http://[IP]:8100/swagger-ui``即可看到swagger页面。    


### 3.2 解决swagger与主项目跨域的问题

即使``swagger``与后端项目部署在同一机器上，但由于端口不同，仍然会出现跨域问题。    

#### 3.2.1 解决方式1：后端统一添加允许跨域的response header

修改response header，增加``Access-Control-Allow-Origin``和``Access-Control-Allow-Methods``配置：

```
w.Header().Set("Access-Control-Allow-Origin", "*")
w.Header().Set("Access-Control-Allow-Headers", "Origin, X-Requested-With, Content-Type, Accept, Z-Key")
w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
w.Header().Set("Content-Type", "application/json")
```

对于``DELETE``请求，swagger前端还会发送``preflight request``（请求的动作为OPTIONS），根据response header判断资源是否允许跨域。    
因此，对所有REST资源的``OPTIONS``操作要特殊处理（可用通配符匹配所有资源的``OPTIONS``操作，统一返回设置好跨域header的空响应）。      

#### 3.2.2 解决方式2：通过部署到统一机器和端口

通过Nginx，将路径包含``/swagger-ui``的请求转发给swagger，将其他请求转发给后端项目。    

由于后端与前端对外共用同一IP和端口，因此不存在跨域。    


