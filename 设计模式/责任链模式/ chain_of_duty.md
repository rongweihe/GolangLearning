## 责任链模式
责任链模式是一种行为设计模式，允许你将请求沿着处理者链进行发送。收到请求后，每个处理者均可对请求进行处理，或将其传递给链上的下个处理者。

## 概念
责任链模式是一种行为设计模式， 允许你将请求沿着处理者链进行发送。收到请求后，每个处理者均可对请求进行处理，或将其传递给链上的下个处理者。

该模式允许多个对象来对请求进行处理，而无需让发送者类与具体接收者类相耦合。链可在运行时由遵循标准处理者接口的任意处理者动态生成。

一般意义上的责任链模式是说，请求在链上流转时任何一个满足条件的节点处理完请求后就会停止流转并返回，不过还可以根据不同的业务情况做一些改进：

> 请求可以流经处理链的所有节点，不同节点会对请求做不同职责的处理；
> 可以通过上下文参数保存请求对象及上游节点的处理结果，供下游节点依赖，并进一步处理；处理链可支持节点的异步处理，通过实现特定接口判断，是否需要异步处理；

## 实现
责任链对于请求处理节点可以设置停止标志位，不是异常，是一种满足业务流转的中断；

- 责任链的拼接方式存在两种，一种是节点遍历，一个节点一个节点顺序执行；另一种是节点嵌套，内层节点嵌入在外层节点执行逻辑中，类似递归，或者“回”行结构；
- 责任链的节点嵌套拼接方式多被称为拦截器链或者过滤器链，更易于实现业务流程的切面，比如监控业务执行时长，日志输出，权限校验等；


## 
示例

本示例模拟实现机场登机过程

- 第一步办理登机牌，
- 第二步如果有行李，就办理托运，
- 第三步核实身份，
- 第四步安全检查，
- 第五步完成登机；
- 其中行李托运是可选的，其他步骤必选，必选步骤有任何不满足就终止登机；
- 旅客对象作为请求参数上下文，每个步骤会根据旅客对象状态判断是否处理或流转下一个节点；

```go
package chain_of_duty_model

import "fmt"

// BoardingProcessor 登机过程中，各节点统一处理接口
type BoardingProcessor interface {
	SetNextPro(processor BoardingProcessor)
	ExecutePro(pass *Passenger)
}

// Passenger 旅客
type Passenger struct {
	name                  string // 姓名
	hasBoardingPass       bool   // 是否办理登机牌
	hasLuggage            bool   // 是否有行李需要托运
	isPassIdentityCheck   bool   // 是否通过身份校验
	isPassSecurityCheck   bool   // 是否通过安检
	isCompleteForBoarding bool   // 是否完成登机
}

// baseBoardingProcessor 登机流程处理器基类
type baseBoardingProcessor struct {
	// nextProcessor 下一个登机处理流程
	nextProcessor BoardingProcessor
}

// SetNextPro 基类中统一实现设置下一个处理器方法
func (b *baseBoardingProcessor) SetNextPro(processor BoardingProcessor) {
	b.nextProcessor = processor
}

// ExecutePro 基类中统一实现下一个处理器流转
func (b *baseBoardingProcessor) ExecutePro(pass *Passenger) {
	if b.nextProcessor != nil {
		b.nextProcessor.ExecutePro(pass)
	}
}

// boardingPassProcessor 办理登机牌处理器
type boardingPassProcessor struct {
	baseBoardingProcessor // 引用基类
}

func (b *boardingPassProcessor) ExecutePro(pass *Passenger) {
	if !pass.hasBoardingPass {
		fmt.Printf("为旅客%s办理登机牌;\n", pass.name)
		pass.hasBoardingPass = true
	}
	// 成功办理登机牌后，进入下一个流程处理
	b.baseBoardingProcessor.ExecutePro(pass)
}

// luggageCheckInProcessor 托运行李处理器
type luggageCheckInProcessor struct {
	baseBoardingProcessor
}

func (l *luggageCheckInProcessor) ExecutePro(pass *Passenger) {
	if !pass.hasBoardingPass {
		fmt.Printf("旅客%s未办理登机牌，不能托运行李;\n", pass.name)
		return
	}
	if pass.hasLuggage {
		fmt.Printf("为旅客%s办理行李托运;\n", pass.name)
	}
	l.baseBoardingProcessor.ExecutePro(pass)
}

// identityCheckProcessor 校验身份处理器
type identityCheckProcessor struct {
	baseBoardingProcessor
}

func (i *identityCheckProcessor) ExecutePro(pass *Passenger) {
	if !pass.hasBoardingPass {
		fmt.Printf("旅客%s未办理登机牌，不能办理身份校验;\n", pass.name)
		return
	}
	if !pass.isPassIdentityCheck {
		fmt.Printf("为旅客%s核实身份信息;\n", pass.name)
		pass.isPassIdentityCheck = true
	}
	i.baseBoardingProcessor.ExecutePro(pass)
}

// securityCheckProcessor 安检处理器
type securityCheckProcessor struct {
	baseBoardingProcessor
}

func (s *securityCheckProcessor) ExecutePro(pass *Passenger) {
	if !pass.hasBoardingPass {
		fmt.Printf("旅客%s未办理登机牌，不能进行安检;\n", pass.name)
		return
	}
	if !pass.isPassSecurityCheck {
		fmt.Printf("为旅客%s进行安检;\n", pass.name)
		pass.isPassSecurityCheck = true
	}
	s.baseBoardingProcessor.ExecutePro(pass)
}

// completeBoardingProcessor 完成登机处理器
type completeBoardingProcessor struct {
	baseBoardingProcessor
}

func (c *completeBoardingProcessor) ExecutePro(pass *Passenger) {
	if !pass.hasBoardingPass ||
		!pass.isPassIdentityCheck ||
		!pass.isPassSecurityCheck {
		fmt.Printf("旅客%s登机检查过程未完成，不能登机;\n", pass.name)
		return
	}
	pass.isCompleteForBoarding = true
	fmt.Printf("旅客%s成功登机;\n", pass.name)
}
```
## 单侧

```go
package chain_of_duty_model

import (
	"fmt"
	"testing"
)

func TestChainOfDutyModel(t *testing.T) {
	boardingProcessor := BuildBoardingProcessorChain()
	passenger := &Passenger{
		name:                  "李四",
		hasBoardingPass:       false,
		hasLuggage:            true,
		isPassIdentityCheck:   false,
		isPassSecurityCheck:   false,
		isCompleteForBoarding: false,
	}
	fmt.Printf("乘客%s开始登机流程11\n", passenger.name)
	boardingProcessor.ExecutePro(passenger)
	fmt.Printf("乘客%s登机流程结束22\n", passenger.name)
}

// BuildBoardingProcessorChain 构建登机流程处理链
func BuildBoardingProcessorChain() BoardingProcessor {
	completeBoardingNode := &completeBoardingProcessor{}

	//完成登机
	securityCheckNode := &securityCheckProcessor{}
	securityCheckNode.SetNextPro(completeBoardingNode)

	//身份校验
	identityCheckNode := &identityCheckProcessor{}
	//之后安检
	identityCheckNode.SetNextPro(securityCheckNode)

	//托运行李
	luggageCheckInNode := &luggageCheckInProcessor{}
	//之后身份校验
	luggageCheckInNode.SetNextPro(identityCheckNode)

	//办理登机牌
	boardingPassNode := &boardingPassProcessor{}
	//之后托运行李
	boardingPassNode.SetNextPro(luggageCheckInNode)
	return boardingPassNode
}

/*

Running tool: /opt/homebrew/bin/go test -timeout 30s -run ^TestChainOfDutyModel$ chain_of_duty_model/chain_of_duty_model

=== RUN   TestChainOfDutyModel
乘客李四开始登机流程111
为旅客李四办理登机牌;
为旅客李四办理行李托运;
为旅客李四核实身份信息;
为旅客李四进行安检;
旅客李四成功登机;
乘客李四登机流程结束22
--- PASS: TestChainOfDutyModel (0.00s)
PASS
*/

```