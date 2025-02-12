---
title: 成为神话前的程序 ——读《人月神话》
date: 2024-09-05 19:10:48
updated: 2024-09-05 19:10:48
tags: [书评]
categories: [杂谈]
thumbnail: /images/others/mmm.webp
cover: /images/others/mmm.webp
toc: true
---

<p style="text-align: right;"><i>——接着，每个编程人员准备相信这样的神话。</i></p>

管理软件项目开发往往比开发本身更为冗杂，具体了解在一个团队中如何进行开发任务是非常重要的，我想这也就是我阅读这本上世纪的书籍还能感受到有所受益的原因。<!-- more -->比如，书的开篇指出了程序要呈现给其他人的姿态，如果要变成编程产品，即可以被任何人运行、测试、修复和扩展的程序，可以运行在多种操作系统平台上，供多套数据使用则程序必须按照普遍认可的风格来编写，且需要对程序进行彻底测试，还需要有完备的文档，每个人都可以加以使用、修复和扩展。即程序本身的开发只占相同功能的编程产品的1/3。如果要变成编程系统的一部分（即一个大的程序中的一个模块），则需要在各种规范下编写精确定义的接口，符合预留的资源限制，同其他单元进行组合使用，同样也会变成独立程序的三倍。

但只有编程系统产品才是面向用户的，真正有用的产品，是大多数系统开发的目标。也就是说，其不仅作为一个部分存在，更要有独立迭代更新的条件。我认为这是为了方便将一个大产品目标的工作分割到各个团队中所需要的步骤，通过一系列的规范使其不仅能被用还能被理解。这也就牵扯到一点，就是对项目所需时间的错误预估，而且这一灾难性的影响是非常普遍的。

产生的原因在于我们的过度自信项目能正常运作、假设人和月可以互换、对自己的估算缺乏信心、缺少跟踪和监督和使用错误的添加人力的手段救急。在这里提到了本书的书名，也是要表达的一个重要观点。人月是指一种计量工程进度的工作量单位，它暗示着人员数量和时间是可以互相替换的，例如十人花十个月完成的程序，替换成一百个人用一个月完成，但这在系统编程中是不可能的事情，这种想当然的意见往往与事实天差地别，因此被称为“神话”。因为添加人数带来的交流等方面的问题会影响工程时间，沟通、交流的工作量非常大，它很快会消耗任务分解所节省下来的个人时间。从而，添加更多的人手，实际上是延长了，而不是缩短了时间进度。这使我联想到并行计算中的通信时间，当过多地设置并行的线程或者进程数时，在各线程或者进程间的数据通信就会影响计算所需的时间，这会导致有些任务在划分到一定并行数后计算时间不会减少反而增加，这点和“人月”是相似的，不能简单用并行数来替换时间。

另一方面就是测试所需的时间实际上比预想的时间多，而且非常容易错误估计。以及如果只是简单向项目中增加人手来救急，考虑到需要经过一段时间进行培训才能够真正接手项目进行工作，还有需要对项目重新划分等等因素的影响，向进度落后的项目中增加人手，只会使进度更加落后——也是Brooks法则。这就是除去了神话色彩的人月。不仅大型项目是如此，个人项目也会容易受到各种因素的干扰，致使最后推进速度远低于预期。

如何在有意义的时间进度内创建大型的系统？对于效率和概念的完整性来说，最好由少数干练的人员来设计和开发，而对于大型系统，则需要大量的人手，以使产品能在时间上满足要求。本书提出了一种“外科手术团队”式划分，即大型项目的每一个部分由一个团队解决，但是该队伍以类似外科手术的方式组建，而并非一拥而上。如果考虑所有可能想到的工作，这样的队伍应该由以下成员组成：外科医生（首席程序员，定义和设计程序，编写测试和写文档）、副手（外科医生的后备，保险机制）、管理员（控制财务、人员、工作地点安排和机器的专业管理人员）、编辑（根据外科医生的草稿书写文档）、两个秘书（管理员和编辑的副手，项目的写作和非产品文件）、程序职员（维护编程产品库中所有团队的技术记录）、工具维护人员（检查他的外科医生所需要的工具）、测试人员（写测试的）、语言专家（精通编程语言的）。这就是一个十人团队，“外科手术刀”团队式的任务划分。

程序的整个创造性活动包括了三个独立的阶段：体系结构（architecture）、设计实现（implementation）、物理实现（realization）。在实际情况中，它们往往可以同时开始和并发地进行。例如，在计算机的设计中，一旦设计实现人员有了对手册的模糊设想，对技术有了相对清晰的构思以及拥有了定义良好的成本和目标时，工作就可以开始了。他可以开始设计数据流、控制序列、大体的系统划分等等。在物理实现的级别，电路、板卡、线缆、机箱、电源和内存必须分别设计、细化和编制文档。这项工作与体系结构及设计实现并行进行。在编程系统的开发中，必须设定良好定义的时间和空间目标，了解产品运行的平台配置。接着，他可以开始设计模块的边界、表结构、算法以及所有的工具。在物理实现的级别，也有很多可以着手的工作。编程也是一项技术，如果是新型的机器，则在库的调整、系统管理以及搜索和排序算法上，有许多事情需要处理。总之，在概念完整前就可以先投入工作，因为总有要做的事情，就好比开始工作前要打开电脑，一些有规范的先前步骤，或者熟练后所意识到的可能要做的事情，都可以在完整概念形成，统一划分前先着手去做。

大型编程项目中的交流是很重要的，非正式途径（例如电话和通信软件），会议和工作手册都是能让右手知道左手在做什么的办法。。项目工作手册不是独立的一篇文档，它是对项目必须产出的一系列文档进行组织的一种结构。项目所有的文档都必须是该结构的一部分。这包括目的、外部规格说明、接口说明、技术标准、内部说明和管理备忘录。技术说明几乎是必不可少的。如果某人就硬件和软件的某部分，去查看一系列相关的用户手册。他发现的不仅仅是思路，而且还有能追溯到最早备忘录的许多文字和章节，这些备忘录对产品提出建议或者解释设计。对于技术作者而言，文章的剪裁粘贴与钢笔一样有用。
大型编程项目的组织架构是树状组织架构，树状编程队伍，以及要使它行之有效，每棵子树所必须具备的基本要素。它们是： 1. 任务（a mission） 2. 产品负责人（a producer）（组建团队，划分工作及制订进度表。）3. 技术主管和结构师（a technical director or architect）（对设计进行构思，识别系统的子部分，指明从外部看上去的样子，勾画它的内部结构。）4. 进度（a schedule） 5. 人力的划分（a division of labor） 6. 各部分之间的接口定义（interface definitions among the parts）。

巴比伦塔可能是第一个工程上的彻底失败，但它不是最后一个。交流和交流的结果——组织，是成功的关键。交流和组织的技能需要管理者仔细考虑，相关经验的积累和能力的提高同软件技术本身一样重要。
任何管理任务的关注焦点都是时间、地点、人物、做什么、资金。

首先，项目的关键问题是沟通，个性化的工具妨碍——而不是促进沟通。其次，当机器和语言发生变化时，技术也会随之变化，所有工具的生命周期是很短的。毫无疑问，开发和维护公共的通用编程工具的效率更高。
如何根据一个严格的进度表来控制项目？一个步骤是制订进度表。进度表上的每一件事，被称为“里程碑”，它们都有一个日期，在很大程度上依赖以往的经验。

寻找银弹是为了解决软件项目可能变成一个落后进度、超出预算、存在大量缺陷的怪物的问题，寻求求一种可以使软件成本像计算机硬件成本一样降低的尚方宝剑。但《没有银弹》中声称和断定，在近十年内，没有任何单独的软件工程进展可以使软件生产率有数量级的提高（引自1986年的版本）。软件开发中困难的部分是规格化、设计和测试这些概念上的结构，而不是对概念进行表达和对实现逼真程度进行验证。如果这是事实，那么软件开发总是非常困难的。天生就没有银弹。要么防止项目变成吞噬一切的狼人，要么找到银弹解决狼人，很明显我们只能做到前者。通过对时间的正确预计，和完成里程碑式的任务，才能防止项目变成狼人。

作为结束语，摘自最后一段话：软件工程的焦油坑在将来很长一段时间内会继续地使人们举步维艰，无法自拔。软件系统可能是人类创造中最错综复杂的事物，只能期待人们在力所能及的或者刚刚超越力所能及的范围内进行探索和尝试。这个复杂的行业需要：进行持续的发展；学习使用更大的要素来开发；新工具的最佳使用；经论证的管理方法的最佳应用；良好判断的自由发挥；以及能够使我们认识到自己不足和容易犯错的——上帝所赐予的谦卑。
