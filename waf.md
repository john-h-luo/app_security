## AWS Web Application Firewall简称WAF，使用WAF保护应用
本文将介绍WAF的功能，和相关配套使用的工具功能，如AWS Shield，AWS Firewall Manager等，从保护应用的设置和配置方法和费用成本等角度来分析它能提供哪些安全保护，不足的地方怎么补足等。
首先说结果：
### WAF
使用WAF可以有效地针对OWASP TOP 10中定义的App攻击薄弱点提供有效保护，此外WAF提供较细颗粒度的HTTP Header设置规则，可以提供较高效的保护，前提是要足够了解HTTP Header规则，以及在实际场景中利用Header攻击的方法和原理。
### Shield
AWS Shield Standard服务向AWS上服务和资源免费开放，不需要额外的配置动作，它默认即提供OSI模型中3，4层保护，可防止SYN Flood, ICMP FloodDDoS，UDP Reflection攻击，但对于6，7层的攻击没有保护能力。AWS Shield Advance在Standard之上提供6，7层的HTTP Flood，SSL abuset等保护，启用Advance每月费用是3000 USD，且在启用之初要同意一年的期限，后续如需要取消需要提前一个月提出申请。
### Firewall Manager
AWS Firewall Manager是一个集中的规则管理平台，在启用AWS Shield Advance之后可以使用，它的功能是可以集中管理同一组织下不同帐户和App的规则，对于它的研究可以放在建立了防火墙的规则和有了初步安全体系之后。
## WAF的工作原理：
### Network Traffic
对通过WAF访问App的各类请求进行内容检测和验证，确保其安全性与合法性，对非法的请求予以实时阻断，为Web应用提供防护。
![alt](https://www.cloudflare.com/img/learning/ddos/glossary/waf/waf.png)
如上图所示，WAF的安全机制是通过检测访问流量通过时的行为来保护后方的App的安全，即要求所有的访问网络流量必须通过WAF。AWS网络组件包含VPC，ALB，Elastic IP，Security Group等，在一个单一网络环境下的App的网络流量路径大体如下：WAF->ALB->EC2，路径上每一节点应该有严格的开放端口，IP地址白名单、黑名单设置，例如：WAF上侦听443端口，设置ACL黑名单，EC2安全组设置只允许ALB访问，这样才能保证WAF不会被Bypass。不同方案所用的AWS资源可能有所不同，比如其它类型的Load balancer或是非EC2的计算单元，但总体上来说流量路径差别不大。
### ACL说明
WAF对于访问流量的控制是基于Access Control List简称ACL来实现的，创建ACL是按AWS Region划分的，所以如果App部署在某一个Region则相应的ACL也应该部署于同一Region，对于CloudFront支持Global ACL这里暂不讨论，后续会讨论Global ACL有什么优点。
### ACL创建
进入AWS WAF Console选择Web ACLs，Create Web ACL
#### ACL命名和添加资源
输入必要的参数name，其它的参数会自动生成如CloudWatch metric name，如果没有特别的命名规则不必要修改，Resource Type选择Regional resources，然后选择App所在的Region，最后添加AWS resource即WAF保护的后方的App，由于WAF的设计结构只支持添加API Gateway，ALB，AppSync，所以如果App部署在EC2上则首先要将EC2添加到ALB中，再从WAF添加ALB。如果App是部署在EKS平台，则在创建service时会生成ALB（具体的service定义看kubernetes的设计）
#### 添加ACL Rule
点击Add rules可选择添加AWS Managed Rules或是第三方服务商提供的Rule，如F5，Imperva
WCUs全称Web ACL rule capacity units used，按每条Rule的处理算力计算，越复杂的Rule数值则越大，总体上限1500，如一个ACL超过此上限则需要通过升级渠道将需要提交给AWS后台处理。
Default ACL action
支持的Action如下Allow，Block。在Allow的情况下还可以添加自定义的HTTP Request Header，在后方的App可以对自定义Header做处理规则，需要足够了解此类规则。
#### Set rule priority
设置优先级的顺序，调整和优化规则执行的效率。例如Allow和Block规则的优化，举个例子：如果有一个规则是Block那么优先级较高，这样命中规则即Block，而避免其它规则放行后，最终还是被Block。
#### Configure metrics
这里的Metrics和创建ACL的有所不同，这里的是Rule规则的Metric，举例说明：ACL Metric是指从这条ACL中通过了总的流量，而Rule是指这里命中这个规则的Metric。另外启用Request sampling requests会提供一些Metric的信息切片，可以从信息切片中了解到ACL的情况。
#### Review and create web ACL
创建ACL的最后一步，Review设置，和做最后的修改，一切验证通过后选择Create web ACL。之后在Web ACLs界面可以查看图表，和Sampled requests。
### Bot Control
控制网络爬虫，内容爬虫和SEO搜索引擎排名的行为，可设置Allow或Block。启用Bot control需要在添加ACL Rule中添加AWS managed rule列表中的Bot control rule。

