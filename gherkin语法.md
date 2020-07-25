# 关键字

* `Feature`
* `Rule`
    * `Example` or `Scenario`
* `Given, When, Then, And, But`(steps)
* `Background`
* `Scenario Outline` (or `Scenario Template`)
* `Examples`

也有一些次要的关键字:

- `"""` (Doc Strings)
- `|` (Data Tables)
- `@`(Tags)
- `#`(Comments)

# 释义

## `Feature`

提供软件特性的高级描述，并对相关场景进行分组。Gherkin文档中的第一个关键字必须始终是`Feature`，然后是`:`和描述该特性的简短文本。您可以在`Feature`下面添加自由格式的文本来添加更多的描述。这些描述行在运行时被`Cucumber`忽略，但是可以用于报告(它们默认包含在html报告中)。

## Step

每个step都以关键字 `Given`, `When`, `Then`, `And`或 `But` 开头。

Cucumber按您编写的顺序执行场景中的每个步骤，每次执行一个。当Cucumber尝试执行一个步骤时，它会寻找一个匹配的步骤定义来执行。在寻找步骤定义时，没有考虑关键字。这意味着您不能使用与另一个步骤相同的文本执行`Given`, `When`, `Then`, `And`或 `But`步骤。

即，在忽略关键字后，如果两个文本相同，则 `Cucumber` 认为重复了。下面的两行是重复的！

```gherkin
Given there is money in my account
Then there is money in my account
```

### Given

`Given` 步骤用于描述系统的初始上下文—场景。这通常发生在过去。`Given` 步骤的目的是在用户(或外部系统)开始与系统交互之前(在 `When` 步骤中)将系统置于已知状态。避免讨论用户在`Given`步骤中交互。

如果您正在创建用例，`Given` 将是您的先决条件。有几个`Given` 步骤是可以的(对于2个及以上的步骤，只需使用`And`或`But`使其更具可读性)。

多个 `Given` 可以认为是一个数组，`Cucumber`读取的时候，会依次进行初始化。

```gherkin
    Given variable "blockchain_r" is assigned the value "TOKEN_BLOCKCHAIN_R"
    And variable "cas_r" is assigned the value "TOKEN_CAS_R"
    And variable "did_w" is assigned the value "TOKEN_DID_W"
```

上面例子等价于：

```gherkin
    Given variable "blockchain_r" is assigned the value "TOKEN_BLOCKCHAIN_R"
    Given variable "cas_r" is assigned the value "TOKEN_CAS_R"
    Given variable "did_w" is assigned the value "TOKEN_DID_W"
```

如果有下面的函数：

```go
func (d *CommonSteps) setVariable(varName, expr string) error {
	value, err := ResolveVarsInExpression(expr)
	if err != nil {
		return err
	}

	logger.Infof("Setting var [%s] to [%s]", varName, value)

	SetVar(varName, value)

	return nil
}
```

则在解析上面的`feature`文件时，通过打断点可以发现，该函数被连着调用三次，分别初始化`blockchain_r`、`cas_r`和`did_w`。

### when

`When` 步骤用于描述事件或动作。这可以是与系统交互的人，也可以是由另一个系统触发的事件

强烈建议每个场景只有一个`When`步骤。如果你觉得有必要添加更多，这通常是一个信号，表明你应该将场景分成多个场景。

### then

`Then` 步骤用于描述预期的结果

`Then` 步骤应该使用一个断言来比较实际的结果(系统实际做了什么)和预期的结果(这个步骤说明了系统应该做什么)

### And, But

如果你有几个`Given`’s, `When`’s, 或 `Then`s，你可以这样写:

```gherkin
Example: Multiple Givens
  Given one thing
  Given another thing
  Given yet another thing
  When I open my eyes
  Then I should see something
  Then I shouldn't see something else
```

或者,你可以让它阅读更流畅:



```gherkin
Example: Multiple Givens
  Given one thing
  And another thing
  And yet another thing
  When I open my eyes
  Then I should see something
  But I shouldn't see something else
```

## Background

有时候，您会发现自己在某个特性的所有场景中重复相同的`Given`步骤。由于它在每个场景中都是重复的，这表明这些步骤对于描述场景并不重要;这些都是偶然的细节。您可以通过将这些`Given`步骤分组到一个背景部分中，将它们移动到`Background`中。

`Background`允许您向特性中的场景添加一些上下文。它可以包含一个或多个`Given`步骤。

`Background`在每个场景之前运行，并在任何Before hooks之后运行。在feature 文件中，将`Background`放在第一个`Scenario`之前。

每个特性只能有一组`Background`步骤。如果您需要针对不同的场景使用不同的`Background`步骤，则需要将它们拆分为不同的功能文件。

# @（Tags）

一个.feature 文件里面可以有很多scenario组成。如果我们运行了一个包含有很多个scenario的feature文件时，它会执行这个文件里面所有的scenario；但是有的时候我们可能只想运行某一个/些特别的scenario时，这时我们可以使用Tags; 

可以在Feature或Scenario关键字前给feature或scenario添加任意数量的tags，如： 

```gherkin
　　@approved @book_flight
　　Feature: Book flight 
        　　@wip
        　　Scenario: Book a flight on web
```

一个Scenario会继承指定给Feature的tags，所以在上面的例子中，Scenario有三个tags：@approved @book_flight @wip. 然后我们就可以使用命令：cucumber --tags tag_name来运行我们想运行的那部分Scenario.如：cucumber –tags @wip 

可以将 `Tags` 传递给函数，以便让程序根据所选择的 `Tags` 运行不同的Scenario部分！

如

```go
func main(){
  tags := "all"

	flag.Parse()
	cmdTags := flag.CommandLine.Lookup("test.run")
	if cmdTags != nil && cmdTags.Value != nil && cmdTags.Value.String() != "" {
		tags = cmdTags.Value.String()
	}
  status := godog.RunWithOptions("godogs",func(s *godog.Suite){
    s.BeforeSuite(func(){
      
    })
  },godog.Options{
    Tags: tags,
  })
}
```

此时可以根据 `--test.run tagsName` 来运行想要运行的部分



# 参考：

1. [Cucumber入门之Gherkin](https://www.cnblogs.com/puresoul/archive/2011/12/28/2305160.html)
2. [零基础实现BDD自动化测试](http://cuketest.com/zh-cn/)
3. [github.com/cucumber/gherkin](https://github.com/cucumber/cucumber/tree/master/gherkin)
4. [Gherkin(小黄瓜) 参考文档](https://www.jianshu.com/p/43cb0e79f075)
5. [Gherkin Reference](https://cucumber.io/docs/gherkin/reference/)

