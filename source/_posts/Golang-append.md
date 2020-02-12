---
title: Golang 内置函数 append() 踩坑
date: 2020-02-12 10:10:18
tags:
- Golang
- Slice
categories:
- Golang
- 踩坑
---

# Golang 内置函数 append() 踩坑

## 1 场景重现

> 用 Golang 完成 [leetcode-78.子集](https://leetcode-cn.com/problems/subsets/) 的时候，发现执行某个测试用例输出了错误的答案，子集中包含了两组相同的集合。

### 1.1 解题

题目:
```
给定一组不含重复元素的整数数组 nums，返回该数组所有可能的子集（幂集）。

说明：解集不能包含重复的子集。

示例:
输入: nums = [1,2,3]
输出:
[
  [3],
  [1],
  [2],
  [1,2,3],
  [1,3],
  [2,3],
  [1,2],
  []
]

来源：力扣（LeetCode）  
链接：https://leetcode-cn.com/problems/subsets  
著作权归领扣网络所有。商业转载请联系官方授权，非商业转载请注明出处。  
```

我写的一个解答:
```golang
func subsets(nums []int) [][]int {
	if nums == nil {
		return [][]int{}
	}
	sort.Ints(nums)
	sets := [][]int{{}}
	for i := 0; i < len(nums); i++ {
		sets = append(sets, sub([]int{}, nums, i+1)...)
	}
	return sets
}

func sub(prefix, options []int, need int) [][]int {
	need--
	var sets [][]int
	for i := 0; i < len(options); i++ {
		if need > 0 {
			sets = append(sets, sub(append(prefix, options[i]), options[i+1:], need)...)
		} else {
			sets = append(sets, append(prefix, options[i]))
		}
	}
	return sets
}
```

测试代码:
```golang
type args struct {
	nums []int
}

var tests = []struct {
	name string
	args args
	want [][]int
}{
	{"sample-7", args{[]int{5, 1, 2, 3, 4}}, [][]int{{}, {5}, {1}, {1, 5}, {2}, {2, 5}, {1, 2}, {1, 2, 5}, {3}, {3, 5}, {1, 3}, {1, 3, 5}, {2, 3}, {2, 3, 5}, {1, 2, 3}, {1, 2, 3, 5}, {4}, {4, 5}, {1, 4}, {1, 4, 5}, {2, 4}, {2, 4, 5}, {1, 2, 4}, {1, 2, 4, 5}, {3, 4}, {3, 4, 5}, {1, 3, 4}, {1, 3, 4, 5}, {2, 3, 4}, {2, 3, 4, 5}, {1, 2, 3, 4}, {1, 2, 3, 4, 5}}},
	{"sample-0", args{[]int{1, 2, 3}}, [][]int{{3}, {1}, {2}, {1, 2, 3}, {1, 3}, {2, 3}, {1, 2}, {}}},
	{"case-0", args{[]int{1, 2}}, [][]int{{1}, {2}, {1, 2}, {}}},
}

func Test_subsets(t *testing.T) {
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			if got := p78_1.Subsets(tt.args.nums); !util.EqualsIgnoreOrder(got, tt.want) {
				t.Errorf("subsets() = %v, want %v", got, tt.want)
			}
		})
	}
}

```

测试结果:  
![output](https://wuweijie.oss-cn-shenzhen.aliyuncs.com/blog/resource/2020/02/golang-append/output.png)
"sample-0", "case-0"的执行结果都正确；  
但"sample-7"的实际输出结果少了子集`[1,2,3,4]`，出现了两个`[1,2,3,5]`

### Debug

![step1](https://wuweijie.oss-cn-shenzhen.aliyuncs.com/blog/resource/2020/02/golang-append/step1.png)
将代码运行到输出`[1,2,3,5]`前一步，发现子集变量sets里面已经有一个切片`[1,2,3,4]`，为什么最后的答案中没有？  
继续执行下一步，sets的预期结果是`[[1,2,3,4],[1,2,3,5]]`

![step2](https://wuweijie.oss-cn-shenzhen.aliyuncs.com/blog/resource/2020/02/golang-append/step2.png)
子集sets中确实新增了一个切片`[1,2,3,5]`，**但原本的`[1,2,3,4]`中第3个元素也变成了5**，这是WA的直接原因。  

### 解决方案

将prefix复制一份副本后，再进行append操作，提交结果为**AC**
```golang
func subsets(nums []int) [][]int {
	if nums == nil {
		return [][]int{}
	}
	sort.Ints(nums)
	sets := [][]int{{}}
	for i := 0; i < len(nums); i++ {
		sets = append(sets, sub([]int{}, nums, i+1)...)
	}
	return sets
}

func sub(prefix, options []int, need int) [][]int {
	need--
	var sets [][]int
	for i := 0; i < len(options); i++ {
		// todo take notes
		copied := make([]int, len(prefix))
		copy(copied, prefix)
		if need > 0 {
			sets = append(sets, sub(append(copied, options[i]), options[i+1:], need)...)
		} else {
			sets = append(sets, append(copied, options[i]))
		}
	}
	return sets
}
```

## 2 理解append()函数

### 2.1 测试append()函数

看下面这段测试代码，执行完毕后，a和b的中的元素分别是什么？
```golang
func TestAppend_0(t *testing.T) {
	base := make([]int, 0, 1)
	base = append(base, 100)
	fmt.Printf("len(base) = %v, cap(base) = %v\n", len(base), cap(base))
	// equivalent
	// base := []int{100}
	a := append(base, 200)
	b := append(base, 300)
	fmt.Printf("base = %v\na = %v\nb = %v\n", base, a, b)
	fmt.Printf("addr:\nbase = %p\n   a = %p\n   b = %p\n", base, a, b)
	fmt.Printf("len(a) = %v, cap(a) = %v\n", len(a), cap(a))
	fmt.Printf("len(b) = %v, cap(b) = %v\n", len(b), cap(b))
}

func TestAppend_1(t *testing.T) {
	base := make([]int, 0, 2)
	base = append(base, 100)
	fmt.Printf("len(base) = %v, cap(base) = %v\n", len(base), cap(base))
	a := append(base, 200)
	b := append(base, 300)
	fmt.Printf("base = %v\na = %v\nb = %v\n", base, a, b)
	fmt.Printf("addr:\nbase = %p\n   a = %p\n   b = %p\n", base, a, b)
	fmt.Printf("len(a) = %v, cap(a) = %v\n", len(a), cap(a))
	fmt.Printf("len(b) = %v, cap(b) = %v\n", len(b), cap(b))
}
```

运行结果：  
```
=== RUN   TestAppend_0
len(base) = 1, cap(base) = 1
base = [100]
a = [100 200]
b = [100 300]
addr:
base = 0xc00000c388
   a = 0xc00000c3a0
   b = 0xc00000c3b0
len(a) = 2, cap(a) = 2
len(b) = 2, cap(b) = 2

=== RUN   TestAppend_1
len(base) = 1, cap(base) = 2
base = [100]
a = [100 300]
b = [100 300]
addr:
base = 0xc00000c390
   a = 0xc00000c390
   b = 0xc00000c390
len(a) = 2, cap(a) = 2
len(b) = 2, cap(b) = 2
```

在`TestAppend_0`中，`base`切片容量已满，因此调用`append(base, ...)`产生的新切片会发生扩容操作。

在`TestAppend_1`中，`base`切片仍有空间，无需扩容，因此`append(base, ...)`操作基于切片原有的底层数组进行，`append(base, ...)`的返回值没有重新赋值给`base`，导致两次`append(base, ...)`都在原有数组同一个位置上操作。

### 2.2 回顾问题

现在回到WA的场景

![step2](https://wuweijie.oss-cn-shenzhen.aliyuncs.com/blog/resource/2020/02/golang-append/step2.png)

可以看到，`prefix`是一个`len:3, cap:4`的切片，因此，加入`sets`变量中的切片其实是同一块内存，当这个切片不需要扩容的时候，使用`append()`函数会修改原有的内存，导致原有的正确答案`[1,2,3,4]`变成了`[1,2,3,5]`。

## 3 疑问

### 3.1 切片地址相同，len却可以不同

回顾2.1节中的测试代码输出结果
```
=== RUN   TestAppend_1
len(base) = 1, cap(base) = 2
base = [100]
a = [100 300]
b = [100 300]
addr:
base = 0xc00000c390
   a = 0xc00000c390
   b = 0xc00000c390
len(a) = 2, cap(a) = 2
len(b) = 2, cap(b) = 2
```

结合定义在`runtime/slice.go`包中的切片的数据结构
```
package runtime

type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}
```

为何切片地址相同，`len()`函数的返回值可以不同？


