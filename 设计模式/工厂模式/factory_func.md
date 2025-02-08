## 工厂方法模式
工厂方法模式是简单工厂模式的扩展，它将对象的创建和使用分离，通过一个工厂接口来创建对象，使得代码更加灵活和易于扩展。


## 实例

```go

package factory_model

import (
	"fmt"
)

// Product 定义产品的接口
type ProductFunc interface {
	Use()
}

// ConcreteProductA 具体产品A
type ConcreteProductAFunc struct{}

func (p *ConcreteProductAFunc) Use() {
	fmt.Println("Using ConcreteProductA")
}

// ConcreteProductB 具体产品B
type ConcreteProductBFunc struct{}

func (p *ConcreteProductBFunc) Use() {
	fmt.Println("Using ConcreteProductB")
}

// Factory 定义工厂接口
type Factory interface {
	CreateProductFunc() Product
}

// ConcreteFactoryA 具体工厂A
type ConcreteFactoryA struct{}

func (f *ConcreteFactoryA) CreateProductFunc() Product {
	return &ConcreteProductA{}
}

// ConcreteFactoryB 具体工厂B
type ConcreteFactoryB struct{}

func (f *ConcreteFactoryB) CreateProductFunc() Product {
	return &ConcreteProductB{}
}

```

## 单测
```go
package factory_model

import (
	"testing"
)

func TestFactoryFuncModel(t *testing.T) {
	// 创建工厂
	factoryA := &ConcreteFactoryA{}
	productA := factoryA.CreateProductFunc()
	productA.Use() // 输出：Using ConcreteProductA
	// 创建产品A

	factoryB := &ConcreteFactoryB{}
	productB := factoryB.CreateProductFunc()
	productB.Use() // 输出：Using ConcreteProductB
	// 创建产品B
}

/*
Running tool: /opt/homebrew/bin/go test -timeout 30s -run ^TestFactoryFuncModel$ chain_of_duty_model/factory_model

=== RUN   TestFactoryFuncModel
Using ConcreteProductA
Using ConcreteProductB
--- PASS: TestFactoryFuncModel (0.00s)
PASS
ok      chain_of_duty_model/factory_model       0.276s
*/

```