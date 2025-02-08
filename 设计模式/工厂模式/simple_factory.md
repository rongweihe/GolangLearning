## 简单工厂模式

工厂模式（Factory Pattern）是一种创建型设计模式，用于封装对象的创建逻辑，从而让客户端代码无需直接调用构造函数来创建对象。工厂模式的核心思想是将对象的创建和使用分离，通过一个工厂类来负责创建对象，使得代码更加灵活和易于扩展。

工厂模式主要有两种类型：

- 简单工厂模式：通过一个工厂类直接创建对象，但不严格符合开闭原则。
- 工厂方法模式：通过定义一个工厂接口，让子类决定具体创建哪个对象，符合开闭原则。


```go
//简单工厂模式：通过一个工厂类直接创建对象，但不严格符合开闭原则。
package factory_model

import (
	"fmt"
)

// Product 定义产品的接口
type Product interface {
	Use()
}

// ConcreteProductA 具体产品A
type ConcreteProductA struct{}

func (p *ConcreteProductA) Use() {
	fmt.Println("Using ConcreteProductA")
}

// ConcreteProductB 具体产品B
type ConcreteProductB struct{}

func (p *ConcreteProductB) Use() {
	fmt.Println("Using ConcreteProductB")
}

// SimpleFactory 简单工厂类
type SimpleFactory struct{}

func (f *SimpleFactory) CreateProduct(productType string) Product {
	if productType == "A" {
		return &ConcreteProductA{}
	} else if productType == "B" {
		return &ConcreteProductB{}
	}
	return nil
}

```

## 单测
```go
package factory_model

import (
	"testing"
)

func TestSimpleFactoryModel(t *testing.T) {
	factory := &SimpleFactory{}
	productA := factory.CreateProduct("A")
	productA.Use() // 输出：Using ConcreteProductA
	productB := factory.CreateProduct("B")
	productB.Use() // 输出：Using ConcreteProductB
}

/*
Running tool: /opt/homebrew/bin/go test -timeout 30s -run ^TestSimpleFactoryModel$ chain_of_duty_model/factory_model

=== RUN   TestSimpleFactoryModel
Using ConcreteProductA
Using ConcreteProductB
--- PASS: TestSimpleFactoryModel (0.00s)
PASS
ok      chain_of_duty_model/factory_model       0.236s
*/

```