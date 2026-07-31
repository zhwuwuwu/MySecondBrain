
### 1. 商业模式
#### summary of 129 openclaw-related startups 
1. 部署能力
	1. 云端一键托管
	2. 本地硬件部署 
		1. SetupClaw [TODO] 他们的客户是由隐私执念的律师、会计群体，可以进一步看看他的usecase
	3. 移动APP封装

2. 一人公司定制: ++UI, template, preset agent role, domain specific
	1. 打造自己的一人公司
	2. 卖agent group模板
	3. 卖垂类AI员工： design + 部署，收取服务费+维护费
		e.g. 行政助手、销售团队, 
		e.g. clawmart 卖调好的agent，
		![[Pasted image 20260304054706.png]]
	
3. 基础设施
	1. token monitor
	2. multi-claw dashboard
	3. routers: 
		1. zero rulers拦截低价值调用
		2. clawtunnel
		3. taskcontrol
	4. Skills
		1. skills市场：Clawmart/水产市场（openclawmp.cc)
		2. Keepsake，个人记忆系统

#### reference：
- [第一批拿OpenClaw赚钱的人：有的月入30万，有的卖虾给屋顶修理工_腾讯新闻](https://news.qq.com/rain/a/20260227A04SA700)
- 	other wechat articles
	
### 2. 应用场景

**reference:**

- 98 实战案例[TODO]：	[如何让 AI 替你干活？OpenClaw 98 个实战案例全公开 - 今日头条](https://www.toutiao.com/article/7607086308216914438/?wid=1772613232108)
- Awesome Openclaw：[hesamsheikh/awesome-openclaw-usecases: A community collection of OpenClaw use cases for making life easier.](https://github.com/hesamsheikh/awesome-openclaw-usecases) 
- Awesome Openclaw-ZH：[cogine-ai/awesome-openclaw-zh: 中文 OpenClaw 实战案例库：168 个可复制使用场景，从5分钟上手到完全精通。案例覆盖自动化、内容创作、运营增长与低成本稳定运行。](https://github.com/cogine-ai/awesome-openclaw-zh) 

#### 2.1. 内容自动化：
Openclaw作为自动化工具, Skill 作为API对接，其中AI作为内容理解和编码工具
1. 内容生成/分析：每日简报/商业分析/竞品调研/数据监控/行政助手
2. 社媒发帖 - RSS blog/X/XHS skill
	1. 热点创作
	2. 转发
	3. 总结切片
	4. 品牌内容
	5. 内容推荐
	6. 内容爬虫（进一步分析）
	- **RISK：** 大部分社媒官方是反自动化的，爬虫；XHS没有官方发帖API 
3. 剪辑视频 - 剪映skill
	1. [自从用OpenClaw剪短影片｜我想退订「剪映」了_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1WpfSBoEeP/?spm_id_from=333.337.search-card.all.click) 
	2. [【AI全自动接管剪映V6】openClaw接管剪映全自动剪辑，剪映skill安装不要太方便_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1oeAhz3Edd/) 
4. 3D打印 - Bambu Skill
	
#### 2.2. 开发运维：
Openclaw具备24h自动化能力，AI写代码
1. Code Review
2. Build status
3. Issue check （类似于邮件check）
4. 自动推送/部署
5. CI-CD全流程
6. 流程图/图表

#### 2.3. ==个人助手/公司业务助手==
有隐私性，和pipeline强绑定，用户粘性高，壁垒不是产品，是位置
1. 日常助手
	1. 邮件分拣-摘要-回复草稿
	2. 日历管理
	3. 购物助手
2. 投资助手
	1. 盯盘
	2. 预测
	3. 24h交易
3. 个人记忆系统/外挂大脑
4. 家庭助手（IOT）
5. 业务运营
	1. 初创公司外部邮件：判断潜在客户/客户分级
	2. 会议记录、分析、转化
	3. 发票管理（属于会计相关应用）
	4. 商业分析/竞品分析
	5. 电商/内容/广告数据分析



#### 2.4. 行业垂直应用：[TODO: 需要较深垂类知识，需要的话可以进一步看]

**1. 生物医药行业**
- **文章**："OpenClaw 降临：生物医药企业如何用行动智能体重塑研发与商业化版图？"（MedicalAI）
- **方案**：对接企业医学知识库，确保 AI 输出合规
- **场景**：药物研发信息整理、文献检索、临床数据分析
- 化验结果分析
- 个人健康数据/运动数据分析与可视化

**2. 金融投研**
- **来源**：awesome-openclaw-usecases 中的 "AI Earnings Tracker"、"Polymarket Autopilot"
- **中文报道**："OpenClaw在金融投研领域的实际应用研究"（江北哈尔滨新区）
- **场景**：财报追踪、预测市场分析、Daily Digest

**3. 军事场景设想**
- **来源**："智鹰科技" 公众号
- **内容**：偏理论探讨，分析 OpenClaw 架构在军事情报整合、态势感知中的潜在应用

**4. 内容创作/自媒体**
- **B站案例**："自从用OpenClaw剪短影片，我想退订剪映了"（院长G大，6.5万播放）
- **方案**：多 Agent 内容工厂（研究、写作、配图 Agent 协作）
- **来源**：awesome-openclaw-usecases "Multi-Agent Content Factory"

### 3. Summary & Thinking：

#### AI level
1. AI核心作用：内容理解和生成/编码
2. 如果内容是AI生成的，也是AI看的，那么内容的形态就应该改变了。
	1. AI content for AI
	2. 人在里边扮演了什么角色？有些内容不需要人来看了，例如安装文档/readme/tuto？知识类的信息对人还有用吗？艺术、娱乐类的信息还是对人有用的。

#### Hybrid level
1. Hybrid 级别 - cloud + local skill是不是也算是hybrid
		1. 现在对安全的担心造成了物理隔离：是用一台mac-mini单独装openclaw
		2. 人们的担心是怕AI乱操作，还没到区分local/remote AI
2. **以AI代替浏览器为例，安全不敏感的云端的AI代替不敏感的互联网应用；那Hybrid AI应该代替一些传统本地应用** 
	1. 个人助手
	2. 商业数据： 图纸，专利，商业数据
3. ==Hybrid的主要优势是安全，在一些居家物联网场景下有潜力==
4. ==一人公司场景很适合Hybrid==
5. [OpenClaw 让 Mac Mini 卖爆了，苹果为什么不自己做一个？-36氪](https://www.36kr.com/p/3675978149716865) 

#### Openclaw Level
1. 目前大部分应用关注在openclaw本身而不是把他当工具，即使是做内容也是生成openclaw的内容
2. openclaw的主要优势是24h不间断+自动化，目前核心应用是自动化助手
3. openclaw本身没有不可替代性，轻量化claw，openwork, pi LLM SDK
4. 对于复杂规划，长时间执行的单一复杂任务，openclaw不如 ralph-loop + opencode/claudecode， 需要一个执行通用任务的RalphLoop
	1. [(9 封私信 / 34 条消息) OpenClaw 搭建了一人开发团队：从每天 50 次提交到一人百万美元公司 - 知乎](https://zhuanlan.zhihu.com/p/2009921770770175271) Obsidian + tmux + cron
5. 可以作为协调者控制多agent/claude并行，但是可以有更高效的方案


### 4. 一人公司 USE CASE 
#### Implementation Requirements
	multi-agent
	authority guardrail
	Local memory/SQL: SQlite/Notion/Obsidial
	session management
	General-task Ralph Loop: Cron supervisor + Execution Hook
	Memory Bottleneck
#### Examples 

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
### RISKS：
1. [工信部NVDB提示：防范OpenClaw开源AI智能体安全风险_腾讯新闻](https://news.qq.com/rain/a/20260205A05KLP00#:~:text=%E7%94%B1%E4%BA%8EOpenClaw%E5%9C%A8%E9%83%A8%E7%BD%B2%E6%97%B6%E2%80%9C%20%E4%BF%A1%E4%BB%BB%E8%BE%B9%E7%95%8C%E6%A8%A1%E7%B3%8A,%E2%80%9D%EF%BC%8C%E4%B8%94%E5%85%B7%E5%A4%87%E8%87%AA%E8%BA%AB%E6%8C%81%E7%BB%AD%E8%BF%90%E8%A1%8C%E3%80%81%E8%87%AA%E4%B8%BB%E5%86%B3%E7%AD%96%E3%80%81%E8%B0%83%E7%94%A8%E7%B3%BB%E7%BB%9F%E5%92%8C%E5%A4%96%E9%83%A8%E8%B5%84%E6%BA%90%E7%AD%89%E7%89%B9%E6%80%A7%EF%BC%8C%E5%9C%A8%E7%BC%BA%E4%B9%8F%E6%9C%89%E6%95%88%E6%9D%83%E9%99%90%E6%8E%A7%E5%88%B6%E3%80%81%E5%AE%A1%E8%AE%A1%E6%9C%BA%E5%88%B6%E5%92%8C%E5%AE%89%E5%85%A8%E5%8A%A0%E5%9B%BA%E7%9A%84%E6%83%85%E5%86%B5%E4%B8%8B%EF%BC%8C%E5%8F%AF%E8%83%BD%E5%9B%A0%E6%8C%87%E4%BB%A4%E8%AF%B1%E5%AF%BC%E3%80%81%E9%85%8D%E7%BD%AE%E7%BC%BA%E9%99%B7%E6%88%96%E8%A2%AB%E6%81%B6%E6%84%8F%E6%8E%A5%E7%AE%A1%EF%BC%8C%E6%89%A7%E8%A1%8C%E8%B6%8A%E6%9D%83%E6%93%8D%E4%BD%9C%EF%BC%8C%E9%80%A0%E6%88%90%E4%BF%A1%E6%81%AF%E6%B3%84%E9%9C%B2%E3%80%81%E7%B3%BB%E7%BB%9F%E5%8F%97%E6%8E%A7%E7%AD%89%E4%B8%80%E7%B3%BB%E5%88%97%E5%AE%89%E5%85%A8%E9%A3%8E%E9%99%A9%E3%80%82%20%E5%BB%BA%E8%AE%AE%E7%9B%B8%E5%85%B3%E5%8D%95%E4%BD%8D%E5%92%8C%E7%94%A8%E6%88%B7%E5%9C%A8%E9%83%A8%E7%BD%B2%E5%92%8C%E5%BA%94%E7%94%A8OpenClaw%E6%97%B6%EF%BC%8C%E5%85%85%E5%88%86%E6%A0%B8%E6%9F%A5%E5%85%AC%E7%BD%91%E6%9A%B4%E9%9C%B2%E6%83%85%E5%86%B5%E3%80%81%E6%9D%83%E9%99%90%E9%85%8D%E7%BD%AE%E5%8F%8A%E5%87%AD%E8%AF%81%E7%AE%A1%E7%90%86%E6%83%85%E5%86%B5%EF%BC%8C%E5%85%B3%E9%97%AD%E4%B8%8D%E5%BF%85%E8%A6%81%E7%9A%84%E5%85%AC%E7%BD%91%E8%AE%BF%E9%97%AE%EF%BC%8C%E5%AE%8C%E5%96%84%E8%BA%AB%E4%BB%BD%E8%AE%A4%E8%AF%81%E3%80%81%E8%AE%BF%E9%97%AE%E6%8E%A7%E5%88%B6%E3%80%81%E6%95%B0%E6%8D%AE%E5%8A%A0%E5%AF%86%E5%92%8C%E5%AE%89%E5%85%A8%E5%AE%A1%E8%AE%A1%E7%AD%89%E5%AE%89%E5%85%A8%E6%9C%BA%E5%88%B6%EF%BC%8C%E5%B9%B6%E6%8C%81%E7%BB%AD%E5%85%B3%E6%B3%A8%E5%AE%98%E6%96%B9%E5%AE%89%E5%85%A8%E5%85%AC%E5%91%8A%E5%92%8C%E5%8A%A0%E5%9B%BA%E5%BB%BA%E8%AE%AE%EF%BC%8C%E9%98%B2%E8%8C%83%E6%BD%9C%E5%9C%A8%E7%BD%91%E7%BB%9C%E5%AE%89%E5%85%A8%E9%A3%8E%E9%99%A9%E3%80%82%20%E5%85%8D%E8%B4%A3%E5%A3%B0%E6%98%8E%EF%BC%9A%E6%9C%AC%E5%86%85%E5%AE%B9%E6%9D%A5%E8%87%AA%E8%85%BE%E8%AE%AF%E5%B9%B3%E5%8F%B0%E5%88%9B%E4%BD%9C%E8%80%85%EF%BC%8C%E4%B8%8D%E4%BB%A3%E8%A1%A8%E8%85%BE%E8%AE%AF%E6%96%B0%E9%97%BB%E6%88%96%E8%85%BE%E8%AE%AF%E7%BD%91%E7%9A%84%E8%A7%82%E7%82%B9%E5%92%8C%E7%AB%8B%E5%9C%BA%E3%80%82) 
2. 自动化违反内容平台政策
3. News on US Depart. Defense's deal with OpenAI/Anthropic, Pentagon can trust on OpenAI 假如云服务商可以根据客户定制数据安全方案，那hybrid在safety上的竞争力会被打击吗？ 
