---
title: "设计模式-工厂模式"
date: 2021-06-18T15:30:15+08:00
draft: false
---

工厂模式可以细分为`简单工厂模式`、`工厂方法模式`以及`抽象工厂模式` 。

## 简单工厂

当对象的创建逻辑很简单，不需要太多的初始化操作时，可以选择简单工厂模式。

Go 语言本身没有构造函数，所以实现简单工厂就可以直接使用 `NewXXX` 这种方式来实现，并且返回一个接口：

```go
package factory

type IRuleConfigParser interface {
	Parse(data []byte)
}

type JsonRuleConfigParser struct {
}

func (j JsonRuleConfigParser) Parse(data []byte) {
	panic("implement me")
}

type XmlRuleConfigParser struct {
}

func (j XmlRuleConfigParser) Parse(data []byte) {
	panic("implement me")
}

func NewIRuleConfigParser(parserType string) IRuleConfigParser {
	switch parserType {
	case "json":
		return JsonRuleConfigParser{}
	case "xml":
		return XmlRuleConfigParser{}
	default:
		return nil
	}
}
```

## 工厂方法

当对象的创建逻辑比较复杂，不只是简单的 `new` 一下就可以，而是要组合其他类对象，做各种初始化操作的时候，推荐使用工厂方法模式，将复杂的创建逻辑拆分到多个工厂类中，每个工厂类都负责一种类型对象的创建：

```go
package factory

type IRuleConfigParserFactory interface {
	CreateRuleConfigParser() IRuleConfigParser
}

type JsonRuleConfigParserFactory struct {

}

func (j JsonRuleConfigParserFactory) CreateRuleConfigParser() IRuleConfigParser {
	// doSomething
	return JsonRuleConfigParser{}
}

type XmlRuleConfigParserFactory struct {

}

func (j XmlRuleConfigParserFactory) CreateRuleConfigParser() IRuleConfigParser {
	// doSomething
	return XmlRuleConfigParser{}
}

// NewIRuleConfigParserFactory 用一个简单工厂封装工厂方法
func NewIRuleConfigParserFactory(t string) IRuleConfigParserFactory {
	switch t {
	case "json":
		return JsonRuleConfigParserFactory{}
	case "xml":
		return XmlRuleConfigParserFactory{}
	}
	return nil
}
```

工厂方法模式比起简单工厂模式更加符合开闭原则。

## 抽象工厂

抽象工厂让一个工厂负责创建多个不同类型的对象（IRuleConfigParser、ISystemConfigParser 等），而不是只创建一种 parser 对象。

该模式用得比较少，故不做讨论。
