### YQL(Yet another-Query-Language)

[![Build Status](https://www.travis-ci.org/caibirdme/yql.svg?branch=master)](https://www.travis-ci.org/caibirdme/yql)
[![GoDoc](https://godoc.org/github.com/caibirdme/yql?status.svg)](https://godoc.org/github.com/caibirdme/yql)

YQL 非常类似于 sql 的 `where` 部分。你可以把它看作另一个 sql，它也支持两个集合之间的比较。YQL 几乎没有新的概念，所以在阅读了这些示例之后，您可以很好地使用它。
虽然它是为规则引擎设计的，但它可以广泛应用于代码逻辑中。

### Install

`go get github.com/caibirdme/yql`

### Exmaple

更多实例请在 `yql_test.go` 和 godoc 中查看。

``` go
	rawYQL := `name='deen' and age>=23 and (hobby in ('soccer', 'swim') or score>90))`
	result, _ := yql.Match(rawYQL, map[string]interface{}{
		"name":  "deen",
		"age":   int64(23),
		"hobby": "basketball",
		"score": int64(100),
	})
	fmt.Println(result)
	rawYQL = `score ∩ (7,1,9,5,3)`
	result, _ = yql.Match(rawYQL, map[string]interface{}{
		"score": []int64{3, 100, 200},
	})
	fmt.Println(result)
	rawYQL = `score in (7,1,9,5,3)`
	result, _ = yql.Match(rawYQL, map[string]interface{}{
		"score": []int64{3, 5, 2},
	})
	fmt.Println(result)
	rawYQL = `score.sum() > 10`
	result, _ = yql.Match(rawYQL, map[string]interface{}{
		"score": []int{1, 2, 3, 4, 5},
	})
	fmt.Println(result)
	//Output:
	//true
	//true
	//false
	//true
```

在大多数情况下，你可以使用 `Rule` 来缓存 AST，然后使用 `Match` 来获取结果，这可以避免成千上万的重复解析过程。

```go
	rawYQL := `name='deen' and age>=23 and (hobby in ('soccer', 'swim') or score>90)`
	ruler,_ := yql.Rule(rawYQL)

	result, _ := ruler.Match(map[string]interface{}{
		"name":  "deen",
		"age":   23,
		"hobby": "basketball",
		"score": int64(100),
	})
	fmt.Println(result)
	result, _ = ruler.Match(map[string]interface{}{
		"name":  "deen",
		"age":   23,
		"hobby": "basketball",
		"score": int64(90),
	})
	fmt.Println(result)
	//Output:
	//true
	//false
```

虽然要匹配的数据类型是 `map[string]interface{}`，但只支持 5 种类型:

* int
* int64
* float64
* string
* bool

### Helpers

在 `score.sum() > 10` 中， `sum` 是一个辅助函数，它将 score 中的所有数字加起来，这也意味着 score 的类型必须是 []int，[]int64 或 []float64 之一。

这个 repo 是在早期阶段，所以现在只有几个辅助函数，请随意创建一个关于您的需求的问题。支持的帮手如下:
* sum: ...
* count: return the length of a slice or 1 if not a slice
* avg: return the average number of a slice(`float64(total)/float64(len(slice))`)
* max: return the maximum number in a slice
* min: return the minimum number in a slice

### Usage scenario

显然，它很容易在规则引擎中使用。

```go
var handlers = map[int]func(map[string]interface{}){
	1: sendEmail,
	2: sendMessage,
	3: alertBoss,
}

data := resolvePostParamsFromRequest(request)
rules := getRulesFromDB(sql)

for _,rule := range rules {
	if success,_ := yql.Match(rule.YQL, data); success {
		handler := handlers[rule.ID]
		handler(data)
		break
	}
}
```

此外，它可以用于你的日常工作中，这可以显著减少深嵌入 `if else` 语句:

```go
func isVIP(user User) bool {
	rule := fmt.Sprintf("monthly_vip=true and now<%s or eternal_vip=1 or ab_test!=false", user.ExpireTime)
	ok,_ := yql.Match(rule, map[string]interface{}{
		"monthly_vip": user.IsMonthlyVIP,
		"now": time.Now().Unix(),
		"eternal_vip": user.EternalFlag,
		"ab_test": isABTestMatched(user),
	})
	return ok
}
```

甚至如果你不想手动编写它，你也可以使用 `json.Marshal` 来生成 map[string]interface{}。确保结构标记应该与 rawYQL 中的名称相同。

### Syntax

查看 [grammar file](./internal/grammar/Yql.g4)

### Compatibility promise

`Match` API 现在是稳定的。它的语法不会再改变了，我接下来只会做的是优化，加速和添加更多的辅助函数，如果需要的话。

### Further Trial

虽然创建一个健壮的新的 Go 编译器比较困难，但仍然有一些有趣的事情可以做。例如，在 Go 中 引入 lambda 函数，它可能看起来像:

```go
var scores = []int{1,2,3,4,5,6,7,8,9,10}
newSlice := yql.Filter(`(v) => v % 2 == 0`).Map(`(v) => v*v`).Call(scores).Interface()
//[]int{4,16,36,64,100}
```

如果 lambda 函数不会一直更改，那么可以像 opcode 一样缓存它，这与编译后的代码一样快。在大多数情况下，谁在乎呢?

这不容易，但很有趣，不是吗?欢迎加入我的团队，提出一些问题，和我讨论你的想法。也许有一天它会成为像 javascript 中的[babel](http://babeljs.io/)那样的预编译工具。
  
#### Attention

`Lambda expression` 目前处于非常早的阶段，**别在生产环境中使用它**。

可以先看看 [test case](/lambda/lambda_test.go)

```go
type Student struct {
	Age  int
	Name string
}

var students = []Student{
	Student{
		Name: "deen",
		Age:  24,
	},
	Student{
		Name: "bob",
		Age:  22,
	},
	Student{
		Name: "alice",
		Age:  23,
	},
	Student{
		Name: "tom",
		Age:  25,
	},
	Student{
		Name: "jerry",
		Age:  20,
	},
}

t = yql.Filter(`(v) => v.Age > 23 || v.Name == "alice"`).Call(students).Interface()
res,_ := t.([]Student)
// res: Student{"deen",24} Student{"alice", 23} Student{"tom", 25}
```

链式语法

```go
dst := []int{1, 2, 3, 4, 5, 6, 7}
r := Filter(`(v) => v > 3 && v <= 7`).Map(`(v) =>  v << 2`).Filter(`(v) => v % 8 == 0`).Call(dst)
s, err := r.Interface()
ass := assert.New(t)
ass.NoError(err)
ass.Equal([]int{16, 24}, s)
```
