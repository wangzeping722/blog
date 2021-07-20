---
title: "利用comment来做些有趣的事情"
date: 2021-07-20T10:03:51+08:00
draft: false
---

在语言层面上，go语言不像java那样灵活，能够利用annotation来做很多事情。但是go提供了对ast（抽象语法树）的支持 , 通过 go官方提供的工具包可以解析并生成 ast。

![https://blog-wero.oss-cn-shanghai.aliyuncs.com/img/ast%E5%88%86%E6%9E%90.png](https://blog-wero.oss-cn-shanghai.aliyuncs.com/img/ast%E5%88%86%E6%9E%90.png)

## 节点

我们知道抽象语法树都是由节点(Node)构成，Golang的AST主要由三种节点构成：分别是表达式和类型节点(Expr)、语句节点(Stmt)和声明节点(Decl)。所有的节点都包含标识其在源代码中开头和结尾位置的信息。

```go
// Node AST树节点
type Node interface {
  Pos() token.Pos // 首字母的位置
  End() token.Pos // 结束位置
}

// 三个主要的节点类型
// 1. 表达式和类型节点
type Expr interface {
  Node
  exprNode()
}

// 2. 语句节点
type Stmt interface {
  Node
  stmtNode()
}

// 3. 声明节点
type Decl interface {
  Node
  declNode()
}
```

例子：[https://astexplorer.net/](https://astexplorer.net/)

```go
// 1. 新建文件集合
fSet := token.NewFileSet()

// 2. 解析源文件，返回一个 ast.File
f, err := parser.ParseFile(fSet, fileName, nil, parser.ParseComments)

// 3. 利用 ast.File 来做有意思的事情
...
...
```

其中:

- 通过token.NewFileSet来生成一个FileSet对象，这个对象会保存详细的源码位置信息
- file是文件名, 当src为nil时会读取该文件的内容；当src是字节数组/字符串时会直接将src的内容作为文件内容解析
- 需要通过ParseComments参数告知parser把注释信息也一并记录下来，因为很多有意思的功能都可以通过注释来实现。

一些例子。

...

---

回到正题，接下来我们分析一下如何基于comment进行做一些有趣的事情。

### 案例：[swag](https://github.com/swaggo/swag)

Swag将Go的注释转换为Swagger2.0文档。并且为流行的 Go Web Framework 创建了各种插件，这样可以与现有Go项目快速集成（使用Swagger UI）。其实这个工具的原理就是利用ast解析comment，然后根据规则生成api文档。

swag 利用 [ParseRouterAPIInfo](https://github.com/swaggo/swag/blob/17c1766b6349df2ab31d52e87be0ae3abca0d239/parser.go#L559) 函数来分析 `ast.FuncDecl` ，并解析func上面的注解（例如：@Param，@Router）。

```go
// ParseRouterAPIInfo parses router api info for given astFile
func (parser *Parser) ParseRouterAPIInfo(fileName string, astFile *ast.File) error {
	for _, astDescription := range astFile.Decls {
		switch astDeclaration := astDescription.(type) {
		case *ast.FuncDecl:
			if astDeclaration.Doc != nil && astDeclaration.Doc.List != nil {
				operation := NewOperation(parser, SetCodeExampleFilesDirectory(parser.codeExampleFilesDir)) //for per 'function' comment, create a new 'Operation' object
				for _, comment := range astDeclaration.Doc.List {
					if err := operation.ParseComment(comment.Text, astFile); err != nil {
						return fmt.Errorf("ParseComment error in file %s :%+v", fileName, err)
					}
				}
				var pathItem spec.PathItem
				var ok bool

				if pathItem, ok = parser.swagger.Paths.Paths[operation.Path]; !ok {
					pathItem = spec.PathItem{}
				}
				switch strings.ToUpper(operation.HTTPMethod) {
				case http.MethodGet:
					pathItem.Get = &operation.Operation
				case http.MethodPost:
					pathItem.Post = &operation.Operation
				case http.MethodDelete:
					pathItem.Delete = &operation.Operation
				case http.MethodPut:
					pathItem.Put = &operation.Operation
				case http.MethodPatch:
					pathItem.Patch = &operation.Operation
				case http.MethodHead:
					pathItem.Head = &operation.Operation
				case http.MethodOptions:
					pathItem.Options = &operation.Operation
				}

				parser.swagger.Paths.Paths[operation.Path] = pathItem
			}
		}
	}

	return nil
}
```

pathItem 存储了路由信息，并且在后面会被处理，然后输出成json。

```go
type Swagger struct {
 VendorExtensible
 SwaggerProps
}

type SwaggerProps struct {
	ID                  string                 `json:"id,omitempty"`
	Consumes            []string               `json:"consumes,omitempty"`
	Produces            []string               `json:"produces,omitempty"`
	Schemes             []string               `json:"schemes,omitempty"`
	Swagger             string                 `json:"swagger,omitempty"`
	Info                *Info                  `json:"info,omitempty"`
	Host                string                 `json:"host,omitempty"`
	BasePath            string                 `json:"basePath,omitempty"`
	Paths               *Paths                 `json:"paths"`
	Definitions         Definitions            `json:"definitions,omitempty"`
	Parameters          map[string]Parameter   `json:"parameters,omitempty"`
	Responses           map[string]Response    `json:"responses,omitempty"`
	SecurityDefinitions SecurityDefinitions    `json:"securityDefinitions,omitempty"`
	Security            []map[string][]string  `json:"security,omitempty"`
	Tags                []Tag                  `json:"tags,omitempty"`
	ExternalDocs        *ExternalDocumentation `json:"externalDocs,omitempty"`
}

type Paths struct {
	VendorExtensible
	Paths map[string]PathItem `json:"-"` // custom serializer to flatten this, each entry must start with "/"
}
```

### 实战：gengen

在日常开发中，我们经常会用写http errorcode，代码写一遍，json文件还有再写一遍。。。好繁琐啊，开发一下工具吧。

gengen目前支持根据常量定义，生成对应的code到字符串的map映射，也能够生成json文件。

例如：

```go
package data

//go:generate gengen -type json -generator ErrCode -o zh.json
//go:generate gengen -type go -generator ErrCode

// 公共
const (
	SystemFailed       = 10000000 // 系统繁忙，请稍后重试
	RecordNotFound     = 10000001 // 数据不存在
	RecordExportFailed = 10000002 // 数据导出失败
	FileOverMaxSize    = 10000003 // 文件不能超过20M
)
```

根据源代码生成如下两份代码：

```go
// Code generated by GENGEN DO NOT EDIT
// data const code errcode msg
package data

// UnKnownErr if code is not found, GetMsg will return this
const UnKnownErr = "未知错误"

var messages = map[int]string{
	10000000: "系统繁忙，请稍后重试", 
	10000001: "数据不存在", 
	10000002: "数据导出失败", 
	10000003: "文件不能超过20M"
}

// GetMsg get error msg
func GetMsg(code int) string {
	var (
		msg string
		ok  bool
	)
	if msg, ok = messages[code]; !ok {
		msg = UnKnownErr
	}
	return msg
}
```

```json
{
	"10000000": "系统繁忙，请稍后重试",
	"10000001": "数据不存在",
	"10000002": "数据导出失败",
	"10000003": "文件不能超过20M",
	"10100001": "资质账号绑定失败"
}
```

见代码

