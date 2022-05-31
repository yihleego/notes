# Go

## 每日一题

### 数组的比较

```go
package main

import "fmt"

func main() {
	type pos [2]int
	a := pos{4, 5}
	b := pos{4, 5}
	fmt.Println(a == b)
}
```

数组是值类型，长度相等，每个元素也相等，数组就相等，所以结果是`true`。
