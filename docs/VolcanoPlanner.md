## VolcanoPlanner

findBestExp() 是整个搜索引擎的入口，在这个搜索引擎中，搜索过程分成了好几个阶段，每个阶段都有自己的 rule list。
每个阶段的搜索算法都是复用的同一段代码逻辑

RuleQueue：VolcanoPlanner 专用，需要关注的是 phaseRuleMapping 和 matchListMap。
1. phaseRuleMapping 的初始化由 RuleQueue 的构造函数完成，这里会回调 VolcanoPlanner，辅助完成初始化工作。
  经过初始化后，OPTIMIZE 阶段的 rule 被注册为了 ALL_RULES，而其他阶段被注册为了一个无意义的 "xxx"。
2. matchListMap 的初始化也是由 RuleQueue 的构造函数完成，初始时，每个 phase 的 match list 为空

在 findBestExp() 中，不断地调用 ruleQueue.popMatch(phase) 函数得到一个当前 phase 中 importance 值最高的 VolcanoRuleMatch，
然后调用这个 match 的 onMatch 函数，完成 rule 的 apply。那么问题来了：

- [x] popMatch() 函数需要从 matchListMap 中获取当前 phase 的 matchList，这个 matchList 刚开始的时候是空的，
  里面的 ruleMatch 是什么时候添加进去的？
  
    搜索完所有对 matchListMap 的引用后，这个 Map 只在 addMatch() 函数中被写入了。在这里，会把当前这个 VolcanoRuleMatch
    添加到所有阶段的 matchList 中，如果当前阶段所注册的 ruleList（通过 phaseRuleMapping 获取）中没有这个 match 上的 rule，则跳过。
    
这里有一个问题留着以后讨论：
- [ ] **为什么每个阶段都要去注册一下当前这个 VolcanoRuleMatch？**
    
我们先来看这个问题：
- [ ] addMatch() 什么时候会被调用？
