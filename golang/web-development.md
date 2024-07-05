# TIL - Web Development

My notes about some Web Development in Golang

## Beego Web Framework

Beego: https://beego.wiki

### How to add route in Beego

Install `f***ing` beego tool:

```shell
go install github.com/beego/bee/v2@latest
```

First, create routers definition in `router.go` file:

```go
// routers/router.go file content
package routers

import (
	"github.com/demo-api/src/controllers"
	"github.com/beego/beego/v2/server/web"
)

func Init(configDir string) {
	web.SetStaticPath("/swagger", "swagger")
	web.Router("/", &controllers.MainController{})
	ns := web.NewNamespace("/api/v1",
		web.NSNamespace("/resources",
			web.NSInclude(
				&controllers.APIResourceController{},
			),
		),
	)
	web.AddNamespace(ns)
}

```

Then add controller file:

```go
// controllers/resoure.go file content

// Get lists  resources
// @Title list
// @Description List  resources
// @Success 200 request success
// @Failure 400 request failure
// @router / [get]
func (c *APIResourceController) List() {
	status := RespJson{}
	status.Value1 = "abc"
	status.Value2 = "def"
	c.ServeJSONData(status)
}

// Get Get  resource by ID
// @Title list
// @Description Get  resource by ID
// @Success 200 request success
// @Failure 400 request failure
// @router /:id  [get]
func (c *APIResourceController) GetResource(id int) {
	log := logs.NewLogger(10000)
	status := RespJson{
		Value3: id + 1,
		Value2: "abc",
		Value1: "def",
	}
	log.Info("Check Data: %d", id)
	var content []byte
	var err error
	content, err = json.MarshalIndent(&status, "", "  ")
	if err == nil {
		log.Info("Check Data JSON: %s", string(content))
	} else {
		log.Error("Error when parsing Data JSON: %s", err)
	}
	c.Data["json"] = &status
	c.ServeJSON()
}


```

Run this command to generate route:

```go
bee generate routers -routersPkg=routers -ctrlDir=controllers
```
After this command, file `src/routers/commentsRouter.go` will be generated with content:

```go

package routers

import (
	beego "github.com/beego/beego/v2/server/web"
	"github.com/beego/beego/v2/server/web/context/param"
)

func init() {

    beego.GlobalControllerRouter["github.com/demo-api/src/controllers:APIResourceController"] = append(beego.GlobalControllerRouter["github.com/demo-api/src/controllers:APIResourceController"],
        beego.ControllerComments{
            Method: "List",
            Router: `/`,
            AllowHTTPMethods: []string{"get"},
            MethodParams: param.Make(),
            Filters: nil,
            Params: nil})

    beego.GlobalControllerRouter["github.com/demo-api/src/controllers:APIResourceController"] = append(beego.GlobalControllerRouter["github.com/demo-api/src/controllers:APIResourceController"],
        beego.ControllerComments{
            Method: "GetResource",
            Router: `/:id`,
            AllowHTTPMethods: []string{"get"},
            MethodParams: param.Make(
				param.New("id", param.InPath),
			),
            Filters: nil,
            Params: nil})

}

```


### Generate swagger path

```shell
bee run  -gendoc=true
```

## How to marshal struct to JSON output

https://stackoverflow.com/questions/26327391/json-marshalstruct-returns

You need to export the fields in TestObject by **capitalizing the first letter in the field name**. Change kind to Kind and so on.

```go
type TestObject struct {
 Kind string `json:"kind"`
 Id   string `json:"id,omitempty"`
 Name  string `json:"name"`
 Email string `json:"email"`
}
```
The encoding/json package and similar packages ignore unexported fields.

The `json:"..."` strings that follow the field declarations are struct tags. The tags in this struct set the names of the struct's fields when marshaling to and from JSON.