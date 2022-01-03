---
layout: title
title: "golang常用包系列: resource"
date: 2021-11-04 14:06:44
tags:
  - golang
  - resource
categories:
  - 专业
  - 开发
  - golang
index_img: https://cdn.jsdelivr.net/gh/simonyangchao/resources@image/golang-1.png
banner_img: https://cdn.jsdelivr.net/gh/simonyangchao/resources@image/golang-1.png
---

# 简介

resource库封装了golang常用的数据量，包括三种格式：指数形式、二进制幂、十进制幂。Kubernetes中使用resource.Quantity存储和运算内存、cpu和存储等数据。

<!-- more -->

```
const (
	DecimalExponent = Format("DecimalExponent") // e.g., 12e6
	BinarySI        = Format("BinarySI")        // e.g., 12Mi (12 * 2^20)
	DecimalSI       = Format("DecimalSI")       // e.g., 12M  (12 * 10^6)
)
```



# 数据结构


Quantity本质上是数字的定点表示形式，提供了String()和AsInt64()访问器，和JSON, YAML marshal/unmarshal
```
type Quantity struct {
	// i is the quantity in int64 scaled form, if d.Dec == nil
	i int64Amount
	// d is the quantity in inf.Dec form if d.Dec != nil
	d infDecAmount
	// s is the generated value of this quantity to avoid recalculation
	s string
	// Change Format at will. See the comment for Canonicalize for
	// more details.
	Format
}
type infDecAmount struct {
	*inf.Dec
}
// Format lists the three possible formattings of a quantity.
type Format string
```

可以直接通过quantity.Format获取该Quantity的格式，比如DecimalExponent, BinarySI或DemicalSI

# 常用方法

## 1 创建并初始化Quantity

### 1.1 从字符串解析

```
func MustParse(str string) Quantity
func ParseQuantity(str string) (Quantity, error)
```
```
package main
import (
	"fmt"
	"k8s.io/apimachinery/pkg/api/resource"
)
func main() {
	memorySize := resource.MustParse("5Gi")
	fmt.Printf("memorySize = %v (%v)\n", memorySize.Value(), memorySize.Format)
	diskSize := resource.MustParse("5G")
	fmt.Printf("diskSize = %v (%v)\n", diskSize.Value(), diskSize.Format)
	cores := resource.MustParse("5300m")
	fmt.Printf("milliCores = %v (%v)\n", cores.MilliValue(), cores.Format)
	cores2 := resource.MustParse("5.4")
	fmt.Printf("milliCores = %v (%v)\n", cores2.MilliValue(), cores2.Format)
}
```

```
Output:
memorySize = 5368709120 (BinarySI)
diskSize = 5000000000 (DecimalSI)
milliCores = 5300 (DecimalSI)
milliCores = 5400 (DecimalSI)
```



### 1.2 从数字创建

```
func NewMilliQuantity(value int64, format Format) *Quantity
```

```
func NewQuantity(value int64, format Format) *Quantity
```

```
func NewScaledQuantity(value int64, scale Scale) *Quantity
//Scale用于获取和设置以10为基数的缩放值。为了简化，不包括以2为底的缩放。
type Scale int32
const (
	Nano  Scale = -9
	Micro Scale = -6
	Milli Scale = -3
	Kilo  Scale = 3
	Mega  Scale = 6
	Giga  Scale = 9
	Tera  Scale = 12
	Peta  Scale = 15
	Exa   Scale = 18
)
```

### 1.3 赋值操作

```
func (q *Quantity) Set(value int64)
```

```
func (q *Quantity) SetMilli(value int64)	// q的值为value * 1/1000
```

```
func (q *Quantity) SetScaled(value int64, scale Scale)	//value * 10^scale
```

## 2 运算

### 2.1 大小比较

#### 2.1.1 Quantity与Quantity比较

如果数量等于y，则Cmp返回0；如果数量小于y，则Cmp返回-1；如果数量大于y，则Cmp返回1

```
func (q *Quantity) Cmp(y Quantity) int
```

#### 2.1.2 Quantity与int64比较

```
func (q *Quantity) CmpInt64(y int64) int
```

#### 2.1.3 其他

判断是否为零值

```
func (q *Quantity) IsZero() bool
```

判断是否相等
```
func (q Quantity) Equal(v Quantity) bool
```

### 2.2 取整（四舍五入）操作

```
func (q *Quantity) RoundUp(scale Scale) bool
```

根据给定的scale缩放值(scale >=1)原地更新quantity，如果舍入操作导致精度丢失，则返回False。负数的RoundUp操作与证书相反，往远离0的方向：比如-9 scale 1四舍五入为-10

### 2.3 减法

Sub()从当前的当前值中减去y的数量，是一个in-place操作。如果当前值为零，则数量的格式将更新为y的格式。

```
func (q *Quantity) Sub(y Quantity)
```

### 2.4 取反

inplace操作，原地取反

```
func (q *Quantity) Neg()
```

### 2.5 自我类型更新

inplace操作，ToDec使用inf.Dec表示形式原地更新，并返回自身。

```
func (q *Quantity) ToDec() *Quantity {
	if q.d.Dec == nil {
		q.d.Dec = q.i.AsDec()
		q.i = int64Amount{}
	}
	return q
}
```

也就是将q.i转换为inf.Dec类型之后，赋值给q.d，然后q.i设置为0

## 3. 类型转换

### 3.1 转换为float64

可能丢失精度，如果quantity值小于float64 -Inf或者大于+Inf

```
func (q *Quantity) AsApproximateFloat64() float64
```

### 3.2 转换为inf.Dec

直接返回q.d.Dec：

```
func (q *Quantity) AsDec() *inf.Dec
// AsDec returns the quantity as represented by a scaled inf.Dec.
func (q *Quantity) AsDec() *inf.Dec {
	if q.d.Dec != nil {
		return q.d.Dec
	}
	q.d.Dec = q.i.AsDec()
	q.i = int64Amount{}
	return q.d.Dec
}
```

由于是直接返回指针，所以对q.AsDec()进行Dec封装的方法调用时，可以原地更新q

比如自定义Quantity的乘法：

```
// multiBy multiplies the provided quantity from the current value in place
func multiBy(x, y resource.Quantity) resource.Quantity {
	if y.CmpInt64(1) == 0 {
		return x
	}
	rst := resource.NewQuantity(x.Value(), x.Format)
	rst.AsDec().Mul(x.AsDec(), y.AsDec())
	return *rst
}
```

`rst.AsDec().Mul()`借助`inf.Dec`的`Mul()`方法，实现rst的原地更新

```

```

**inf.Dec**

inf包（类型inf.Dec）实现了“无限精度”十进制算法。“无限精度”描述了两个特征：十进制数表示实际上具有无限精度，并且不支持使用任何特定的固定精度进行计算。 （尽管对精度没有实际限制，但inf.Dec只能表示有限的小数。）

```
type Dec struct {
	unscaled big.Int
	scale    Scale
}
// Scale represents the type used for the scale of a Dec.
type Scale int32
```

Dec的数学值为:

```
unscaled * 10**(-scale)
```

请注意，不同的Dec表示形式可能具有相等的数学值，Dec的零值表示小数位数为0的值0。

```
unscaled  scale  String()
-------------------------
       0      0    "0"
       0      2    "0.00"
       0     -2    "0"
       1      0    "1"
     100      2    "1.00"
      10      0   "10"
       1     -1   "10"
```

### 3.3 转换为String

```
func (q *Quantity) String() string
```

### 3.4 转换为Int64

如果可以进行快速转换，则AsInt64将当前值的表示形式返回为int64。如果返回false，则调用者必须使用此数量的inf.Dec形式。

```
func (q *Quantity) AsInt64() (int64, bool)
```

### 3.5 转换为Scale

AsScale返回当前值，四舍五入到提供的缩放比例，如果缩放导致精度损失，则返回false

```
func (q *Quantity) AsScale(scale Scale) (CanonicalValue, bool)
// CanonicalValue允许将Quantity转换为字符串
type CanonicalValue interface {
	// AsCanonicalBytes returns a byte array representing the string representation
	// of the value mantissa and an int32 representing its exponent in base-10. Callers may
	// pass a byte slice to the method to avoid allocations.
	AsCanonicalBytes(out []byte) ([]byte, int32)
	// AsCanonicalBase1024Bytes returns a byte array representing the string representation
	// of the value mantissa and an int32 representing its exponent in base-1024. Callers
	// may pass a byte slice to the method to avoid allocations.
	AsCanonicalBase1024Bytes(out []byte) ([]byte, int32)
}
```

### 其他

```
func (q *Quantity) AsCanonicalBytes(out []byte) (result []byte, exponent int32)
```

```
func (q *Quantity) CanonicalizeBytes(out []byte) (result, suffix []byte)
```

```
// ToUnstructured实现value.UnstructuredConverter接口。
func (q Quantity) ToUnstructured() interface{}
```

```
// UnmarshalJSON实现json.Unmarshaller接口。
// TODO:删除对前导/尾随空格的支持
func (m *Quantity) Unmarshal(data []byte) error
```

```
// UnmarshalJSON实现json.Unmarshaller接口。
// TODO：删除对前导/尾随空格的支持
func (q *Quantity) UnmarshalJSON(value []byte) error
```



## 4. 获取quantity内部信息

### 取符号

```
func (q *Quantity) Sign() int
```

### 取值

返回未缩放的q值，该值四舍五入为最接近0的整数。与AsInt64()不同，不做是否丢失精度的判断

```
func (q *Quantity) Value() int64
```

```
func (q *Quantity) MilliValue() int64 // MilliValue返回ceil(q * 1000)的值；这可能会溢出int64;请首先调用Value()以验证数字是否足够小。
```

ref:

https://pkg.go.dev/k8s.io/apimachinery/pkg/api/resource
