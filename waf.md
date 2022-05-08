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