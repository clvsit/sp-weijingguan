# 微景观小程序

编写人 | 编写日期 | 版本号 | 备注
---|---|---|---
陈煦 | 18-05-22 | 1.0 | 总结第一版所要实现的功能以及技术难点

## 百科模块
> 百科模块相当于一本电子版的植物百科全书，主要用于展示植物相关的百科知识，包括植物的科属、习性以及养护小知识。

#### 今日植物（可选）
> 微景观小程序每日推荐一颗植物。用户也可以点击“往期植物”进入到“植物历”页面，在该页面可以查看往日推荐的所有植物。

【重要性】：★★☆

【实现要求】：
1. 数据支持：项目必须有大量的植物百科数据。
2. 技术支持：要实现该功能必须要前后端同时克服一定的技术难点。

【技术难点 1】：<br/>
如何随机挑选植物，并且要确保所挑选的植物不能重复？

【解决方案 1】：
1. 方案一：在植物表（存放植物数据的表）中新增一列 isListed。1 表示已经展示，0 表示还未展示。在随机挑选植物时，首先根据 where isListed = 0 进行数据过滤，然后再采用算法去选择植物。挑选好植物后，将该植物对应的 isListed 字段值修改为 1。
2. 方案二：新建一张表，可以命名为 today。该表的作用是用来存放已经展示的植物数据。那么如何获取未展示的植物数据呢？可以通过子查询的方式来实现，思路是只要植物表中的植物数据不在 today 表中（where not in (select id from today)）。这种方式的优势在于**获取往期植物**时更易处理。我们可以在 today 表中新增一个日期，比如说该植物是在18年5月27日加入到 today 表中。在获取数据时可以直接按照该日期降序排列，从而实现植物历往期植物的效果。

【技术难点 2】：<br/>
如何每日自动挑选随机植物？

【解决方案 2】：
1. 方案一：专门的负责人员，每日登录后台管理系统进行植物挑选。
2. 方案二：后台使用定时任务，于每日凌晨启动，将挑选的植物插入到 today 表中。

【实现流程】：
![今日植物功能实现流程](https://i.imgur.com/pu61l9p.jpg)

个人建议：方案一相比方案二要更稳定以及快捷。
1. 方案一请求中不需要携带请求日期，比方案二减少了交互上的开销；
2. 方案二存在一定的隐患，例如凌晨启动定时任务的同时，用户查看今日植物，并向后台发起了请求（携带日期）。然而此时定时任务还未完成，也就是说数据表中不存在这条数据，因此 SQL 语句会执行失败。若后台语言编写时未考虑到这种情况，很容易导致本次请求失败且无任何提示，对用户体验不友好。

#### 热门搜索（可选）
> 小程序会记录下用户每次搜索的内容，并将其存储到相应的数据库中。通过一定的算法推算出目前搜索次数最多的植物，将这些植物呈现在页面上。

【重要性】：★★☆

【实现要求】：
1. 数据存储：当用户群体增加时，搜索记录数也会随之增加。因此对数据量要做一定的限制。但目前的限制也只是对返回给用户的数据进行限制，比如说最多显示最近的十条搜索记录，第十一条仍然保存在数据库中。
2. 模糊查询：对用户输入的内容进行模糊查询。但需要明确所谓的模糊是针对什么内容，植物的名称还是植物所属的种类？

【技术难点 1】：
- 如何根据用户输入的模糊查询来确定热门搜索的植物？这个问法不太妥当，用一个例子来说明：

    【举例】：当用户输入“花”时，我们会将所有与花有关的植物返回给页面。此时，我们记录的搜索内容是“花”而不是具体的某一颗植物。如果最终“花”搜索的次数最多，那么就会存在一个问题，实际上并没有“花”这一种植物，而我们的热门搜索是需要呈现具体的一颗植物。

【解决方案 1】：
1. 方案一：我们将用户检索的记录都存放到一张表中（暂且命名为 record），然后通过 select \*, count(\*) as 'count' from record group by xxx 语句获取搜索次数最多的内容。再将这些内容作为检索条件，从植物表中获取对应的植物。不过该方案也有个极大的缺陷。如果用户输入最多的内容检索不出植物呢？比如说“会说人话的花”！emmmmm，这个应该是检索不出结果的，到时候百科首页热门搜索区域就会有空白。
2. 方案二：该做法中规中矩，在每次用户搜索完成时将相关的植物搜索次数 + 1。获取热门搜索植物的时候，只需要根据搜索次数进行降序排列，从中获取指定数量的植物即可。该方案确保获取热门搜索操作一定能够获取到植物数据。但效率是非常低的，尤其在每次将相关植物的搜索次数 + 1时。
3. 方案三：将方案一和方案二整合之后想到的一个折中方案。整体流程同方案一，但不同的是，我们在用户搜索之后对用户本次的搜索进行一次评定，来判断该搜索内容是否为一个有效搜索。简单的判断标准可以是该检索内容是否有返回内容，如果有，则标志为有效，反之，则无效。那么，像“会说人话的花”这一类检索内容就不能作为有效的检索。在执行 select ... from ... 语句时只获取有效的检索，比如说新增 where isValided = 1（有效）条件来剔除无效的检索。

【技术难点 2】：<br/>
模糊查询的依据是什么？

【解答 2】：<br/>
目前来看，简单的处理方式是依据植物的名称。比如说我们在搜索输入框内输入“花”，可以检索出“迎春花”。因为其名称中带有“花”字。后期可以考虑扩大模糊查询的范围，例如植物的科属等。

![热门搜索功能实现流程](https://i.imgur.com/uFXn9n2.jpg)

#### 猜你喜欢（可选）
> 微景观小程序可通过一定的算法推演当前用户所喜欢的植物，并将其推送给当前用户。

技术实现较为困难，且没有一定的数据支持，开发阶段会遇到不少的阻力。建议将该模块推后到其他版本，在有一定的用户数据之后可以尝试去实现。

## 智能设备
> 小程序的核心，用以展示检测仪获取的数据，同时提供合理的养护建议。

#### 数据报告（可选）
> 为用户提供每日植物生长情况报告，包括检测次数、数据折线图以及计算后各项数据的平均值，以及总评内容。

【重要性】：★★☆

【实现要求】：
1. 报告生成：用户点击数据报告即可自动生成一份关于今日植物生长情况的报告。
2. 智能评语：系统根据检测所得的数据智能生成相应的评语。

【技术难点】：<br/>
虽然在实现要求中描述了报告自动生成和智能评语什么的，但真正实现起来无非是基础操作的组合，并没有涉及到新且难的技术。在下方的实现过程中大致描述报告生成以及智能评语的实现方式。

【实现过程】：

![数据报告实现流程](https://i.imgur.com/tgwO18C.jpg)

前端向后台发起请求，后台接受到请求后获取当前系统时间，然后根据系统时间（例如 2018-06-13）到数据库中检索今日的数据。再对已获取的数据求平均值，接着依据求得的平均值到数据库中获取相应的评语。最后返回最近 X 条数据 + 各项数据的平均值 + 评语。

【问】：为什么只返回最近 X 条数据呢？<br/>
【答】：因为手机屏幕的分辨率较低，如果用户一日之内检测次数过多，举个极端的例子，检测了二十次，我们是不可能生成一个拥有 20 个数据点的折线图。至于为什么是 X，还有待于去实践，以及根据样式效果去判断显示多少数量的数据，会使得界面效果更好。

【问】：如何根据平均值获取评语呢？<br/>
【答】：我们可以将各项数据分级，例如设定一条标准线，将高于该标准线的数据值标记为高，低于则标记为低。当然，如果要评级更为精确，可以增加更多的标准线。通过这种方式，我们就能够将平均数据依次对应一个评级，最后就能够将实际的数值转化为“高低低高”这种形式，2 的 4 次方为 16，也就是说能够将所有可能的数据对应 16 条评语中的 1 条。当然也不一定非得有 16 条评语，例如“低低低低”和“低低低高”可以对应一条相同的评语。

#### 历史记录
> 记录一周乃至一月的植物生长情况，并以折线图的方式呈现数据波动。同时在折线图下方显示每日或每周的数据报告。

【重要性】：★★★★☆

【实现要求】：
1. 周历史轨迹：显示最近一周的检测数据。
2. 月历史轨迹：显示最近一月的检测数据。

【技术难点-周历史轨迹】：
1. 需要和数据报告配合：前面所讲的数据报告是指用户主动点击之后生成的报告。该处的数据报告是指每日结束之后，系统自动（真正意义上的自动）生成的报告，因此需要运用定时任务。

【实现过程-周历史轨迹】：<br/>
其实现过程同**数据报告的实现过程**，也是前端向后台发起请求，后台收到请求开始获取系统时间，然后获取这一周的所有数据报告，再根据这些数据报告，计算出周平均数据。最后将所有数据报告的数值 + 周平均数据返回给前端。

【技术难点-月历史轨迹】：
1. 以周历史作为基本单位：手机屏幕的尺寸有限，不可能在屏幕上显示一个月每天的数据。虽然可以实现，但这首先会影响页面布局，其次对性能的影响也很大。而且做出来意义也不大，还不如做一个检索历史记录有意义。

【实现过程-月历史轨迹】：<br/>
其实现过程同**周历史轨迹的实现过程**，不同的是每周执行一次定时任务，除此之外还需要有一个单独的数据表，用以存放周数据报告（数据报告和周历史轨迹使用的都是日数据报告）。

【问】：为什么历史记录里面没有评语？<br/>
【答】：首先检测仪本身存在一定的偏差，平均取值也不能准确地表达真实的环境情况，因此评语无法准确地描述此刻植物的真实生长情况。换言之，在历史记录中写评语价值不大。

#### 养护小贴士（可选）
> 根据当前植物的生长状况提供相应的养护建议。

这部分我建议在后续版本中再推出，尤其当我们满足以下两个条件时：
- 数据方面：积累了一部分社区帖子，且这些帖子涉及到植物养护方面的知识。
- 技术方面：开发人员掌握数据挖掘相关的技术，能够根据植物生长状况去获取有关的帖子。

【建议】：第一版，可以将养护小贴士放到数据报告中，当然也可以直接推迟到后续版本再出，这取决于我们现阶段能够想出或找到足够多的养护知识。