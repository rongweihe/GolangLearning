# 适配器模式

适配器模式是一种结构型设计模式，它允许将一个类的接口转换成客户希望的另一个接口。**适配器模式使得原本由于接口不兼容而不能一起工作的类可以一起工作。**

要点描述
- 一般来说，适配器模式可以看做一种“补偿模式”，用来补救设计上的缺陷。

- 应用这种模式算是“无奈之举”。如果在设计初期，我们就能规避协调接口不兼容的问题，那这种模式就没有应用机会了。

- 适配器模式主要应用于“希望复用一些现存的类，但是接口又与复用环境要求不一致的情况”，在遗留代码复用、类库迁移等方面非常有用。

## 问题描述
通过充电宝给不同充电接口的手机充电是一个非常符合适配器模式特征的生活示例；

- 一般充电宝提供 USB 电源输出接口，手机充电输入接口则分为两类一是苹果手机的lightning 接口，另一类是安卓手机的 typeC 接口

## 解决方案 
- 这两类接口都需要通过适配电源线连接充电宝的 USB 接口，这里 USB 接口就相当于充电宝的通用接口，lightning 或 typeC 接口要想充电需要通过充电线适配。

## 示例代码
```go
package adapter_model

import "fmt"

// 手机充电插头

type HuaWeiPlug interface {
	ConnectTypeC() string
}
type HuaWeiPhone struct {
	model string
}

func NewHuaWeiPhone(model string) *HuaWeiPhone {
	return &HuaWeiPhone{
		model: model,
	}
}

func (h *HuaWeiPhone) ConnectTypeC() string {
	return fmt.Sprintf("华为手机%s通过typeC接口连接充电宝", h.model)
}

type ApplePlug interface {
	ConnectLightning() string
}
type ApplePhone struct {
	model string
}

func NewApplePhone(model string) *ApplePhone {
	return &ApplePhone{
		model: model,
	}
}
func (a *ApplePhone) ConnectLightning() string {
	return fmt.Sprintf("苹果手机%s通过lightning接口连接充电宝", a.model)
}

// 充电宝适配器

// CommonPlug 通用的USB电源插槽
type CommonPlug interface {
	ConnectUSB() string
}

// HuaweiPhonePlugAdapter 华为TypeC充电插槽适配通用USB充电插槽
type HuaweiPhonePlugAdapter struct {
	huaweiPhone HuaWeiPlug
}

// NewHuaweiPhonePlugAdapter 创建华为手机适配USB充电插槽适配器
func NewHuaweiPhonePlugAdapter(huaweiPhone HuaWeiPlug) *HuaweiPhonePlugAdapter {
	return &HuaweiPhonePlugAdapter{
		huaweiPhone: huaweiPhone,
	}
}

// ConnectUSB 链接USB
func (h *HuaweiPhonePlugAdapter) ConnectUSB() string {
	return fmt.Sprintf("%v adapt to usb ", h.huaweiPhone.ConnectTypeC())
}

// ApplePhonePlugAdapter 苹果Lightning充电插槽适配通用USB充电插槽
type ApplePhonePlugAdapter struct {
	iPhone ApplePlug
}

// NewApplePhonePlugAdapter 创建苹果手机适配USB充电插槽适配器
func NewApplePhonePlugAdapter(iPhone ApplePlug) *ApplePhonePlugAdapter {
	return &ApplePhonePlugAdapter{
		iPhone: iPhone,
	}
}

// ConnectUSB 链接USB
func (a *ApplePhonePlugAdapter) ConnectUSB() string {
	return fmt.Sprintf("%v adapt to usb ", a.iPhone.ConnectLightning())
}

// PowerBank 充电宝
type PowerBank struct {
	brand string
}

// Charge 支持通用USB接口充电
func (p *PowerBank) Charge(plug CommonPlug) string {
	return fmt.Sprintf("%v power bank connect usb plug, start charge for %v", p.brand, plug.ConnectUSB())
}

```

## 单侧

```go
package adapter_model

import (
	"fmt"
	"testing"
)

func TestAdapter(t *testing.T) {
	huaweiMate40Pro := NewHuaWeiPhone("华为 P70 pro")
	iphone13MaxPro := NewApplePhone("苹果 iphone16 pro max")

	powerBank := &PowerBank{"飞利浦"}
	fmt.Println(powerBank.Charge(NewHuaweiPhonePlugAdapter(huaweiMate40Pro)))
	fmt.Println(powerBank.Charge(NewApplePhonePlugAdapter(iphone13MaxPro)))
}

/*
Running tool: /opt/homebrew/bin/go test -timeout 30s -run ^TestAdapter$ chain_of_duty_model/adapter_model

=== RUN   TestAdapter
飞利浦 power bank connect usb plug, start charge for 华为手机华为 mate40 pro通过typeC接口连接充电宝 adapt to usb
飞利浦 power bank connect usb plug, start charge for 苹果手机苹果 iphone13 pro max通过lightning接口连接充电宝 adapt to usb
--- PASS: TestAdapter (0.00s)
PASS
ok      chain_of_duty_model/adapter_model       0.242s

*/

```