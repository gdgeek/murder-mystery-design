# 需求文档：线上剧本杀AI生成工具

## 简介

本系统是一个基于大语言模型（LLM）的线上剧本杀AI生成工具，由三个核心子系统组成：

1. **剧本生成子系统**：用户通过配置基本参数（玩家人数、时长、游戏类型、各项比例、背景设定等），系统调用LLM并结合知识库（Knowledge_Base），自动生成完整的剧本杀内容（DM手册、玩家手册、游戏物料）。生成的剧本作为独立资产存储，可被反复游玩。

2. **线上游玩子系统**：基于已生成的剧本，提供线上Web游戏平台，支持玩家选角、DM（人类或AI）线上组织游戏。同一剧本可创建多个游戏会话，被不同玩家反复游玩。

3. **知识库子系统**：系统的核心智能引擎，实现"学习→实践→反馈→精进"的持续学习闭环。从在线资源、已有剧本和玩家反馈中提取剧本杀设计知识，通过向量嵌入（Embedding）和RAG（检索增强生成）技术在生成时检索最相关的知识，并根据玩家反馈动态调整知识有效性权重。代码开源，但每个部署实例的知识库独立演化，适应各自的玩家群体偏好。

三个子系统通过评价反馈闭环连接：玩家完成游戏后提交评价和建议，这些数据反馈到知识库和生成子系统，驱动知识的持续积累和剧本质量的持续提升。系统采用Docker + Docker Compose容器化部署，通过GitHub Actions实现CI/CD。

## 技术选型

- **语言**：全栈 TypeScript
- **后端**：Node.js + TypeScript
- **前端**：Vue 3 + TypeScript
- **数据库**：MySQL（持久化存储）+ Redis（缓存、会话管理、实时状态）
- **容器化**：Docker + Docker Compose
- **CI/CD**：GitHub Actions
- **国际化**：支持多语言（中文、英文等），前后端均需实现 i18n 国际化方案

## 术语表

- **Generator（生成引擎）**：核心AI生成模块，负责调用LLM并结合知识库生成剧本杀内容
- **Skill（技能模板）**：预定义的剧本杀生成辅助模板和规则片段，作为知识库的初始种子数据导入，是Knowledge_Entry的一种特殊类别
- **DM_Handbook（DM手册）**：供主持人（DM）使用的完整游戏主持指南，包含时间线、线索分发规则、真相还原等
- **Player_Handbook（玩家手册）**：供每位玩家使用的角色剧本，包含角色背景、目标、线索等
- **Material（游戏物料）**：游戏过程中使用的线索卡、道具卡、投票卡等辅助材料
- **Game_Session（游戏会话）**：一次完整的线上剧本杀游戏实例，包含房间、玩家、DM和游戏状态
- **Config（配置参数）**：用户定义的剧本生成参数集合
- **Feedback_System（反馈系统）**：收集玩家评价并用于优化后续生成内容的模块
- **Script（剧本）**：由Generator生成的完整剧本杀内容包，包含DM手册、所有玩家手册和游戏物料。剧本作为独立资产存储，可被反复游玩和持续优化
- **Script_Repository（剧本库）**：存储所有已生成剧本的持久化仓库，供游玩子系统选择和使用
- **Game_Platform（游戏平台）**：线上Web游戏平台，基于剧本库中的已有剧本创建游戏会话，支持玩家选角、DM组织和实时游戏交互
- **LLM_Adapter（LLM适配器）**：与大语言模型API交互的适配层，支持多种LLM提供商
- **AI_DM（AI主持人）**：基于LLM的自动化游戏主持人，能够替代人类DM组织和主持游戏，支持语音输出
- **TTS_Engine（语音合成引擎）**：将文本转换为语音的模块，用于AI_DM的语音主持功能
- **Leaderboard（排行榜）**：展示剧本评分排名的模块，根据玩家评价数据进行排序
- **Live_Suggestion（实时建议）**：游戏过程中玩家提交的优化建议，用于实时或后续改进剧本内容
- **Player_Account（玩家账户）**：玩家的注册账户，包含身份信息、游戏历史、成就和收藏数据
- **Achievement（成就）**：基于玩家游戏行为和次数解锁的奖励徽章
- **Collection（收藏）**：玩家通过游玩积累的收藏品，如结局卡、角色卡、稀有线索卡等
- **Video_Generator（视频生成器）**：基于AI的视频生成模块，可将关键剧情文本转换为短视频，用于游戏中播放增强沉浸感
- **Asset_Storage（资源存储）**：统一的资源存储抽象层，支持本地存储和云存储（如腾讯COS）插件扩展，管理图片、视频、音乐等多媒体资源
- **Plugin_System（插件系统）**：可扩展的插件架构，支持通过插件接入不同的AI生成服务（图片、音乐、视频）和云存储服务
- **Tag（标签）**：剧本的分类标签，用于检索和筛选，包括系统自动标签和用户自定义标签
- **AI_Music_Generator（AI音乐生成器）**：基于AI生成背景音乐和氛围音效的模块
- **AI_Image_Generator（AI图片生成器）**：基于AI生成场景图、角色立绘、线索卡插图等图片资源的模块
- **Knowledge_Base（知识库）**：存储所有知识条目及其元数据的结构化数据集合，每个部署实例拥有独立的知识库，通过反馈闭环持续演化
- **Knowledge_Entry（知识条目）**：一条具体的剧本杀设计知识，包含内容描述、分类、向量嵌入、有效性权重和来源信息。Skill模板作为初始种子数据导入后成为Knowledge_Entry的一种
- **Embedding（向量嵌入）**：知识条目内容的高维向量表示，用于语义相似度搜索（RAG检索）
- **Vector_Store（向量存储）**：存储和检索向量嵌入的数据结构，支持近似最近邻搜索，使用进程内方案（如hnswlib）
- **Effectiveness_Score（有效性分数）**：知识条目的质量评分（0.0-1.0），由反馈数据驱动动态调整，反映该知识在实际剧本生成中的效果
- **Knowledge_Category（知识分类）**：知识条目的分类维度，包括：techniques（设计技巧）、patterns（结构模式）、examples（优秀案例）、anti_patterns（反面教材）、player_preferences（玩家偏好）、prompt_templates（提示词模板）、reasoning_chains（推理链模式）
- **Knowledge_Source（知识来源）**：知识条目的来源类型，包括：seed（Skill种子导入）、manual（手动录入）、article（文章提取）、script_analysis（剧本分析）、feedback_extraction（反馈提炼）、high_rated_script（高评分剧本提取）
- **Learning_Pipeline（学习管道）**：知识库的闭环学习流程：摄入→提取→嵌入→应用→评估→精炼
- **RAG（检索增强生成）**：在LLM生成时，通过向量相似度搜索从知识库中检索相关知识条目，注入到提示词上下文中以提升生成质量
- **Knowledge_Snapshot（知识快照）**：某一时刻知识库的完整状态快照，用于版本追溯和回滚
- **Feedback_Knowledge_Link（反馈知识关联）**：记录每次剧本生成使用了哪些知识条目，用于将玩家反馈关联回具体知识条目以更新有效性分数
- **Embedding_Provider（嵌入提供商）**：生成向量嵌入的AI服务适配层，支持配置不同的嵌入模型提供商
- **Pre_Generation_Interview（预生成访谈）**：在剧本生成前，AI与创作者进行的交互式访谈，通过自适应问答收集游戏设计偏好，访谈结果作为生成上下文的一部分
- **Interview_Template（访谈模板）**：定义访谈问题流程和分支逻辑的模板，本身也是知识库中的知识条目，可通过反馈持续优化
- **Interview_Session（访谈会话）**：一次完整的预生成访谈实例，记录所有问答对和最终生成的访谈摘要
- **Prompt_Template（提示词模板）**：可版本化管理的LLM提示词模板，支持变量插值和条件逻辑
- **Prompt_Version（提示词版本）**：提示词模板的版本记录，支持A/B测试和回滚
- **Few_Shot_Example（少样本示例）**：用于LLM提示词中的输入-输出示例对，按类别和游戏类型组织
- **Reasoning_Chain（推理链模式）**：用于复杂剧本逻辑生成的链式思维推理模板
- **Token_Usage_Record（Token用量记录）**：记录每次LLM调用的token消耗、成本和响应时间
- **Model_Evaluation（模型评估）**：对不同LLM模型或提示词版本的生成质量进行对比评估的记录
- **AB_Test（A/B测试）**：对两个提示词版本或模型配置进行对比实验，根据玩家反馈确定优胜方案
- **Player_Rating_Weight（玩家评价权重）**：基于玩家游玩次数的评价权重系数，资深玩家（游玩次数多）的评价在反馈计算和排行榜排名中占更大权重
- **Designer_Identity（设计师身份）**：用户的剧本设计师身份，与Player_Account中的玩家身份独立，拥有独立的等级、经验值和牌库系统
- **Designer_Level（设计师等级）**：设计师身份的等级，通过设计剧本和剧本被游玩获得经验值升级，等级越高可用的点数和可解锁的牌越多
- **Designer_XP（设计师经验值）**：设计师身份的经验值，通过创建剧本、剧本被游玩、获得高评价等行为积累
- **Skill_Card（技能牌）**：将Skill模板包装为可选择的"牌"，设计师在创建剧本时从牌库中选择技能牌来定制生成效果
- **Card_Tier（牌等级）**：技能牌的稀有度/等级分类，分为基础（Basic）、进阶（Advanced）、专家（Expert）、传奇（Legendary）四个等级
- **Card_Point_Cost（牌点数消耗）**：每张技能牌在使用时消耗的点数，等级越高的牌消耗点数越多
- **Design_Point_Budget（设计点数预算）**：设计师每次创建剧本时可用的总点数，由设计师等级决定，等级越高可用点数越多
- **Designer_Deck（设计师牌库）**：设计师已解锁并可使用的技能牌集合，设计师可以从中组合自己的常用牌组
- **Custom_Deck（自定义牌组）**：设计师从牌库中挑选组合的个人牌组，方便快速选牌
- **Designer_Reward（设计师奖励）**：当设计师创建的剧本被其他玩家游玩时获得的经验值和特殊奖励
- **Player_Level（玩家等级）**：用户的玩家身份等级，通过游玩剧本获得经验值升级，与设计师等级独立

## 需求

### 需求 1：剧本生成参数配置

**用户故事：** 作为剧本创作者，我希望通过配置基本参数来定义剧本杀的基础设定，以便系统能根据我的需求生成定制化的剧本。

#### 验收标准

1. WHEN 用户创建新剧本项目时，THE Config SHALL 提供以下必填参数的配置界面：玩家人数（1-6人，DM始终由AI担任）、游戏时长（2-6小时，以小时为单位选择）、游戏类型（本格、新本格、变格中选择一种，均为推理类型；机制类和情感类暂不支持，预留扩展）、目标年龄段（小学生、中学生、大学生、成年人中选择一种）
2. WHEN 用户设置游戏比例参数时，THE Config SHALL 接受还原比例和推理比例的数值输入，两项比例之和等于100%
3. WHEN 用户设置背景参数时，THE Config SHALL 接受时代背景、地点设定、主题风格的文本输入
4. WHEN 用户提交的参数不符合约束条件时，THE Config SHALL 返回具体的参数校验错误信息，指明哪个参数不合法及其合法范围
5. WHEN 用户完成所有必填参数配置后，THE Config SHALL 生成一个完整的配置对象并传递给Generator
6. WHEN 用户选择游戏时长后，THE Config SHALL 根据时长自动适配游戏轮次结构：每轮包含阅读、搜证、推证三个阶段。2小时适配2轮（每轮约50分钟+总结20分钟），3小时适配3轮（每轮约50分钟+总结30分钟），4小时适配3-4轮，5小时适配4轮，6小时适配4-5轮
7. WHEN Config生成轮次结构时，THE Config SHALL 为每轮分配阅读时间（约10-15分钟）、搜证时间（约15-20分钟）、推证/讨论时间（约15-20分钟），并预留最终投票和真相揭示时间
8. WHEN 用户选择目标年龄段后，THE Config SHALL 将年龄段信息传递给Generator，用于调整剧本的内容复杂度、用词难度、主题适宜性和推理难度（小学生：简单词汇、轻松主题、低推理难度；中学生：适中复杂度；大学生：较高复杂度；成年人：不限制）

### 需求 2：Skill库管理（知识库种子数据）

**用户故事：** 作为剧本创作者，我希望系统内置丰富的剧本杀Skill模板库作为知识库的初始种子数据，以便AI生成时能参考专业的剧本杀设计规则和模式。

#### 验收标准

1. THE Skill SHALL 包含以下类别的预定义模板：角色设计、线索设计、时间线构建、动机设计、诡计设计、还原逻辑、推理链条
2. WHEN Generator请求特定类别的Skill时，THE Skill SHALL 返回该类别下所有可用的模板列表
3. WHEN 用户选择游戏类型为"本格"时，THE Skill SHALL 优先提供本格推理类Skill模板（如密室诡计、不在场证明、物证链条、公平线索布局、逻辑推理链）
4. WHEN 用户选择游戏类型为"新本格"时，THE Skill SHALL 优先提供新本格类Skill模板（如特殊设定构建、设定内公平推理、超能力边界设计、叙述性诡计、时间线诡计、身份诡计、设定规则利用型诡计、多重反转）
5. WHEN 用户选择游戏类型为"变格"时，THE Skill SHALL 优先提供变格类Skill模板（如心理悬疑、氛围营造、角色内心独白、道德困境、开放式结局）
6. THE Skill SHALL 支持以JSON格式存储和读取Skill模板数据
7. WHEN 读取Skill模板JSON数据后再序列化回JSON时，THE Skill SHALL 产生与原始数据等价的结果（往返一致性）
8. WHEN 系统首次初始化知识库时，THE Skill SHALL 将所有预定义Skill模板作为种子数据批量导入Knowledge_Base，来源类型标记为"seed"

### 需求 3：AI剧本生成引擎

**用户故事：** 作为剧本创作者，我希望系统能调用大语言模型，结合知识库自动生成完整的剧本杀内容，以便我能快速获得高质量的剧本。

#### 验收标准

1. WHEN 用户提交有效的Config后，THE Generator SHALL 调用LLM_Adapter生成完整的Script，包含DM_Handbook、所有Player_Handbook和Material
2. WHEN Generator生成Script时，THE Generator SHALL 确保生成的玩家手册数量与Config中指定的玩家人数一致
3. WHEN Generator生成Script时，THE Generator SHALL 确保每个角色的时间线在逻辑上无矛盾
4. WHEN Generator生成Script时，THE Generator SHALL 确保所有线索在DM_Handbook中有对应的分发规则
5. IF LLM_Adapter调用失败，THEN THE Generator SHALL 返回包含错误类型和建议操作的错误信息
6. WHEN Generator生成Script时，THE Generator SHALL 根据Config中的还原比例和推理比例调整内容侧重
7. WHEN Generator生成Script时，THE Generator SHALL 生成分支叙事结构，包含多个剧情分支点（由玩家投票决定走向）和多个结局（至少3个以上），充分利用AI动态生成的优势
8. WHEN Generator生成分支结构时，THE Generator SHALL 确保每个分支路径的线索链条完整且逻辑自洽，不同分支的真相可以不同
9. WHEN Generator完成生成后，THE Generator SHALL 将Script序列化为JSON格式存储到Script_Repository，反序列化后与原始Script等价
10. WHEN Generator生成Script后，THE Generator SHALL 自动为Script生成标签，包括：游戏类型、年龄段、玩家人数、时代背景、主题风格等，并支持创作者手动添加自定义标签
11. WHEN Generator基于反馈优化已有Script时，THE Generator SHALL 创建新版本而非覆盖原始版本，保留完整的版本历史
12. WHEN Generator生成新剧本时，THE Generator SHALL 通过RAG从Knowledge_Base中检索与当前Config语义最相关的知识条目，嵌入到LLM提示词上下文中
13. WHEN Generator完成剧本生成后，THE Generator SHALL 记录本次生成使用了哪些知识条目（Feedback_Knowledge_Link），用于后续反馈关联

### 需求 4：DM手册生成

**用户故事：** 作为DM（主持人），我希望获得一份完整的DM手册，以便我能顺利组织和主持一场剧本杀游戏。

#### 验收标准

1. WHEN Generator生成DM_Handbook时，THE DM_Handbook SHALL 包含以下章节：游戏概述、角色列表、完整时间线、线索分发表、各轮次流程指引、分支决策点定义、多结局描述、真相还原、结局判定规则
2. WHEN DM_Handbook包含线索分发表时，THE DM_Handbook SHALL 为每条线索指定分发时机、接收角色和分发条件
3. WHEN DM_Handbook包含时间线时，THE DM_Handbook SHALL 确保时间线中的事件按时间顺序排列且无逻辑矛盾
4. WHEN DM_Handbook包含分支决策点时，THE DM_Handbook SHALL 为每个分支点定义投票问题、选项、以及各选项对应的后续剧情走向
5. WHEN DM_Handbook包含多结局时，THE DM_Handbook SHALL 为每个结局定义触发条件（基于投票和玩家行为的累积）、结局叙述和每位玩家的个人结局
6. WHEN DM_Handbook包含结局判定规则时，THE DM_Handbook SHALL 定义明确的胜负条件和评分标准

### 需求 5：玩家手册生成

**用户故事：** 作为玩家，我希望获得一份专属的角色手册，以便我能了解自己的角色并参与游戏。

#### 验收标准

1. WHEN Generator生成Player_Handbook时，THE Player_Handbook SHALL 包含以下内容：角色名称、角色背景故事、角色目标、角色关系描述、已知线索、各轮次行动指引
2. WHEN Generator生成Player_Handbook时，THE Player_Handbook SHALL 确保每个玩家手册只包含该角色应知的信息，不泄露其他角色的秘密
3. WHEN Generator为同一Script生成多份Player_Handbook时，THE Generator SHALL 确保不同角色的背景故事在交叉引用时保持一致
4. WHEN Player_Handbook包含角色目标时，THE Player_Handbook SHALL 为每个角色提供至少一个主要目标和一个次要目标

### 需求 6：游戏物料生成

**用户故事：** 作为DM，我希望系统能生成游戏所需的各种物料，以便线上游戏时能方便地分发和使用。

#### 验收标准

1. WHEN Generator生成Material时，THE Material SHALL 包含以下类型的物料：线索卡、道具卡、投票卡、场景描述卡
2. WHEN Material包含线索卡时，THE Material SHALL 确保每张线索卡有唯一标识符、内容描述和关联角色
3. WHEN Material中的线索卡与DM_Handbook中的线索分发表关联时，THE Material SHALL 确保线索卡标识符与分发表中的引用一致

### 需求 7：线上游戏平台

**用户故事：** 作为玩家，我希望能在Web平台上参与剧本杀游戏，以便我能远程与其他玩家一起游戏。

#### 验收标准

1. WHEN 玩家创建Game_Session时，THE Game_Platform SHALL 从Script_Repository中展示可用剧本列表供选择，并生成包含二维码的房间页面，可在手机、iPad或电脑上展示
2. WHEN 其他玩家通过手机扫描房间二维码时，THE Game_Platform SHALL 在玩家手机浏览器中打开游戏页面并加入该Game_Session
3. WHEN 玩家加入Game_Session后，THE Game_Platform SHALL 显示可选角色列表，已被选择的角色标记为不可选
4. WHEN 所有玩家完成选角并全部点击"开始游戏"后，THE Game_Platform SHALL 通知AI_DM接管游戏，正式开始主持
5. WHILE Game_Session处于进行中状态时，THE Game_Platform SHALL 提供实时文字聊天功能供玩家交流
6. WHILE Game_Session处于进行中状态时，THE Game_Platform SHALL 由AI_DM按流程自动分发线索卡和物料给指定玩家
7. WHEN AI_DM推进游戏轮次时，THE Game_Platform SHALL 更新所有玩家的游戏阶段显示
8. WHEN AI_DM在推理环节提出关键问题时，THE Game_Platform SHALL 向所有玩家展示投票界面，收集玩家的选择
9. WHEN 玩家完成投票后，THE AI_DM SHALL 根据投票结果动态选择剧情分支，推进不同的故事走向
10. WHEN Game_Session进入结局阶段时，THE Game_Platform SHALL 根据游戏过程中所有投票和玩家行为的累积结果，展示对应的多结局之一
11. WHEN Game_Session结束后，THE AI_DM SHALL 为每位玩家生成个人评价，包含推理表现、角色扮演表现、关键决策回顾
12. THE Game_Platform SHALL 支持响应式布局，适配手机、iPad和电脑三种屏幕尺寸
8. WHEN Game_Session结束时，THE Game_Platform SHALL 展示对应结局的真相还原、游戏结果和AI为每位玩家生成的个人评价
9. WHILE Game_Session处于进行中状态时，THE Game_Platform SHALL 提供建议提交入口，允许玩家提交对当前剧本的优化建议（Live_Suggestion）
10. WHEN 玩家提交Live_Suggestion时，THE Game_Platform SHALL 将建议内容与当前Game_Session和Script关联存储
11. THE Game_Platform SHALL 支持同一Script创建多个独立的Game_Session，各会话之间互不影响，不同会话可能因投票不同而走向不同结局

### 需求 8：评价反馈系统

**用户故事：** 作为玩家，我希望在游戏结束后能对剧本进行评价，以便AI能根据反馈持续优化生成质量。

#### 验收标准

1. WHEN Game_Session结束后，THE Feedback_System SHALL 向每位玩家展示评价表单，包含以下维度：剧情评分（1-10）、推理难度评分（1-10）、角色体验评分（1-10）、整体满意度评分（1-10）、文字评价
2. WHEN 玩家提交评价时，THE Feedback_System SHALL 验证所有评分在1-10的有效范围内
3. WHEN Feedback_System收集到评价数据后，THE Feedback_System SHALL 将评价数据与对应的Script和Config关联存储
4. WHEN Generator生成新Script时，THE Generator SHALL 查询历史评价数据，将平均评分低于6分的维度作为优化重点纳入LLM提示词
5. WHEN Generator生成新Script时，THE Generator SHALL 查询关联的Live_Suggestion数据，将高频建议纳入LLM提示词以优化生成内容
6. WHEN 用户选择对已有Script进行优化提炼时，THE Feedback_System SHALL 汇总该Script所有Game_Session的评价和建议，提供给Generator作为优化输入
7. WHEN 某Script累计收到的评价数量达到阈值（默认5次）且任一维度平均评分低于6分时，THE Feedback_System SHALL 自动触发Generator对该Script进行优化，生成新版本并标记版本号
8. WHEN 自动优化触发时，THE Generator SHALL 基于所有评价和建议数据生成优化后的Script新版本，版本号自动递增（如v1.0→v1.1），原版本保留不变
9. WHEN 自动优化完成后，THE Feedback_System SHALL 通知剧本创建者新版本已生成，并展示优化摘要（哪些维度被优化、参考了哪些反馈）
10. WHEN Feedback_System计算Script的各维度平均评分时，THE Feedback_System SHALL 根据评价者的Player_Account游玩次数应用评价权重（游玩≥25次权重1.5，≥10次权重1.2，≥5次权重1.0，<5次权重0.7），使用加权平均而非简单平均

### 需求 9：剧本排行榜

**用户故事：** 作为玩家，我希望能查看剧本排行榜，以便发现高质量的剧本并选择参与。

#### 验收标准

1. THE Leaderboard SHALL 根据玩家评价的整体满意度平均分对Script进行排名
2. WHEN 查看排行榜时，THE Leaderboard SHALL 展示剧本名称、游戏类型、玩家人数、平均评分和游玩次数
3. WHEN 新的评价数据提交后，THE Leaderboard SHALL 重新计算对应Script的排名
4. WHEN 排行榜中的Script游玩次数少于3次时，THE Leaderboard SHALL 将该Script标记为"评分待定"而非显示排名
5. THE Leaderboard SHALL 支持按标签筛选和搜索剧本，玩家可通过游戏类型、年龄段、玩家人数、主题风格等标签组合查询
6. WHEN Leaderboard计算Script排名评分时，THE Leaderboard SHALL 使用与Feedback_System一致的玩家评价权重（基于游玩次数），确保资深玩家的评价对排名影响更大

### 需求 10：AI主持人（AI DM）

**用户故事：** 作为玩家，我希望游戏由AI主持人自动组织和主持，以便随时开局无需等待人类DM。

#### 验收标准

1. WHEN 创建Game_Session时，THE Game_Platform SHALL 自动分配AI_DM作为主持人，无需人类DM
2. WHEN 使用AI_DM主持游戏时，THE AI_DM SHALL 根据DM_Handbook中的流程指引自动推进游戏轮次
3. WHEN AI_DM需要分发线索时，THE AI_DM SHALL 根据DM_Handbook中的线索分发表在正确时机向指定玩家分发线索
4. WHEN AI_DM主持游戏时，THE AI_DM SHALL 通过TTS_Engine将主持文本转换为语音输出，优先使用实时TTS流式生成；若实时生成延迟过高（超过2秒），则回退到使用预生成的语音资源
5. WHEN Generator生成Script时，THE Generator SHALL 为DM_Handbook中的固定叙述文本（开场白、轮次过渡、结局揭示等）预生成TTS语音文件，存储到Asset_Storage
6. WHEN 玩家向AI_DM提问时，THE AI_DM SHALL 根据DM_Handbook中的信息范围回答问题，避免泄露不应公开的信息，回答语音通过实时TTS生成
7. IF TTS_Engine调用失败，THEN THE AI_DM SHALL 回退到文字主持模式并通知玩家

### 需求 11：LLM适配层

**用户故事：** 作为系统管理员，我希望系统能灵活接入不同的大语言模型提供商，以便根据需要切换或升级AI能力。

#### 验收标准

1. THE LLM_Adapter SHALL 提供统一的接口用于发送提示词和接收生成结果
2. WHEN 配置LLM提供商时，THE LLM_Adapter SHALL 支持通过环境变量配置API密钥和端点地址
3. IF LLM API返回错误响应，THEN THE LLM_Adapter SHALL 实施最多3次的指数退避重试策略
4. IF 重试次数耗尽后仍失败，THEN THE LLM_Adapter SHALL 返回包含错误详情的结构化错误对象
5. WHEN LLM_Adapter发送请求时，THE LLM_Adapter SHALL 记录请求的token数量和响应时间用于监控

### 需求 12：玩家账户系统

**用户故事：** 作为玩家，我希望拥有自己的账户，以便记录我的游戏历史、管理个人信息和跨设备登录。

#### 验收标准

1. THE Player_Account SHALL 支持邮箱注册和登录
2. WHEN 玩家登录后，THE Player_Account SHALL 展示个人主页，包含昵称、头像、游戏统计（总游玩次数、参与剧本数、解锁结局数）
3. WHEN 玩家参与Game_Session时，THE Player_Account SHALL 自动记录游戏历史，包含剧本名称、扮演角色、游戏时间、达成结局
4. WHEN 玩家未登录时，THE Game_Platform SHALL 要求玩家登录或注册后才能加入Game_Session
5. THE Player_Account SHALL 支持修改昵称和头像

### 需求 13：收藏与成就奖励系统

**用户故事：** 作为玩家，我希望根据游玩次数和表现获得收藏品和成就奖励，以便增加游戏的长期吸引力和成就感。

#### 验收标准

1. WHEN 玩家完成一局Game_Session后，THE Collection SHALL 自动为玩家解锁该局对应的结局卡和角色卡收藏品
2. WHEN 玩家在同一Script中达成不同结局时，THE Collection SHALL 分别记录每个结局卡，玩家可在个人主页查看已收集和未收集的结局
3. THE Achievement SHALL 定义以下基于游玩次数的成就里程碑：首次游玩（1次）、初入江湖（5次）、推理达人（10次）、侦探大师（25次）、传奇侦探（50次）
4. WHEN 玩家游玩次数达到成就里程碑时，THE Achievement SHALL 自动解锁对应成就徽章并通知玩家
5. WHEN 玩家在游戏中获得AI_DM的高评价（推理表现评分≥8）时，THE Achievement SHALL 解锁特殊成就徽章（如"明察秋毫"、"逻辑之王"）
6. THE Player_Account SHALL 在个人主页展示已解锁的成就徽章和收藏品列表

### 需求 14：AI关键剧情视频生成（可选）

**用户故事：** 作为剧本创作者，我希望能为关键剧情生成AI视频，以便在游戏中播放增强玩家的沉浸感。

#### 验收标准

1. WHEN Generator生成Script时，THE Generator SHALL 标记关键剧情节点（如案件发生、重大反转、结局揭示）为可生成视频的场景
2. WHEN 剧本创作者选择为关键剧情生成视频时，THE Video_Generator SHALL 根据剧情文本描述调用AI视频生成服务，生成对应的短视频（10-30秒）
3. WHEN Video_Generator生成视频后，THE Video_Generator SHALL 将视频文件与对应的Script和剧情节点关联存储
4. WHEN AI_DM在游戏中推进到已关联视频的剧情节点时，THE Game_Platform SHALL 向所有玩家同步播放对应的剧情视频
5. IF 视频生成失败或视频文件不存在，THEN THE Game_Platform SHALL 回退到文字叙述模式，不影响游戏正常进行
6. THE Video_Generator SHALL 为视频生成功能标记为可选项，剧本可以在无视频的情况下正常运行

### 需求 15：插件系统与资源存储

**用户故事：** 作为系统管理员，我希望系统具备可扩展的插件架构，以便灵活接入不同的AI生成服务和云存储服务。

#### 验收标准

1. THE Plugin_System SHALL 定义统一的插件接口规范，支持以下插件类型：存储插件（Storage）、图片生成插件（ImageGen）、音乐生成插件（MusicGen）、视频生成插件（VideoGen）、TTS语音插件（TTSGen）
2. THE Asset_Storage SHALL 提供统一的资源存取接口，默认使用本地文件存储，可通过存储插件切换到云存储（如腾讯COS、AWS S3等）
3. WHEN 系统配置了云存储插件时，THE Asset_Storage SHALL 将所有生成的多媒体资源（图片、视频、音乐、语音）上传到配置的云存储服务
4. THE Plugin_System SHALL 支持通过配置文件注册和启用/禁用插件，无需修改核心代码
5. WHEN 插件调用失败时，THE Plugin_System SHALL 记录错误日志并回退到默认行为（如云存储失败回退到本地存储）

### 需求 16：AI多媒体资源生成（可选）

**用户故事：** 作为剧本创作者，我希望系统能通过AI生成游戏所需的图片、音乐等多媒体资源，以便增强游戏的沉浸感。

#### 验收标准

1. WHEN 剧本创作者选择生成图片资源时，THE AI_Image_Generator SHALL 根据剧本中的场景描述和角色描述，生成场景图、角色立绘和线索卡插图
2. WHEN 剧本创作者选择生成音乐资源时，THE AI_Music_Generator SHALL 根据剧本的主题风格和氛围，生成背景音乐和氛围音效
3. WHEN AI生成多媒体资源后，THE Asset_Storage SHALL 将资源文件与对应的Script关联存储
4. WHEN AI_DM在游戏中推进到特定场景时，THE Game_Platform SHALL 自动播放关联的背景音乐和展示场景图片
5. IF 多媒体资源生成失败或不存在，THEN THE Game_Platform SHALL 正常运行游戏，不影响核心游戏流程
6. THE AI_Image_Generator 和 AI_Music_Generator SHALL 均为可选功能，通过插件系统接入，剧本可在无多媒体资源的情况下正常运行

### 需求 17：容器化部署与CI/CD

**用户故事：** 作为开发者，我希望系统能通过Docker容器化部署并配置自动化CI/CD流水线，以便实现可靠的部署和持续交付。

#### 验收标准

1. THE System SHALL 提供Dockerfile用于构建前端和后端的容器镜像
2. THE System SHALL 提供docker-compose.yml文件，定义所有服务（前端、后端、数据库MySQL和Redis）的编排配置
3. WHEN 执行docker-compose up命令时，THE System SHALL 启动所有服务并使系统可访问
4. THE System SHALL 提供GitHub Actions工作流配置文件，包含代码检查、测试运行和镜像构建步骤
5. WHEN 代码推送到main分支时，THE System SHALL 触发CI/CD流水线自动执行构建和测试

### 需求 18：多语言国际化支持

**用户故事：** 作为国际用户，我希望系统支持多语言切换，以便不同语言的用户都能使用本平台。

#### 验收标准

1. THE Game_Platform SHALL 支持至少中文和英文两种语言，用户可在界面中切换
2. THE Game_Platform SHALL 使用 Vue I18n 实现前端国际化，所有界面文本通过语言包管理，不硬编码
3. THE System SHALL 在后端 API 响应中支持根据请求语言返回对应语言的错误信息和提示文本
4. WHEN Generator生成Script时，THE Generator SHALL 根据用户选择的语言生成对应语言的剧本内容
5. THE System SHALL 支持后续扩展更多语言，语言包采用独立文件管理，新增语言只需添加对应语言包文件

### 需求 19：知识条目数据模型与向量存储

**用户故事：** 作为系统管理员，我希望知识条目以结构化方式存储并支持语义搜索，以便系统能通过RAG技术高效检索相关知识来指导剧本生成。

#### 验收标准

1. THE Knowledge_Base SHALL 为每个Knowledge_Entry存储以下字段：唯一标识符、内容描述、知识分类（Knowledge_Category）、适用游戏类型列表（Game_Type_Tag）、有效性分数（Effectiveness_Score，0.0-1.0，默认0.5）、来源类型（Knowledge_Source）、来源引用、向量嵌入（Embedding）、使用次数、创建时间、更新时间、状态（active/deprecated）
2. WHEN 存储Knowledge_Entry到数据库后再读取时，THE Knowledge_Base SHALL 产生与原始数据等价的结果（往返一致性）
3. WHEN Knowledge_Entry的有效性分数不在0.0到1.0范围内时，THE Knowledge_Base SHALL 拒绝该操作并返回具体的校验错误信息
4. THE Knowledge_Base SHALL 支持按知识分类、游戏类型标签、状态和有效性分数范围组合查询知识条目
5. WHEN 序列化Knowledge_Entry为JSON格式后再反序列化时，THE Knowledge_Base SHALL 产生与原始对象等价的结果
6. WHEN 创建或更新Knowledge_Entry时，THE Knowledge_Base SHALL 调用Embedding_Provider为内容描述生成向量嵌入，并存储到Vector_Store中
7. THE Vector_Store SHALL 支持基于余弦相似度的近似最近邻搜索，返回与查询向量最相似的前K个知识条目

### 需求 20：知识管理与导入

**用户故事：** 作为管理员，我希望能手动管理知识条目并从外部来源批量导入，以便快速丰富知识库内容。

#### 验收标准

1. WHEN 管理员提交包含内容描述和知识分类的新知识条目时，THE Knowledge_Base SHALL 创建该条目并分配默认有效性分数0.5、来源类型"manual"，同时生成向量嵌入
2. WHEN 管理员编辑已有知识条目的内容时，THE Knowledge_Base SHALL 更新对应字段、更新时间，并重新生成向量嵌入
3. WHEN 管理员将知识条目状态设置为"deprecated"时，THE Knowledge_Base SHALL 将该条目标记为弃用，Generator在后续RAG检索中排除该条目
4. WHEN 管理员提交一篇文章文本进行知识提取时，THE Knowledge_Base SHALL 调用LLM_Adapter分析文本，提取结构化知识条目列表供管理员审核后导入
5. WHEN 管理员提交一个已有剧本进行分析时，THE Knowledge_Base SHALL 调用LLM_Adapter分析剧本结构，提取设计技巧作为知识条目供审核后导入
6. WHEN 管理员手动调整知识条目有效性分数时，THE Knowledge_Base SHALL 记录一条权重更新记录，包含调整原因"manual"、调整前后分数值和操作时间
7. IF LLM_Adapter在知识提取过程中调用失败，THEN THE Knowledge_Base SHALL 返回包含错误类型和建议操作的错误信息

### 需求 21：RAG检索增强生成

**用户故事：** 作为剧本创作者，我希望系统在生成剧本时通过语义搜索从知识库中检索最相关的知识，以便生成更高质量、更符合需求的剧本。

#### 验收标准

1. WHEN Generator生成新剧本时，THE Generator SHALL 将当前Config的游戏类型、主题风格、时代背景等参数组合为查询文本，调用Embedding_Provider生成查询向量
2. WHEN Generator获取查询向量后，THE Generator SHALL 在Vector_Store中执行语义搜索，检索与查询最相关的active状态知识条目
3. WHEN Generator检索到知识条目后，THE Generator SHALL 按有效性分数降序排列，选取前N个条目（N由配置参数决定，默认20），并根据LLM上下文窗口大小动态调整N值
4. WHEN Generator组装LLM提示词时，THE Generator SHALL 将选取的知识条目内容作为生成指导嵌入提示词中，有效性分数越高的条目在提示词中的位置越靠前
5. WHILE Knowledge_Base中没有任何active状态的知识条目时，THE Generator SHALL 使用原有的Skill模板作为回退方案正常生成剧本
6. WHEN Generator完成剧本生成后，THE Generator SHALL 创建Feedback_Knowledge_Link记录，关联本次生成使用的所有知识条目ID到生成的Script

### 需求 22：反馈驱动的有效性分数更新

**用户故事：** 作为系统管理员，我希望系统能根据玩家反馈自动调整知识条目的有效性分数，以便知识库持续优化、越用越聪明。

#### 验收标准

1. WHEN Feedback_System收集到某Script的新评价后，THE Knowledge_Base SHALL 通过Feedback_Knowledge_Link查询该Script生成时使用的知识条目列表
2. WHEN Knowledge_Base获取到关联知识条目后，THE Knowledge_Base SHALL 根据评价的各维度评分计算综合反馈分数（加权平均：剧情30%、推理难度20%、角色体验30%、整体满意度20%）
3. WHEN 综合反馈分数高于7分时，THE Knowledge_Base SHALL 对关联的每个知识条目有效性分数增加正向调整量（delta = 0.02 × (score - 7) / 3），有效性分数上限为1.0
4. WHEN 综合反馈分数低于5分时，THE Knowledge_Base SHALL 对关联的每个知识条目有效性分数减少负向调整量（delta = 0.02 × (5 - score) / 5），有效性分数下限为0.0
5. WHEN 综合反馈分数在5分到7分之间时，THE Knowledge_Base SHALL 保持关联知识条目的有效性分数不变
6. WHEN 知识条目有效性分数发生调整时，THE Knowledge_Base SHALL 为每个调整创建一条权重更新记录，记录调整原因"feedback"、关联的Feedback ID、调整前后分数值
7. WHEN 某知识条目的有效性分数降至0.1以下且连续5次反馈均为负向调整时，THE Knowledge_Base SHALL 将该条目状态自动设置为"deprecated"并通知管理员
8. WHEN Knowledge_Base计算综合反馈分数时，THE Knowledge_Base SHALL 根据评价者的Player_Account游戏记录计算评价权重：游玩次数≥25次的玩家权重为1.5，游玩次数≥10次的玩家权重为1.2，游玩次数≥5次的玩家权重为1.0，游玩次数<5次的玩家权重为0.7。加权后的综合分数用于知识条目有效性分数调整

### 需求 23：反馈知识提炼与高评分剧本学习

**用户故事：** 作为系统管理员，我希望系统能从玩家反馈和高评分剧本中自动提炼新知识，以便知识库持续扩充和进化。

#### 验收标准

1. WHEN 某Script累计收到的文字评价和Live_Suggestion数量达到阈值（默认10条）时，THE Knowledge_Base SHALL 自动触发知识提炼流程
2. WHEN 知识提炼流程触发后，THE Knowledge_Base SHALL 调用LLM_Adapter分析所有文字评价和建议，提取可操作的设计改进建议作为候选知识条目
3. WHEN Knowledge_Base提取出候选知识条目后，THE Knowledge_Base SHALL 将候选列表提交给管理员审核，管理员可确认、修改或拒绝
4. WHEN 管理员确认候选知识条目后，THE Knowledge_Base SHALL 创建新知识条目，来源类型设置为"feedback_extraction"，初始有效性分数设置为0.5
5. WHEN 某Script的整体满意度平均分达到8分以上且游玩次数达到5次以上时，THE Knowledge_Base SHALL 自动触发高评分剧本学习流程，调用LLM_Adapter分析该剧本的设计特点，提取成功经验作为候选知识条目
6. WHEN 高评分剧本学习提取出候选知识条目后，THE Knowledge_Base SHALL 将候选列表提交给管理员审核，来源类型标记为"high_rated_script"
7. IF LLM_Adapter在知识提炼过程中调用失败，THEN THE Knowledge_Base SHALL 记录错误日志并在下次达到阈值时重试

### 需求 24：知识库快照、导出与部署差异化

**用户故事：** 作为系统管理员，我希望能管理知识库版本并支持导出导入，以便在不同部署实例间共享知识，同时保持各实例的独立演化。

#### 验收标准

1. WHEN 管理员手动触发快照创建时，THE Knowledge_Base SHALL 创建一个Knowledge_Snapshot，包含当前所有知识条目的完整状态（内容、分类、有效性分数、状态、向量嵌入）
2. WHEN 系统执行批量有效性分数更新（处理反馈后）前，THE Knowledge_Base SHALL 自动创建一个Knowledge_Snapshot
3. WHEN 管理员选择回滚到某个快照时，THE Knowledge_Base SHALL 将所有知识条目恢复到该快照记录的状态，并创建一条新的快照记录标记为"rollback"
4. WHEN 管理员选择导出知识库时，THE Knowledge_Base SHALL 将所有active状态的知识条目序列化为JSON文件，包含内容、分类、有效性分数和来源信息
5. WHEN 管理员导入外部知识库JSON文件时，THE Knowledge_Base SHALL 将导入的知识条目合并到当前知识库，重复内容（基于语义相似度>0.95判定）自动跳过，新条目重新生成向量嵌入
6. WHEN 序列化Knowledge_Snapshot为JSON格式后再反序列化时，THE Knowledge_Base SHALL 产生与原始快照等价的结果

### 需求 25：知识库统计与Embedding提供商适配

**用户故事：** 作为系统管理员，我希望能查看知识库健康状况并灵活配置向量嵌入服务，以便监控知识库演化并适配不同的AI服务提供商。

#### 验收标准

1. WHEN 管理员访问知识库仪表盘时，THE Knowledge_Base SHALL 展示以下统计数据：知识条目总数、各分类条目数量、各状态条目数量、平均有效性分数、有效性分数分布直方图
2. WHEN 管理员查看知识条目有效性分数变化趋势时，THE Knowledge_Base SHALL 展示指定条目在过去N次更新中的分数变化折线图
3. WHEN 管理员查看知识库演化报告时，THE Knowledge_Base SHALL 展示最近一段时间内新增条目数量、弃用条目数量、分数上升最多和下降最多的条目列表
4. THE Embedding_Provider SHALL 提供统一的接口用于生成向量嵌入，支持通过环境变量配置API密钥和端点地址
5. WHEN 配置Embedding提供商时，THE Embedding_Provider SHALL 支持通过配置切换不同的嵌入模型（如OpenAI text-embedding-3-small、本地模型等）
6. IF Embedding_Provider调用失败，THEN THE Knowledge_Base SHALL 记录错误日志，对于新建知识条目标记为"embedding_pending"状态，待服务恢复后重新生成嵌入

### 需求 26：知识库REST API

**用户故事：** 作为前端开发者，我希望通过REST API访问知识库功能，以便在前端界面中集成知识库管理和可视化功能。

#### 验收标准

1. THE Knowledge_Base SHALL 提供以下REST API端点：知识条目CRUD（创建、读取、更新、弃用）、知识条目列表查询（支持分页、筛选、排序）、知识导入触发（文章提取、剧本分析）、快照管理（创建、列表、回滚）、导出导入、统计数据查询、权重更新历史查询、语义搜索（输入文本返回相似知识条目）
2. WHEN API接收到无效请求参数时，THE Knowledge_Base SHALL 返回包含具体错误字段和错误描述的结构化错误响应
3. WHEN API请求需要管理员权限时，THE Knowledge_Base SHALL 验证请求者的认证token和管理员角色，未授权请求返回403错误

### 需求 27：AI预生成访谈系统

**用户故事：** 作为剧本创作者，我希望在生成剧本前与AI进行交互式访谈，以便AI更精准地理解我的创作意图，生成更贴合需求的剧本。

#### 验收标准

1. WHEN 用户选择"开始访谈"后，THE Pre_Generation_Interview SHALL 基于当前Config参数初始化一个Interview_Session，并向用户提出第一个访谈问题
2. WHEN 用户回答一个访谈问题后，THE Pre_Generation_Interview SHALL 调用LLM_Adapter分析用户回答，动态生成下一个最相关的追问问题，而非按固定顺序提问
3. WHEN Pre_Generation_Interview生成追问问题时，THE Pre_Generation_Interview SHALL 覆盖以下核心维度：叙事风格偏好、情感基调、推理复杂度期望、角色关系深度、玩家互动模式、特殊机制需求
4. WHEN 访谈进行到用户选择结束或达到最大问题数（默认10个）时，THE Pre_Generation_Interview SHALL 调用LLM_Adapter将所有问答对汇总为结构化的访谈摘要
5. WHEN 访谈摘要生成后，THE Pre_Generation_Interview SHALL 将摘要展示给用户确认，用户可修改后确认
6. WHEN 用户确认访谈摘要后，THE Pre_Generation_Interview SHALL 将摘要作为附加上下文传递给Generator，与Config参数一起用于剧本生成
7. WHEN Generator接收到访谈摘要时，THE Generator SHALL 将访谈摘要嵌入LLM提示词中，优先级高于通用知识条目
8. IF LLM_Adapter在访谈过程中调用失败，THEN THE Pre_Generation_Interview SHALL 提供预设的备选问题列表供用户选择回答，不中断访谈流程
9. WHEN 访谈完成后，THE Pre_Generation_Interview SHALL 将完整的Interview_Session（问答对和摘要）持久化存储，关联到后续生成的Script

### 需求 28：访谈模板知识化管理

**用户故事：** 作为系统管理员，我希望访谈模板本身也是知识库的一部分，能通过反馈持续优化访谈质量。

#### 验收标准

1. THE Interview_Template SHALL 作为Knowledge_Entry存储在Knowledge_Base中，知识分类为"prompt_templates"，包含访谈问题模板、分支逻辑和适用场景描述
2. WHEN 系统初始化时，THE Knowledge_Base SHALL 导入默认的Interview_Template种子数据，覆盖不同游戏类型（本格、新本格、变格）的访谈模板
3. WHEN Pre_Generation_Interview初始化访谈时，THE Pre_Generation_Interview SHALL 通过RAG从Knowledge_Base中检索与当前Config最匹配的Interview_Template
4. WHEN 基于某Interview_Template生成的剧本收到玩家反馈后，THE Knowledge_Base SHALL 通过Feedback_Knowledge_Link将反馈关联到该Interview_Template，更新其有效性分数
5. WHEN 管理员创建或编辑Interview_Template时，THE Knowledge_Base SHALL 校验模板格式包含必要字段：初始问题、核心维度覆盖、最大问题数、摘要生成提示词
6. WHEN 序列化Interview_Template为JSON格式后再反序列化时，THE Knowledge_Base SHALL 产生与原始模板等价的结果

### 需求 29：提示词模板管理与版本化

**用户故事：** 作为系统管理员，我希望能管理和版本化所有LLM提示词模板，以便追踪提示词变更历史并支持A/B测试。

#### 验收标准

1. THE Prompt_Template SHALL 支持以下模板类别：剧本生成主模板、角色生成模板、线索生成模板、分支结构生成模板、访谈问题生成模板、知识提炼模板、反馈分析模板
2. WHEN 创建Prompt_Template时，THE System SHALL 为模板分配唯一标识符和初始版本号（v1），支持变量插值语法（如 {{playerCount}}、{{gameType}}）
3. WHEN 更新Prompt_Template时，THE System SHALL 创建新的Prompt_Version记录，版本号自动递增，保留所有历史版本
4. WHEN Generator使用Prompt_Template生成内容时，THE Generator SHALL 记录使用的模板ID和版本号，关联到生成的Script
5. WHEN 管理员选择回滚Prompt_Template到历史版本时，THE System SHALL 将指定历史版本设为当前活跃版本，不删除任何版本记录
6. WHEN 渲染Prompt_Template时，THE System SHALL 将模板中的变量占位符替换为实际参数值，未提供的可选变量使用默认值，缺少必填变量返回错误
7. WHEN 序列化Prompt_Template及其版本历史为JSON后再反序列化时，THE System SHALL 产生与原始数据等价的结果

### 需求 30：少样本示例与推理链管理

**用户故事：** 作为系统管理员，我希望能管理少样本示例和推理链模式，以便通过高质量示例引导LLM生成更好的剧本内容。

#### 验收标准

1. THE Few_Shot_Example SHALL 包含以下字段：唯一标识符、类别（与Prompt_Template类别对应）、适用游戏类型列表、输入描述、期望输出、质量评分（0.0-1.0）、状态（active/deprecated）
2. WHEN Generator组装LLM提示词时，THE Generator SHALL 从Few_Shot_Example中选取与当前生成任务类别和游戏类型匹配的示例，按质量评分降序选取前K个（K由配置决定，默认3个）
3. WHEN 玩家反馈关联到使用了特定Few_Shot_Example的剧本时，THE Knowledge_Base SHALL 根据反馈更新该示例的质量评分，更新逻辑与知识条目有效性分数一致
4. THE Reasoning_Chain SHALL 包含以下字段：唯一标识符、名称、适用场景描述、推理步骤列表（每步包含步骤描述和期望输出格式）、适用游戏类型列表
5. WHEN Generator生成复杂剧本逻辑（分支结构、多结局触发条件、线索链推理）时，THE Generator SHALL 从Reasoning_Chain中检索匹配的推理链模式，嵌入提示词引导LLM进行链式思维推理
6. WHEN 序列化Few_Shot_Example为JSON后再反序列化时，THE System SHALL 产生与原始对象等价的结果
7. WHEN 序列化Reasoning_Chain为JSON后再反序列化时，THE System SHALL 产生与原始对象等价的结果

### 需求 31：Token用量追踪与成本优化

**用户故事：** 作为系统管理员，我希望能追踪所有LLM调用的token用量和成本，以便监控资源消耗并优化使用效率。

#### 验收标准

1. WHEN LLM_Adapter完成一次API调用后，THE System SHALL 创建一条Token_Usage_Record，记录：调用时间、调用类型（生成/访谈/提炼/分析）、使用的模型名称、提示词token数、完成token数、总token数、响应时间（毫秒）、关联的Script ID或Interview_Session ID
2. WHEN 管理员查看Token用量仪表盘时，THE System SHALL 展示以下统计数据：总token消耗、按调用类型分类的token消耗、按时间段（日/周/月）的消耗趋势、平均每次剧本生成的token消耗
3. WHEN 管理员设置token用量预警阈值后，THE System SHALL 在单日token消耗超过阈值时记录警告日志
4. WHEN Generator组装LLM提示词时，THE Generator SHALL 根据LLM上下文窗口大小动态调整知识条目数量和少样本示例数量，确保总token数不超过模型限制的80%

### 需求 32：模型评估与A/B测试

**用户故事：** 作为系统管理员，我希望能对不同的LLM模型或提示词版本进行A/B测试，以便基于数据选择最优配置。

#### 验收标准

1. WHEN 管理员创建AB_Test时，THE System SHALL 接受两个变体配置（变体A和变体B），每个变体指定模型名称和/或Prompt_Template版本，以及流量分配比例（默认50/50）
2. WHILE AB_Test处于运行状态时，THE Generator SHALL 根据流量分配比例将剧本生成请求随机分配到变体A或变体B
3. WHEN 使用AB_Test变体生成的剧本收到玩家反馈后，THE System SHALL 将反馈数据关联到对应的AB_Test变体
4. WHEN 管理员查看AB_Test结果时，THE System SHALL 展示每个变体的样本数量、各维度平均评分、综合反馈分数和统计显著性指标
5. WHEN 管理员结束AB_Test并选择优胜变体时，THE System SHALL 将优胜变体的配置设为系统默认配置，并记录测试结论
6. WHEN AB_Test运行期间某变体的样本数量不足（少于5个反馈）时，THE System SHALL 在结果页面标注"样本不足，结论待定"


### 需求 33：设计师身份与双身份系统

**用户故事：** 作为用户，我希望拥有独立的设计师身份和玩家身份，以便分别追踪我的创作成就和游玩成就，两个身份独立升级。

#### 验收标准

1. THE Player_Account SHALL 同时包含玩家身份（Player_Level）和设计师身份（Designer_Identity），两者拥有独立的等级和经验值
2. WHEN 用户首次创建剧本时，THE System SHALL 自动初始化该用户的Designer_Identity，设计师等级为1，经验值为0
3. WHEN 用户完成一局Game_Session后，THE System SHALL 为该用户的玩家身份增加游玩经验值（基础10XP + 评价加成）
4. WHEN 用户的玩家经验值达到升级阈值时，THE System SHALL 自动提升玩家等级（等级N升级所需XP = N × 100）
5. WHEN 用户的设计师经验值达到升级阈值时，THE System SHALL 自动提升设计师等级（等级N升级所需XP = N × 150）
6. THE Player_Account SHALL 在个人主页分别展示玩家等级信息（等级、经验值、游玩次数）和设计师等级信息（等级、经验值、创建剧本数、剧本被游玩总次数）

### 需求 34：技能牌系统

**用户故事：** 作为设计师，我希望Skill以可选牌的形式呈现，以便我在创建剧本时通过选牌来定制生成效果，让剧本更具个性。

#### 验收标准

1. THE Skill_Card SHALL 包含以下属性：唯一标识符、名称、描述、类别（SkillCategory）、适用游戏类型列表、牌等级（Card_Tier：basic/advanced/expert/legendary）、点数消耗（Card_Point_Cost）、效果内容（Prompt片段）、解锁所需设计师等级
2. WHEN 系统初始化时，THE System SHALL 将所有现有Skill模板转换为Skill_Card，基础类别的牌设为basic等级，高级类别的牌设为对应更高等级
3. WHEN 设计师查看可用技能牌时，THE System SHALL 仅展示该设计师当前等级已解锁的牌，未解锁的牌以灰色展示并标注解锁所需等级
4. THE Skill_Card SHALL 按牌等级分配点数消耗：basic牌消耗1-2点，advanced牌消耗3-4点，expert牌消耗5-7点，legendary牌消耗8-10点
5. WHEN 序列化Skill_Card为JSON格式后再反序列化时，THE System SHALL 产生与原始对象等价的结果（往返一致性）
6. THE Skill_Card SHALL 包含以下特色牌示例：打破第四堵墙（expert）、神话设定背景（advanced）、多杀手设定（expert）、时间循环（legendary）、不可靠叙述者（advanced）、密室逃脱机制（basic）

### 需求 35：设计点数预算系统

**用户故事：** 作为设计师，我希望每次设计剧本时有点数预算限制，以便通过等级提升获得更多点数来选择更强力的牌组合。

#### 验收标准

1. WHEN 设计师开始创建剧本时，THE System SHALL 根据设计师等级计算Design_Point_Budget：等级1=10点，等级2=13点，等级3=16点，等级4=20点，等级5=25点，之后每级+3点
2. WHEN 设计师选择技能牌时，THE System SHALL 实时显示已消耗点数和剩余点数
3. WHEN 设计师选择的技能牌总点数超过Design_Point_Budget时，THE System SHALL 阻止添加该牌并提示点数不足
4. WHEN 设计师确认选牌完成后，THE System SHALL 将选中的Skill_Card列表传递给Generator，作为生成上下文的一部分
5. WHEN Generator接收到Skill_Card列表时，THE Generator SHALL 将每张牌的效果内容嵌入LLM提示词中，替代原有的按游戏类型自动选择Skill模板的逻辑
6. IF 设计师未选择任何技能牌，THEN THE Generator SHALL 回退到按游戏类型自动选择基础Skill模板的默认行为

### 需求 36：设计师牌库与自定义牌组

**用户故事：** 作为设计师，我希望能管理自己的牌库并组合自定义牌组，以便快速复用常用的牌组合来设计不同风格的剧本。

#### 验收标准

1. THE Designer_Deck SHALL 包含该设计师当前等级已解锁的所有Skill_Card
2. WHEN 设计师等级提升时，THE System SHALL 自动将新解锁等级对应的Skill_Card添加到Designer_Deck中，并通知设计师
3. WHEN 设计师创建Custom_Deck时，THE System SHALL 允许设计师从Designer_Deck中选择牌组成自定义牌组，并为牌组命名
4. THE Custom_Deck SHALL 不限制牌的总点数（点数限制仅在实际创建剧本时生效）
5. WHEN 设计师查看Custom_Deck列表时，THE System SHALL 展示每个牌组的名称、包含的牌数量和总点数消耗
6. THE System SHALL 允许设计师创建、编辑和删除Custom_Deck，每个设计师最多拥有10个自定义牌组
7. WHEN 设计师在创建剧本时选择使用某个Custom_Deck时，THE System SHALL 自动加载该牌组中的所有牌，设计师可在此基础上增减牌

### 需求 37：设计师奖励与经验值系统

**用户故事：** 作为设计师，我希望当我创建的剧本被其他玩家游玩时能获得奖励，以便激励我持续创作高质量的剧本。

#### 验收标准

1. WHEN 设计师创建的剧本被其他玩家完成一局Game_Session后，THE System SHALL 为该设计师增加设计师经验值（基础5XP）
2. WHEN 设计师创建的剧本收到玩家评价且整体满意度≥8分时，THE System SHALL 为该设计师额外增加经验值加成（额外10XP）
3. WHEN 设计师创建的剧本累计被游玩次数达到里程碑（10次、25次、50次、100次）时，THE System SHALL 为该设计师解锁对应的设计师成就徽章
4. WHEN 设计师等级提升时，THE System SHALL 通知设计师新解锁的牌和新增的点数预算
5. THE System SHALL 在设计师个人页面展示设计师统计数据：创建剧本数、剧本被游玩总次数、平均评分、设计师等级和经验值进度
6. WHEN 计算设计师经验值加成时，THE System SHALL 根据评价的加权平均分（使用Player_Rating_Weight）计算，而非简单平均分
