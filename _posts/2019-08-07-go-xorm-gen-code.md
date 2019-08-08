---
layout: post
title: Xorm Generate Code
subtitle: xorm 根据数据库生成struct
date: 2019-08-08
categories: go
cover: 
tags: all xorm
---

> 每一个orm代码生成器都需要改造

数据库对应的models书写总是一个及其枯燥的工作，即使你会一些编辑器的批处理操作

orm一般都自带一个代码生成工具，xorm也不例外

首先需要安装一些工具
```go
go get github.com/go-xorm/cmd/xorm
go get github.com/go-sql-driver/mysql  //MyMysql
go get github.com/ziutek/mymysql/godrv  //MyMysql
go get github.com/lib/pq  //Postgres
go get github.com/mattn/go-sqlite3  //SQLite
go get github.com/go-xorm/xorm
```

之后我们可以执行命令生成代码
> xorm reverse mysql name:password@(ip:port)/xxx?charset=utf8mb4 /cmd/xorm/templates/goxorm

`/cmd/xorm/templates/goxorm` 指代码生成模板，一般位于`$GOPATH/github.com/go-xorm/cmd/xorm/templates/goxorm` 目录下,如果开启了go modules，则有可能位于 `$GOPATH\pkg\mod\github.com\go-xorm\cmd\xorm@v0.0.0-20190426080617-f87981e709a1\templates\goxorm`

模板包含一个配置文件config
```go
lang=go
genJson=0 // 0-不生成json 1-生成json
prefix=cos_
ignoreColumnsJSON=
created=
updated=
deleted=
```
生成代码示例：
```go
package models

type Exception struct {
	Id         int64  `json:"id" xorm:"pk autoincr BIGINT(20)"`
	ComId      int64  `json:"com_id" xorm:"index BIGINT(20)"`
}
```

我们的编程规范中json字段一般采用小驼峰形势，很明显，代码中的json并非我们需要的格式。并且xorm 标签中没有指明column名称，如果我们改动struct成员名，则会导致映射失败

基于xorm工具库修改困难，只好自己造一个轮子，在工具外套一层代码改造，同时，可以将个人所需的基本方法添加进去

创建项目xorm-gen:
```go
var dataSource, tempPath, codePath string

func init() {
	flag.StringVar(&dataSource, "dataSource", "", "database source path(eg: root:admin@(ip:port)/database?charset=utf8mb4)")
	flag.StringVar(&tempPath, "temp", "", "template file path")
	//flag.StringVar(&codePath, "generatedPath", "./models", "generate code to direction")
	flag.Parse()

}

func main() {
	if dataSource == "" {
		fmt.Println("dataSource is necessary")
		return
	}
	if tempPath == "" {
		fmt.Println("temp is necessary")
		return
	}

	cmd := exec.Command("xorm", "reverse", "mysql", dataSource, tempPath)
	out, err := cmd.Output()
	fmt.Println(string(out))
	if err != nil {
		fmt.Println("exec failed: ", err)
		return
	}
	fmt.Println("start")
	var files []os.FileInfo
	for {
		files, err = ioutil.ReadDir("./models")
		if err == nil {
			break
		}
		fmt.Println(err)
		time.Sleep(time.Millisecond * 100)
	}

	for _, f := range files {
		err = modifyFile(f.Name())
		if err != nil {
			fmt.Println(err)
			return
		}
	}
}

// 正则表达式用的不熟练，直接强行判断了
func modifyFile(filename string) error {
	buf, err := ioutil.ReadFile("./models/" + filename)
	if err != nil {
		return err
	}
	reader := bufio.NewReader(bytes.NewReader(buf))

	var desc string
	var structName string
	start := false
	var id string
	l, _, err := reader.ReadLine()
	for l != nil && err == nil {
		line := string(l)
		if strings.HasPrefix(line, "type") {
			structName = strings.TrimSpace(strings.TrimPrefix(strings.TrimSuffix(line, "struct {"), "type"))
			start = true
		} else if line = strings.TrimSpace(line); start && len(line) > 1 {
			if strings.Contains(line, "json") {
				arr := splitLine(line)
				col := strings.Split(strings.TrimPrefix(arr[2], "`json:\""), "\"")[0]
				if col != "-" {
					json := parseCamelCase(col)
					line = strings.Replace(line, col, json, -1)
					line = strings.Replace(line, "xorm:\"", "xorm:\""+col+" ", -1)
				}
			}
			if id == ""{
				arr := splitLine(line)
				id = arr[0]
			}

			line = "\t" + line
		}
		if desc == "" {
			desc = line
		} else {
			desc += "\n" + line
		}

		l, _, err = reader.ReadLine()
	}
	desc += fmt.Sprintf(temp, structName, strings.TrimSuffix(filename, ".go"), structName, id)

	return ioutil.WriteFile("./models/"+filename, []byte(desc), 0666)
}

func TestSplit(t *testing.T) {
	fmt.Println(splitLine("	Id  int64    fsdfdfdfsdfsdfdffd  adfsdf  asdfsdf"))
}

func splitLine(line string) (arr []string) {
	start := -1
	hasStart := false
	num := 0
	for i := range line {
		if !hasStart && line[i] != ' ' {
			start = i
			hasStart = true
		}
		if hasStart && line[i] == ' ' {
			arr = append(arr, line[start:i])
			hasStart = false
			num++
		}
		if num == 2 {
			arr = append(arr, strings.TrimSpace(line[i:]))
			break
		}
	}
	return
}

func parseCamelCase(s string) string {
	ifToUp := false
	var camelString strings.Builder
	for _, char := range s {
		if char == '_' {
			ifToUp = true
		} else {
			if ifToUp && char <= 122 && char >= 97 {
				camelString.WriteByte(byte(char - 32))
			} else {
				camelString.WriteByte(byte(char))
			}
			ifToUp = false
		}

	}
	return camelString.String()
}

var temp = `

func (u *%s) TableName() string {
	return "%s"
}

func (u *%s) GetId() interface{} {
	return u.%s
}
`
```

执行`go install`

然后执行命令
> xorm-gen -dataSource=name:password@(ip:port)/xxx?charset=utf8mb4 -temp=templates/path

代码生成结果：
```go
type Exception struct {
	Id         int64  `json:"id" xorm:"id pk autoincr BIGINT(20)"`
	ComId      int64  `json:"comId" xorm:"com_id index BIGINT(20)"`
	Type       string `json:"type" xorm:"type VARCHAR(20)"`
	Except     string `json:"except" xorm:"except LONGTEXT"`
	CreateTime string `json:"createTime" xorm:"create_time CHAR(19)"`
}

func (u *Exception) TableName() string {
	return "exception"
}

func (u *Exception) GetId() interface{} {
	return u.Id
}
```