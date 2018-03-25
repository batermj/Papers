# Cobra(白盒代码审计)

## 1. 代码审计

#### 源码审计
> 由于开发人员的技术水平和安全意识各不相同，导致可能开发出一些存在安全漏洞的代码。
> 攻击者可以通过渗透测试来找到这些漏洞，从而导致应用被攻击、服务器被入侵、数据被下载、业务受到影响等等问题。
> "源代码安全审计"是指通过审计发现源代码中的安全隐患和漏洞，而Cobra可将这个流程自动化。

#### 产品定位
如果从漏洞影响的角度讲，可以大致分为：

|类型|影响等级|影响描述|典型例子|典型软件|
|---|---|---|---|---|
|黑盒|100|可直接利用|SQL Injection、XXE、SSRF等|[SQLMap](http://sqlmap.org/)|
|白盒1|80|可能能直接利用或可间接利用或暂时不能利用|某个存在SSRF的函数，但发现时不一定被调用了|[RIPS](https://github.com/ripsscanner/rips)|
|白盒2|20|不能直接利用，但不符合安全规范|使用了因安全问题弃用的函数|[SonarQube](https://sonarqube.com/)|

而Cobra的目前定位就是覆盖白盒1和白盒2两个等级：
1. 能找到代码中所有的漏洞点（这些漏洞能不能直接利用都算作风险，因为这些风险的存在，迟早有可能出现安全事故）。
2. 能够快速应对新型漏洞的扫描。

## 2. 业内调研

目前业内已经有很多款代码审计工具了

| 名称                                       | 是否开源 |缺点|
| ---------------------------------------- | ---- |---|
| [droopescan](https://github.com/droope/droopescan)  | 是    |仅覆盖CMS框架|
| [GrepBugs](https://grepbugs.com/) | 是    |简单的正则匹配，误报较高|
| [Pixy](https://github.com/oliverklee/pixy) | 是    |未商用，不成熟|
| [RIPS](http://rips-scanner.sourceforge.net/) | 是    |仅覆盖PHP|
| [SWAAT](https://www.owasp.org/index.php/Category:OWASP_SWAAT_Project) |是|.NET写的、无法快速扩展新型漏洞|
| [PHP-SAT](http://www.program-transformation.org/PHP/PhpSat) | 是    |仅覆盖PHP、长期不维护、无法快速扩展新型漏洞|
| [Yasca](http://scovetta.github.io/yasca/) | 是    |偏向于代码安全规范的扫描、无法快速扩展新型漏洞|
|[SonarQube](https://sonarqube.com/)|是|偏向于代码安全规范的扫描、规则使用Java代码开发、无法快速增加新型漏洞扫描规则|
|[PreFast](http://msdn.microsoft.com/en-us/library/ms933794.aspx)|是|仅针对C/C++|
|[PMD](http://pmd.sourceforge.net/)|是|仅针对Java|
|[Google CodeSearchDiggity](https://www.bishopfox.com/resources/tools/google-hacking-diggity/attack-tools/)|是|工具类|
|[FxCop](http://msdn.microsoft.com/en-us/library/bb429476(VS.80).aspx)|是|针对.NET|
|[Flawfinder](http://www.dwheeler.com/flawfinder/)|是|针对C/C++|
|[Ruby on Rails](http://brakemanscanner.org/)|是|仅针对Ruby on Rails|
|[VCG](http://sourceforge.net/projects/visualcodegrepp/)|是|GUI程序、简单规则匹配|
|[bugScout](https://buguroo.com/products/bugblast-next-gen-appsec-platform/bugscout-sca)|否|无法灵活扩展|
|[Contrast from Contrast Security](http://www.contrastsecurity.com/)|否|无法灵活扩展|
|[IBM Security AppScan Source Edition](http://www-01.ibm.com/software/rational/products/appscan/source/)|否|无法灵活扩展|
|[Insight](http://www.klocwork.com/products/insight.asp)|否|无法灵活扩展|
|[Parasoft Test](http://www.parasoft.com/jsp/capabilities/static_analysis.jsp?itemId=547)|否|无法灵活扩展|
|[Pitbull SCC](http://www.pitbullsoftware.net/pitbull-scc-en/)|否|无法灵活扩展|
|[Seeker](http://www.quotium.com/prod/security.php)|否|无法灵活扩展|
|[Pentest](http://www.sourcepatrol.co.uk/)|否|无法灵活扩展|
| [CodeSecure Verifier](www.armorize.com) | 否    |无法灵活扩展|
|[Coverity](http://www.coverity.com/products/security-advisor.html)|否|无法灵活扩展|
|[PVS-Studio](http://www.viva64.com/en/)|否|无法灵活扩展|
|[HP/Fortify](https://www.fortify.com/products/hpfssc/source-code-analyzer.html)|否|无法灵活扩展|
|[Veracode](http://www.veracode.com/)|否|无法灵活扩展|

这些项目专注的点都不一样，商业源代码扫描器动不动就数十万，一般企业负担不起，开源项目又大都不是定位于企业级的，专注的点都不一样。
还没有一款开源的，并且定位于企业级使用的。

作为甲方企业，我们所需要的：
- 能快速扫描新型漏洞（Web应用每天都会有新的漏洞/攻击手法出现，要能快速响应新型漏洞的扫描）
- 能够扫描多种开发语言（公司大了开发语言也会多种多样，需要支持多种开发语言的扫描）
- 能够自动扫描、自动报告（人工参与到每个项目成本太大）

根据我们的目的调研比较后发现，没有一款开源扫描器符合我们的需求。
于是我们选择了做一套符合企业级的白盒代码审计系统。

## 3. 如何实现

代码审计大体方式可以分为两种：静态和动态，

#### 1. 静态审计

通过静态分析源码，发现源码中的逻辑、数据处理、函数使用不当来确认源码中可能存在的漏洞。

同时静态分析技术目前也分为几种。

##### 1.1 规则匹配

说白了就是根据指定的规则扫描代码中的问题。

比如在PHP Kohana框架内，是有封装好统一的取参数的方法，并且经过安全过滤的。而可能出现的情况是新来的研发人员不熟悉导致使用了PHP内置的```$_GET``` / ```$_POST```，从而导致XSS，这时就可以使用```$_GET``` / ```$_POST```作为规则，找出源码中所有出现这些函数的地方。

这种方式也是很多白帽审计代码时用到的，虽然很直接了当，但是误报会比较多。

##### 1.2 代码解析法

通过解析代码的语法，分析出代码执行流程。

##### 1.3 数据流转分析

通过Fuzz输入数据，跟踪数据流转，来判断是否存在风险。

#### 2. 动态审计

通过运行需要审计的代码，跟踪数据的流转来判断系统中是否存在漏洞。

我们一开始只打算扫我们关心的漏洞类型：高危文件、高危函数以及常规Web漏洞。
所以我们第一个版本只做静态审计中的**规则匹配法**，关于误报问题我们在实际扫描中采用了多种方式进行改进，确保误报率在5%以下。

## 4. 接入时机

#### 1. 代码提交时

代码提交时检测问题是最忌时机，具体可以通过hook svn或git的commit来扫描提交的代码。

优点：及时

缺点：影响提交效率

#### 2. 代码提交后

通过设置定时任务在凌晨进行有规律的代码审计，结果通过邮件或BUG系统同步给提交人。

优点：不影响代码提交

缺点：可能代码已经上线才扫到问题

#### 3. 代码发布时

代码发布前的测试环境进行扫描，上线前必须已经扫描完毕并且没有高危漏洞。

优点：不会出现代码已经上线才扫到问题，确保所有上线代码都经过扫描

缺点：发现的不及时


企业可以先接入在**代码发布时**，通过设置**定时任务扫描**来保证及时性和线上代码的覆盖率。

通过我们实际使用感受，发现还有更多用法：

#### 1. 用来判断新漏洞的影响

比如用来判断ImageMagick在公司所有项目中的影响，我们就可以通过设置扫描规则来扫描公司所有项目，看哪些项目有调用ImageMagick并且没有可以利用的。

#### 2. 用来检测明显代码逻辑问题

开发人员不小心将==写成=，造成逻辑问题，甚至可能引发安全问题。

或者是少些了结束的分号，但测试没有覆盖到，导致线上5xx。

这些问题也可以通过代码审计发掘出来。


还有其它更多的使用技巧，后续再慢慢补充...

---
## 5. 应用场景

##### 1.漏洞出现前（检测）
我们将互联网上常见的漏洞梳理为Cobra的检测规则，能够在漏洞被白帽子发现前就扫描出风险点并解决，防范于未然。

**例：**
提前检测代码中是否存在高危文件(.tar.gz/.rar/.bak/.swp)，可以避免高危文件被下载。

##### 2.漏洞出现中（扫描）
当企业收到白帽子提交的漏洞后，企业会在第一时间修复漏洞，并可以通过Cobra来添加扫描规则检测企业的所有项目是否存在类似漏洞。

**例：**
出现了ImageMagick漏洞后，可以通过Cobra设置扫描规则对历史所有项目进行快速扫描，几分钟内就能知道企业数十个项目中哪些有用到ImageMagick组件，哪些存在漏洞，哪些可以免疫。

##### 3.漏洞出现后（限制）
当企业修复漏洞后，可以通过设置修复/验证规则来限制以后所有提交的代码都需要过修复/验证规则，否则不予上线，减少相同漏洞再次出现的可能性。

## 扫描方式

提供自助操作界面供扫描，同时也提供标准全功能[API](http://cobra.feei.cn/api)接口供第三方系统调用（比如发布系统）
![Cobra Framework](https://raw.githubusercontent.com/whaleshark-team/cobra/master/docs/report_01.jpg)
![Cobra Manual](https://raw.githubusercontent.com/whaleshark-team/cobra/master/docs/report_02.jpg)

## 6. 漏洞类型

除了常见的应用程序漏洞，还包括一些代码逻辑漏洞以及文件权限、敏感文件等
具体参见[Cobra Vulnerabilities](http://cobra.feei.cn/labels)

扫描文件涵盖所有常见文件类型，注册检测规则支持Java、PHP两种语言。
具体参见[Cobra Support Language](http://cobra.feei.cn/languages)

另外扫描的准确度、范围都是受各自开启的规则数影响的。

## 7. 扫描过程

扫描器采用多进程设计，可以做到多任务并发扫描，每个扫描进程间互相不会受影响。并发数大小受机器硬件因素决定，可快速增加机器以扩展并发扫描效率。
扫描器对一般项目扫描能做到每秒n万个文件（1 < n < 10），扫描时间主要受触发的规则数所影响。

## 8. 扫描结果

每次扫描完成后，会通知对应的责任人，并生成一张扫描报告。报告内会分析出现漏洞的代码并给修该漏洞的修复方式。
同时可以通过Cobra API获取扫描结果，发布系统可以根据扫描结果里面的建议决定此次是否允许发布上线。

## 9. 规则有效性

新规则的录入时，可以先对历史项目进行测试验证，待误报率降低后再讲该规则状态改为线上开启。
也可以针对自己的业务，手动的关闭一些特殊规则。

## 10. 误报
规则验证的结果还是会存在误报，我们可以通过两个方式解决误报问题。
1. 优化规则 - 通过对误报的分析去优化规则的检测逻辑
2. 白名单 - 某些状态下我们需要某个规则对某个文件的某行放行（比如某些框架层的eval($cmd)），则可以使用白名单

## 11. 现状
目前项目已经开源至GitHub: [Cobra](https://github.com/WhaleShark-Team/cobra)，已经在多家公司内部应用，也欢迎有意向参与维护的提交Pull Request。

