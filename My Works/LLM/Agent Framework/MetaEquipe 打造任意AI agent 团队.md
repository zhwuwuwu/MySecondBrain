## 1. Target
#### Background
multi-agent框架在不断完善，我们有opencode/openclaw，实现了很多好的feature [e.g. ]。
现有很多有multiple AI agent组成的ai agent团队甚至一人公司 [e.g. Oh-My-Opencode，社媒，电商等]。
但是每个完成特定任务的multi-agent系统都需要case by case手工打造，手段通常是基于framework，通过编程的方式逐个定制各个角色和交互方式，补充一些特定的角色解决特定任务。
#### Goal
希望能够实现一个multi-agent 系统，具备父级能力，类似于工业母机，作为multi-agent 团队创建者，可以帮我们自动化打造任意agent team。
他本身也是一个写代码的multi-agent system， 不过他的任务就是创造multi-agent system，相当于“元能力”，所以称它为MetaEquipe。
#### Examples of multi-agent system

1. 以内容生产为核心角色的个人MCN公司为例
	1. 内容生成相关 agent （生产公开内容-cloud）
	2. 公司行政 agent （涉及本公司商业数据-local）
	3. 会计agent （涉及本公司商业数据-local）
	4. 运营/商务agent（涉及本公司商业数据-local）
	
	[一个人，16 个 AI 员工，13 个自媒体平台，每天 10 分钟搞定。_openclaw 多agent员工-CSDN博客](https://blog.csdn.net/2202_75716091/article/details/158316802)
	
	16 个 Agent 分成了三层：
	
	第一层：7 个基础 Agent（后台支撑）
	
	小墨（总助）:日常协调、日程提醒、任务分发
	墨笔（内容）:写稿机器，所有平台的内容创作都经过它
	墨影（设计）:配图、封面、视觉素材
	墨码（开发）:技术支持，修 bug、写脚本、搭工具
	墨风（增长）:SEO 数据、关键词排名、流量分析
	墨账（财务）:支出汇总、发票管理、收入统计
	墨盾（安全）:合规检查、合同审查、风控
	第二层：1 个运营总监
	
	墨媒（运营总监）:统筹调度，每天选题方向、数据汇总、异常上报、周报月报
	第三层：8 个平台 Agent（各自独立闭环）
	
	墨微（公众号）:长文创作、排版、发布
	墨知（知乎）:问答、专栏、热榜追踪
	墨红（小红书）:图文笔记、话题标签
	墨推（X/Twitter）:英文推文、互动策略
	墨拍（视频号+抖音）:短视频脚本、发布
	墨星（掘金+知识星球）:技术文章、社群运营
	墨播（B 站+YouTube）:视频内容、SEO 优化
	墨圈（微博+即刻）:短内容、热点跟进

2. 电商一人公司
	1. 广告agent （cloud）
	2. 客服agent（cloud）
	3. 库存agent （local）
	4. 数据日报agent （local）
	
3. 软开流程team
	1. Auto test agent (只跑测试，不接触实际代码，可以用cloud model)
	2. Issue report agent
	3. triaging and analysis agent
	4. debug agent （debug需要看到真实代码，local model）
	5. back to test 

4. Oh-My-Opencode team

## 2. Implementation
### 2.1 打造思路
迭代形式 ：
	从手动打造原型开始，使用具备下述能力的多agent系统，实现更强一级能力的下述系统

确定有TEAM需要哪些feature
- 初始的feature下边列出来
- 请opencode介绍自己的各种能力和优秀feature 作为补充
用现有的opencode作为一个基础的team builder V1.0 
- 让他在现有实现 (oh my opencode 插件) 基础上构建一个opencode TEAM builder 插件
### 2.2 算法伪代码 
```
<init>
opencode + oh-my-opencode -> a coding agent team
Clarify the features that a team need (manual def + OMO features)
++ instructions -> MetaEquipe V0

<while true>
	Use MetaEquipe Vi -genarate-> Vi+1
	if 收敛 [TODO]：
		Stop
<end>
```


### 2.3 Requirements [TODO]
一个完善的multi-agent system应该具备这些能力：
#### 任务解决能力
通过递归multi-agent来解决问题

- 并行任务
	- subagent
- 重复任务
	- cron
- 复杂任务
	- General-task Ralph Loop: Cron supervisor + Execution Hook
	- tmux
- 其他普通任务
#### Safety guardrail 
- 执行前权限审核
- 输入内容审核
#### Local memory/SQL: SQlite/Notion/Obsidial
通过session management实现下面的功能 (利用框架现有的能力)
- 用户历史问题
- 判断最终任务
- 已完成任务
- 未完成任务
- 当前上下文
	- 文件
	- 外部信息
	- 当前任务执行状态
	- 当前任务依赖
- 任务约束条件
- 每个agent状态


## 3. 其他更难解决的问题
### Memory Bottleneck