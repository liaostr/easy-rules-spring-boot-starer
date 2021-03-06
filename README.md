#  包装easy-rules 的spring boot starter

> 当前主要包装spel，对于mvel 后期添加，对于rules配置文件格式支持json&&yaml

## 使用说明

*  添加依赖

>  oss.sonatype.org SNAPSHOT 已经可用 支持easy-rules 4.0

```code
<dependency>
    <groupId>com.github.rongfengliang</groupId>
    <artifactId>easy-rules-spring-boot-starter</artifactId>
    <version>2.0-SNAPSHOT</version>
</dependency>
```

* 参考demo


* 配置说明

添加规则配置

src/main/resources/application.yaml

```code
easyrules:
  skipOnFirstAppliedRule: false
  skipOnFirstNonTriggeredRule: false
  priorityThreshold: 1000000
  rules:
  - rulesId: "userlogin"
    rulesLocation: "rules-json.json"
    contentType: JSON

```


添加规则文件: rules-json.json

>  注意easy-rules 3.0与4.0有差异，

1.0-SNAPSHOT starter 参考配置

```code
[{
  "name": "1",
  "description": "1",
  "priority": 1,
  "compositeRuleType": "UnitRuleGroup",
  "composingRules": [
    {
      "name": "2",
      "description": "2",
      "condition": "#{#biz.age >= 18}",
      "priority": 2,
      "actions": [
        "#{@myService.setInfo(#biz)}",
        "#{T(com.dalong.easyrulesdemo.demo.UserServiceImpl).doAction4(#biz)}"
      ]
    }
  ]}
]
```

2.0-SNAPSHOT starter 参考配置

```code
[{
  "name": "1",
  "description": "1",
  "priority": 1,
  "compositeRuleType": "UnitRuleGroup",
  "composingRules": [
    {
      "name": "2",
      "description": "2",
      "condition": "#biz.age >18",
      "priority": 2,
      "actions": [
        "@myService.setInfo(#biz)",
        "T(com.dalong.easyrulesv4.UserServiceImpl).doAction4(#biz)"
      ]
    }
  ]}
]
```

## 代码使用说明


1.0-SNAPSHOT 参考使用

```code
@RestController
public class UserApi {

    @Autowired
    Map<String,Rules> configRules;
    @RequestMapping(value = "/", method = RequestMethod.POST)
    public  Object info(@RequestBody User user) throws Exception {
        Rules rules = configRules.get("userlogin");
        Facts facts = new Facts();
        // 生成一个唯一id，方便基于数据id规则流程查询
        user.setUniqueId(UUID.randomUUID().toString());
        FinalRule<User> rule = new FinalRule<User>();
        // rules.register(spELRule);
        rules.register(rule);
        facts.put("biz",user);
        //  默认模式
        // myEngine.fire(rules,facts);
        // 应该使用原型模式
        SpringBeanUtil.getBean("rulesEngine",RulesEngine.class).fire(rules,facts);
        if(rule.isExecuted()){
            User userResult=  rule.getResult();
            System.out.println("result from final ruls"+userResult.toString());
            rules.unregister(rule);
            return userResult;
        }
        else  {
            return  null;
        }
    }

}

```

2.0-SNAPSHOT 参考使用


```code

@RestController
public class UserApi {

    @Autowired
    Map<String,Rules> configRules;
    @RequestMapping(value = "/", method = RequestMethod.POST)
    public  Object info(@RequestBody User user) throws Exception {
        Rules rules = configRules.get("userlogin");
        Facts facts = new Facts();
        // 生成一个唯一id，方便基于数据id规则流程查询
        user.setUniqueId(UUID.randomUUID().toString());
        facts.put("biz",user);
        //  默认模式
        // myEngine.fire(rules,facts);
        // 应该使用原型模式
        SpringBeanUtil.getBean("rulesEngine",RulesEngine.class).fire(rules,facts);
        User userResult=  facts.get("biz");
        System.out.println("result from final ruls"+userResult.toString());
        return userResult;
    }

}
```

## 几个扩展点

* rulelistener

可以添加自己的bean，方便记录信息，比如分析业务规则的执行，每个阶段的数据，计划支持prometheus的metrics
以及rule pipeline

参考实现:
```code
@Component
public class MyRuleListener implements RuleListener {
    Log log = LogFactory.getLog(DefaultRulesListener.class);
    @Override
    public boolean beforeEvaluate(Rule rule, Facts facts) {
        return true;
    }

    @Override
    public void afterEvaluate(Rule rule, Facts facts, boolean b) {
        log.info("-----------------afterEvaluate-----------------");
        log.info("my RulesListener: "+"rule name: "+rule.getName()+"rule desc: "+rule.getDescription()+facts.toString());
    }

    @Override
    public void beforeExecute(Rule rule, Facts facts) {
        log.info("-----------------beforeExecute-----------------");
        log.info("my RulesListener: "+"rule name: "+rule.getName()+"rule desc: "+rule.getDescription()+facts.toString());
    }

    @Override
    public void onSuccess(Rule rule, Facts facts) {
        log.info("-----------------onSuccess-----------------");
        log.info("my RulesListener: "+"rule name: "+rule.getName()+"rule desc: "+rule.getDescription()+facts.toString());
    }

    @Override
    public void onFailure(Rule rule, Facts facts, Exception e) {
        log.info("-----------------onFailure-----------------");
        log.info("my RulesListener: "+"rule name: "+rule.getName()+"rule desc: "+rule.getDescription()+facts.toString());
    }
}

```
* ruleEnginelistener

可以添加自己的bean，方便记录信息，方便分析rulegine的情况计划，计划支持prometheus的metrics

