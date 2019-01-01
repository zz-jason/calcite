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
- [x] addMatch() 什么时候会被调用？

    搜索完所有对这个函数调用后，addMatch() 这个函数只被 DeferringRuleCall 在 onMatch() 的时候被调用了，
    而 DeferringRuleCall 的 onMatch() 函数只可能在某个 sub plan match 上一个 rule 后被调用，发生在 VolcanoRuleCall
    的 matchRecurse() 函数里面。
    
    而 VolcanoRuleCall 的 match 函数只会在 VolcanoPlanner 的 fireRules() 函数中被调用。扫了一圈所有 fireRules() 的函数调用，发现传入的
    deferred 参数的值都是 true
    
    也就是说，当我们 fireRules() 找到匹配的 rule 后，会将当前的 rule 放入到对应阶段的 match list 中。这个 insert 的过程略微曲折：构造一个
    DeferringRuleCall，然后调用他的 onMatch() 函数。
    
    那么现在我们需要重新审视 matchListMap 了：它里面的所有 VolcanoRuleMatch 都是根据当前 RelNode 找到的 rule match，但是这个 rule
    没有在发现 match 后立马 apply，而是放入到了 matchListMap 中。
    
那么另一个问题来了：

- [ ] 什么时候 fireRules()?

    目前只有两个地方会去 fireRules()，分别是 VolcanoPlanner.registerImpl() 和 RelSet.mergeWith()，然后再翻一下调用链会发现，
    所有的线索都指向了 VolcanoPlanner.registerImpl()。它才是一切 fireRules() 的源头。那么 registerImpl() 主要是做什么呢？谁在调用它?
    
    VolcanoPlanner 中有一个 **mapRel2Subset** 用来将 RelNode 映射成 RelSubset，需要关注一下，判断某个 RelNode 是否注册过也是通过这个
    Map 来实现的。在调用 registerImpl() 函数的时候：
    - 需要确保该 RelNode 没有被注册过
    - 需要确保该 RelNode 的 callint convention trait 不为空
    - 需要确保该 RelNode 的 trait 数量和 VolcanoPlanner 中注册的 trait definition 数量相同
    
### RelTraitDef
    
### RelSet/RelSubSet/RelNode

这里总结一下 VolcanoPlanner 的几个 Register 方法以及涉及到的 RelSet/RelSubSet/RelNode 的概念

- **RelNode**:
  1. RelSubSet 是一类特殊的 RelNode，RelSubSet 中的所有 RelNode 具有同样的 `RelTrait`s
  2. Converter 也是一类特殊的 RelNode
  
RelSet 的合并发生在函数

`VolcanoPlanner.register(RelNode rel, RelNode equiv)`:
  - 先找到 `equiv` 所属的 `RelSet`，然后调用 `VolcanoPlanner.registerImpl(rel, set)` 递归的对 rel 中的各个节点完成注册和 rule 的 fire（将 match 的 rule 添加到 rule match queue 里面）
    
