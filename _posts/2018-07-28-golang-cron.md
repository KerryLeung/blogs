---
layout:     post                    # 使用的布局（不需要改）
title:      golang cron包源码解析          # 标题 
subtitle:   第三方golang代码库源码阅读 #副标题
date:       2018-07-27              # 时间
author:     KL                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - golang
    - cron
    - open_source
---

# GO Cron库源代码阅读
项目中用到了GO的一个第三方库，内置实现了cron调度的功能，学习一下其设计与实现
项目地址： https://github.com/robfig/cron.git
首先看下官方文档的 https://godoc.org/github.com/robfig/cron 的简单用例

```
c := cron.New()
c.AddFunc("0 30 * * * *", func() { fmt.Println("Every hour on the half hour") })
c.AddFunc("@hourly",      func() { fmt.Println("Every hour") })
c.AddFunc("@every 1h30m", func() { fmt.Println("Every hour thirty") })
c.Start()
..
// Funcs are invoked in their own goroutine, asynchronously.
...
// Funcs may also be added to a running Cron
c.AddFunc("@daily", func() { fmt.Println("Every day") })
..
// Inspect the cron job entries' next and previous run times.
inspect(c.Entries())
..
c.Stop()  // Stop the scheduler (does not stop any jobs already running).
```  
我们首先从`cron.New()`函数开始看

## cron.go 文件
### New()方法
从下面能看出函数New()初始化了一个空的Cron结构体

```
// New returns a new Cron job runner, in the Local time zone.
func New() *Cron {
	return NewWithLocation(time.Now().Location()) // 默认应该UTC timezone
}

// NewWithLocation returns a new Cron job runner.
func NewWithLocation(location *time.Location) *Cron {
	return &Cron{
		entries:  nil,  //entries  []*Entry Entry是cron的核心组件
		add:      make(chan *Entry),
		stop:     make(chan struct{}),
		snapshot: make(chan []*Entry),
		running:  false,
		ErrorLog: nil,
		location: location,
	}
}

// Entry consists of a schedule and the func to execute on that schedule.
type Entry struct {
	// The schedule on which this job should be run.
	Schedule Schedule // 实现了Next方法，可以返回下一次启动cron的时间

	// The next time the job will run. This is the zero time if Cron has not been
	// started or this entry's schedule is unsatisfiable
	Next time.Time

	// The last time this job was run. This is the zero time if the job has never
	// been run.
	Prev time.Time

	// The Job to run.
	Job Job //一个接口实现了Run()，用来存储真正执行的任务是什么
}
```
### AddFunc方法
主要代码如下

```
// AddFunc adds a func to the Cron to be run on the given schedule.
func (c *Cron) AddFunc(spec string, cmd func()) error {
	return c.AddJob(spec, FuncJob(cmd))
}

// AddJob adds a Job to the Cron to be run on the given schedule.
func (c *Cron) AddJob(spec string, cmd Job) error {
	schedule, err := Parse(spec)
	if err != nil {
		return err
	}
	c.Schedule(schedule, cmd)
	return nil
}

// Schedule adds a Job to the Cron to be run on the given schedule.
func (c *Cron) Schedule(schedule Schedule, cmd Job) {
	entry := &Entry{
		Schedule: schedule,
		Job:      cmd,
	}
	if !c.running {
		c.entries = append(c.entries, entry)
		return
	}

	c.add <- entry
}
```
AddFunc第一参数spec是一个string，用来接收cron 时间的定义，从例子上可以看到`"0 30 * * * *"` 类似于这样的，或者`@daily`这样的用法，具体cron package可以支持哪些用法，一会可以看下另外一个文件parser.go。第二个参数cmd是一个函数类型，是说明时间到了之后真正要做的事情。
从上面一大段代码中能看到的AddFunc的做的事情主要是
> 解析cron时间点的定义，把解析之后的结果，放到cron的add chan里面

### Start()方法
cron提供了两种启动方法，一种是带Recovery保护的（runWithRecovery），一种不带。两者都是用go routine新启动了一个进程做cron的调度。
runWithRecovery是在调用cmd函数的时候使用的，可以保证用户自定义函数的panic不会导致cron本身程序的崩溃，是一种保护机制。
然后看下run()真正做了什么。

```
// Start the cron scheduler in its own go-routine, or no-op if already started.
func (c *Cron) Start() {
	if c.running {
		return
	}
	c.running = true
	go c.run()
}

// Run the scheduler. this is private just due to the need to synchronize
// access to the 'running' state variable.
func (c *Cron) run() {
	// Figure out the next activation times for each entry.
	now := c.now()
	for _, entry := range c.entries {
		entry.Next = entry.Schedule.Next(now) // 这里用到了schedule包的Next方法，得到了下次job的启动时间
	}

	for {
		// Determine the next entry to run.
		sort.Sort(byTime(c.entries)) // cron 包定义了自己的排序方法，按照下一次job的启动时间早晚就行排序

		var timer *time.Timer
		if len(c.entries) == 0 || c.entries[0].Next.IsZero() { // 没有任务或者第一个任务没有schedule time，睡眠
			// If there are no entries yet, just sleep - it still handles new entries
			// and stop requests.
			timer = time.NewTimer(100000 * time.Hour)
		} else {
			timer = time.NewTimer(c.entries[0].Next.Sub(now))
		}

		for {
			select {
			case now = <-timer.C: //其实还是使用了time.Timer来触发任务
				now = now.In(c.location)
				// Run every entry whose next time was less than now
				for _, e := range c.entries {
					if e.Next.After(now) || e.Next.IsZero() {
						break
					}
					go c.runWithRecovery(e.Job) // 启动任务函数
					e.Prev = e.Next // 记录下了每次任务调度时间的链路，双链表
					e.Next = e.Schedule.Next(now)
				}

			case newEntry := <-c.add: //AddFunc会在c.add channel添加任务
				timer.Stop()
				now = c.now()
				newEntry.Next = newEntry.Schedule.Next(now) //Schedule.Next才是cron包的主要工作函数，定义了每次任务的触发时间
				c.entries = append(c.entries, newEntry)

			case <-c.snapshot: //可以调用c.Entries()返回一个现有任务列表的snapshot
				c.snapshot <- c.entrySnapshot() 
				continue

			case <-c.stop:
				timer.Stop()
				return
			}

			break
		}
	}
}
```

## paser.go文件
在AddFunc的时候，有一段如下cron string参数的解析过程，可以看下Parse函数的工作原理
`schedule, err := Parse(spec)`
主要函数如下

```
var defaultParser = NewParser(
	Second | Minute | Hour | Dom | Month | DowOptional | Descriptor,
)

func (p Parser) Parse(spec string) (Schedule, error) {
	if spec[0] == '@' && p.options&Descriptor > 0 { //@every 语言描述用法的支持
		return parseDescriptor(spec)
	}
	// Fill in missing fields 
	fields = expandFields(fields, p.options) // 因为支持*，0 之类的cron传统写法，所以需要把* 之类的值填成0

	var err error
	field := func(field string, r bounds) uint64 { // 这种定义函数的用法，之前比较少见，学习了
		...
	}

	var (
		second     = field(fields[0], seconds)
		minute     = field(fields[1], minutes)
		...
	)

	return &SpecSchedule{
		Second: second,
		Minute: minute,
		...
	}, nil
}

// parseDescriptor returns a predefined schedule for the expression, or error if none matches.
func parseDescriptor(descriptor string) (Schedule, error) {
	switch descriptor {
	case "@yearly", "@annually":
		return &SpecSchedule{
			Second: 1 << seconds.min,
			Minute: 1 << minutes.min,
			...
	const every = "@every "
	if strings.HasPrefix(descriptor, every) {
		duration, err := time.ParseDuration(descriptor[len(every):])
}
```
Parser最终返回一个spec.go里面的结构体, schedule 会根据这个结构体的定义判断下一个运行时间是什么时候
```
type SpecSchedule struct {
	Second, Minute, Hour, Dom, Month, Dow uint64
}
```
`parseDescriptor`基本功能就是一个转换器，把语言描述的cron设置信息，转换成真正的schedule，其中every的解析使用了time.ParseDuration()函数，此函数的comment有如下说明，最大支持为h，最小可到ns
`// Valid time units are "ns", "us" (or "µs"), "ms", "s", "m", "h".`
cron调度本身不太需要小于1s的定时调度，所以cron package支持的最小时间间隔也是1s,可看下面代码

```
// Every returns a crontab Schedule that activates once every duration.
// Delays of less than a second are not supported (will round up to 1 second).
// Any fields less than a Second are truncated.
func Every(duration time.Duration) ConstantDelaySchedule {
	if duration < time.Second {
		duration = time.Second
	}
	return ConstantDelaySchedule{
		Delay: duration - time.Duration(duration.Nanoseconds())%time.Second,
	}
}
```
## spec.go文件
最后一个文件，主要功能是产生下一次job运行的时间。代码并不长，主要就是下面的函数

```
   // Next returns the next time this schedule is activated, greater than the given
	// Start at the earliest possible time (the upcoming second).
	t = t.Add(1*time.Second - time.Duration(t.Nanosecond())*time.Nanosecond)

	// This flag indicates whether a field has been incremented.
	added := false

	// If no time is found within five years, return zero.
	yearLimit := t.Year() + 5

WRAP:
	if t.Year() > yearLimit {
		return time.Time{}
	}

	// Find the first applicable month.
	// If it's this month, then do nothing.
	for 1<<uint(t.Month())&s.Month == 0 { // 0 表示每个月都会运行
		// If we have to add a month, reset the other parts to 0.
		if !added {
			added = true
			// Otherwise, set the date at the beginning (since the current time is irrelevant).
			t = time.Date(t.Year(), t.Month(), 1, 0, 0, 0, 0, t.Location())
		}
		t = t.AddDate(0, 1, 0)

		// Wrapped around.
		if t.Month() == time.January {
			goto WRAP
		}
	}

	// Now get a day in that month.
	for !dayMatches(s, t) {
		if !added {
			added = true
			t = time.Date(t.Year(), t.Month(), t.Day(), 0, 0, 0, 0, t.Location())
		}
		t = t.AddDate(0, 0, 1)

		if t.Day() == 1 {
			goto WRAP
		}
	}
	...

	return t
}
```
这里函数用到我之前项目组从没用过的goto用法，在golang的内存分配，go routinue管理的代码中可看到不少类似的用法，业务代码中不太常见。 从comment中能看到如下描述
> While incrementing the field, a wrap-around brings it back to the beginning of the field list (since it is necessary to re-verify previous field values)

上面代码中有一个地方需要注意，cron有个validation去判断，如果你定义的cron任务5年之后才会第一次运行，程序会认为是非法的。正常情况下也不会真有人这么用😄

## 总结
cron packge代码结构拆分比较合理，代码阅读起来也不费事，调用流程也很容易可以走通。闲暇时间，review学习下他人写的代码也是对自己的一种提高。