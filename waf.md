## AWS Web Application Firewall简称WAF，使用WAF保护应用
本文将介绍WAF的功能，和相关配套使用的工具功能，如AWS Shield，AWS Firewall Manager等，从保护应用的设置和配置方法和费用成本等角度来分析它能提供哪些安全保护，不足的地方怎么补足等。
首先说结果：
### WAF
使用WAF可以有效地针对OWASP TOP 10中定义的App攻击薄弱点提供有效保护，此外WAF提供较细颗粒度的HTTP Header设置规则，可以提供较好的保护，前提是要足够了解HTTP Header规则，以及在实际场景中攻击的方法和原理。费用按每ACL 5美元/条，Rule 1美元/条每月，Request 1,000,000条0.6美元。
### Shield
AWS Shield Standard服务向AWS上服务和资源免费开放，不需要额外的配置动作，它默认即提供OSI模型中3，4层保护，可防止SYN Flood, ICMP Flood，UDP Reflection攻击，但对于6，7层的攻击没有保护能力。AWS Shield Advance在Standard之上提供6，7层的HTTP Flood，SSL abuset等保护，启用Advance每月费用是3000 USD，且在启用之初要同意一年的期限，后续如需要取消需要提前一个月提出申请。
### Firewall Manager
AWS Firewall Manager是一个集中的规则管理平台，在启用AWS Shield Advance之后可以使用，它的功能是可以集中管理同一组织下不同帐户和App的规则，对于它的研究可以放在建立了防火墙的规则和有了初步安全体系之后。
## WAF的工作原理：
### Network Traffic
对通过WAF访问App的各类请求进行内容检测和验证，确保其安全性与合法性，对非法的请求予以实时阻断，为Web应用提供防护。
![alt](screenshot/waf.png)
如上图所示，WAF的安全机制是通过检测访问流量通过时的行为来保护后方的App的安全，即要求所有的访问网络流量必须通过WAF。AWS网络组件包含VPC，ALB，Elastic IP，Security Group等，在一个单一网络环境下的App的网络流量路径大体如下：WAF->ALB->EC2，路径上每一节点应该有严格的开放端口，IP地址白名单、黑名单设置，例如：WAF上侦听443端口，设置ACL黑名单，EC2安全组设置只允许ALB访问，这样才能保证WAF不会被Bypass。不同方案所用的AWS资源可能有所不同，比如其它类型的Load balancer或是非EC2的计算单元，但总体上来说流量路径差别不大。
### ACL说明
WAF对于访问流量的控制是基于Access Control List简称ACL来实现的，创建ACL是按AWS Region划分的，所以如果App部署在某一个Region则相应的ACL也应该部署于同一Region，对于CloudFront支持Global ACL这里暂不讨论，后续会讨论Global ACL有什么优点。
### ACL创建
进入AWS WAF Console选择Web ACLs，Create Web ACL
![alt](screenshot/waf.png)
![alt](screenshot/create_acl.png)
#### ACL命名和添加资源
输入必要的参数name，其它的参数会自动生成如CloudWatch metric name，如果没有特别的命名规则不必要修改，Resource Type选择Regional resources，然后选择App所在的Region，最后添加AWS resource即WAF保护的后方的App，由于WAF的设计结构只支持添加API Gateway，ALB，AppSync，所以如果App部署在EC2上则首先要将EC2添加到ALB中，再从WAF添加ALB。如果App是部署在EKS平台，则在创建service时会生成ALB（具体的service定义看kubernetes的设计）
![alt](screenshot/add_resource.png)
#### 添加ACL Rule
点击Add rules可选择添加AWS Managed Rules或是第三方服务商提供的Rule，如F5，Imperva
WCUs全称Web ACL rule capacity units used，按每条Rule的处理算力计算，越复杂的Rule数值则越大，总体上限1500，如一个ACL超过此上限则需要通过升级渠道将需要提交给AWS后台处理。
Default ACL action
支持的Action如下Allow，Block。在Allow的情况下还可以添加自定义的HTTP Request Header，在后方的App可以对自定义Header做处理规则，需要足够了解此类规则。
![alt](screenshot/add_rules.png)
#### Set rule priority
设置优先级的顺序，调整和优化规则执行的效率。例如Allow和Block规则的优化，举个例子：如果有一个规则是Block那么优先级较高，这样命中规则即Block，而避免其它规则放行后，最终还是被Block降低效率。
#### Configure metrics
这里的Metrics和创建ACL的有所不同，这里的是Rule规则的Metric，举例说明：ACL Metric是指从这条ACL中通过了总的流量，而Rule是指这里命中这个规则的Metric。另外启用Request sampling requests会提供一些Metric的信息切片，可以从信息切片中了解到ACL的情况。
#### Review and create web ACL
创建ACL的最后一步，Review设置，和做最后的修改，一切验证通过后选择Create web ACL。之后在Web ACLs界面可以查看图表，和Sampled requests。
### Bot Control
控制网络爬虫，内容爬虫和SEO搜索引擎排名的行为，可设置Allow或Block。启用Bot control需要在添加ACL Rule中添加AWS managed rule列表中的Bot control rule。
### IP sets / Regex pattern sets
相当于设置一个IP地址集，在设置Rule时可直接引用，例如Block一个IP地址，或是某个HTTP请求中命中Regex。
### Rule groups
创建自定义Rule，相对于AWS Managed rule自定义程度更高，但需要对网络安全理解程序较高。可创建Regular规则，和Rate-based规则。Regular规则支持定义一个Statement和AND/OR/NOT逻辑运算，例如定义一个Statement限定网络地址从除中国以外的全部Block，Statement支持按国家，IP地址，标签，Request Header，Cookie，URI Path，BODY等，所以要求会比较高。而Rate-based规则在Regular规则之上支持执行Rate规则，输入一个Rate limit假设100，例如同一个IP地址在一分钟内创建超过100个Request就会执行定义的Action，例如Block，要求CAPTCHA，或是Count，Count的作用是添加计数，它主要用于处理通过一条Rule很难就定义的Request，例如一个请求来源于一个可疑的IP地址，所以就先给它加一个Count，在后续其它的Rule处理时可以根据计数来衡量整体的安全分数，最终根据整体Count来决定是否Block。
### AWS Marketplace
包含第三方安全厂商设计定义的Rule，每个厂商有不同的安全架构设计经验所以不同的Rule侧重点不同，整体来看如果采用第三方厂商的Rule选择一个与自家的App相似度最高的即足够，然后再使用AWS Managed rule，或是自定义Rule补足。第三方厂商Rule的费用方面除了支持AWS ACL rule的费用之外，还需要支持Subscription的费用。
## Shield Advance
在开始时已经讲过Advance可以保护6，7层攻击，所以这里不再复述，这里主要讲SRT support和其它重要的概念。SRT是AWS Shield Response Team的简称，它的作用是在预防DDoS攻击的发生，从发现攻击开始SRT团队的安全工程师就会介入，帮助抵抗攻击和制定Service Available的方案来保护服务可用性。除了SRT之外，如果App配置了自动扩展，在被攻击时被动发生的自动扩展，和因应对DDoS时扩展资源来保证服务可用性，所产生的费用AWS也会免除。
## CloudFront和Global ACL
CloudFront是建立在Edge computing的基础之上的，相当于说在每个启用了Edge的AWS Region会建一个Edge节点。它对于WAF的影响来说，在于如果使用了CloudFront服务就会获得Edge的安全防护，从App的访问流量路径来看，要首先经过AWS Edge节点，这样可以利用AWS平台的资源和安全优势来抵挡攻击，等于在你的App的WAF之前再额外加一层AWS平台的保护。
