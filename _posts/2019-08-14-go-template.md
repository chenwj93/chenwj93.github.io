---
layout: post
title: Go Template Elementary
subtitle: go 模板入门
date: 2019-08-14
categories: go
cover: 
tags: all template
---

> 基础

# text template
##### 操作
模板以`\{\{`和`\}\}`分隔文本与操作，以`.`取当前对象，例如
`hello \{\{.Lan\}\}`,解析会将Lan对应的值渲染到模板（假如为"go"），则输出`hello go`
模板支持 if、else、range 等命令
##### 变量
`\{\{$value := 1\}\}`

$value 为变量名，1为值

##### 函数
模板本身可编程性差，但是可以调用go中已定义函数，支持多个输入参数，1个或2个（一个结果和一个错误信息）输出参数

`\{\{Func param1 param2\}\}`

模板默认实现了以下函数
```go
var builtins = FuncMap{
	"and":      and,
	"call":     call,
	"html":     HTMLEscaper,
	"index":    index,
	"js":       JSEscaper,
	"len":      length,
	"not":      not,
	"or":       or,
	"print":    fmt.Sprint,
	"printf":   fmt.Sprintf,
	"println":  fmt.Sprintln,
	"urlquery": URLQueryEscaper,

	// Comparisons
	"eq": eq, // ==
	"ge": ge, // >=
	"gt": gt, // >
	"le": le, // <=
	"lt": lt, // <
	"ne": ne, // !=
}
```

##### 管道`|` 
管道与linux命令类似，可以链接多个命令，并将前一个命令的结果作为后一个命令的最后一个参数

##### 模板嵌套
```go
`\{\{define "T1"\}\}ONE\{\{end\}\}
\{\{define "T2"\}\}TWO\{\{end\}\}
\{\{define "T3"\}\}\{\{template "T1"\}\} \{\{template "T2"\}\}\{\{end\}\}
\{\{template "T3"\}\}`
```
以上定义了T1、T2、T3三个模板，T3嵌套了1、2，并在最后调用了T3

##### 示例
```go
var tempvar = `
\{\{range .\}\}
\{\{$table := .\}\}
type \{\{$table.Name "enc" | Upper \}\} struct {
	\{\{range $index, $element := .Col\}\}
	\{\{$col := $element\}\} \{\{$index\}\}  \{\{Mapper $col.colName\}\} \{\{Tag $col\}\}
	\{\{end\}\}
}
\{\{end\}\}

`

type Table struct {
	name string
	Col  []map[string]interface{}
}

func (t *Table)Name(prefix string) string {
	return prefix + "_" + t.name
}

func TestTemp(t1 *testing.T) {
	t := template.New("test")
	t.Funcs(map[string]interface{}{"Mapper": Mapper, "Tag": Tag, "Upper": Upper})
	temp, err := t.Parse(tempvar)
	if err != nil {
		panic(err)
	}
	var res strings.Builder
	param := []Table{
		{
			name: "Test",
			Col: []map[string]interface{}{
				{
					"colName": "id",
					"tag":     "`column(id)`",
				},
				{
					"colName": "name",
					"tag":     "`column(name)`",
				},
			},
		},{
			name: "Test2",
			Col: []map[string]interface{}{
				{
					"colName": "id2",
					"tag":     "`column(id_2)`",
				},
				{
					"colName": "name_2",
					"tag":     "`column(name_2)`",
				},
			},
		}
	}
	err = temp.Execute(&res, &param)
	//err = temp.ExecuteTemplate(&res, "T1", param) // 按照模板名执行
	if err != nil {
		panic(err)
	}

	fmt.Println(res.String())
}

func Mapper(s string) string {
	return s + "_"
}

func Tag(m map[string]interface{}) string {
	return "tag_" + m["tag"].(string)
}

func Upper(s string) string {
	return strings.ToUpper(s)
}

=====================================================
=== RUN   TestTemp
type ENC_TEST struct {
	
	 0  id_ tag_`column(id)`
	
	 1  name_ tag_`column(name)`
	
}

type ENC_TEST2 struct {
	
	 0  id2_ tag_`column(id_2)`
	
	 1  name_2_ tag_`column(name_2)`
	
}
--- PASS: TestTemp (0.00s)
PASS
```

**注意：** 如果需要在模板中调用对象方法，则需要保证Execute函数入参类型与参数receiver一致