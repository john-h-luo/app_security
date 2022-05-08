## AWS Web Application Firewall保护云上应用，以下简称WAF。
本文将介绍WAF的功能，和相关配套使用的工具功能，如AWS Shield，AWS Firewall Manager等，从保护应用的设置和配置方法和费用成本等角度来分析它能提供哪些安全保护，不足的地方怎么补足等。
首先说结果：
1. 使用WAF可以有效地针对OWASP TOP 10中定义的App攻击薄弱点提供有效保护，此外WAF提供较细颗粒度的HTTP Header设置规则，可以提供较高效的保护，前提是要足够了解HTTP Header规则，以及在实际场景中利用Header攻击的方法和原理。
2. AWS Shield Standard服务向AWS上服务和资源免费开放，不需要额外的配置动作，它默认即提供OSI模型中3，4层保护，可防止SYN Flood, ICMP FloodDDoS，UDP Reflection攻击，但对于6，7层的攻击没有保护能力。AWS Shield Advance在Standard之上提供6，7层的HTTP Flood，SSL abuset等保护，启用Advance每月费用是3000 USD，且在启用之初要同意一年的期限，后续如需要取消需要提前一个月提出申请。
3. AWS Firewall Manager是一个集中的规则管理平台，在启用AWS Shield Advance之后可以使用，它的功能是可以集中管理同一组织下不同帐户和App的规则，对于它的研究可以放在建立了防火墙的规则和有了初步安全体系之后。
WAF的工作原理：
对通过WAF访问App的各类请求进行内容检测和验证，确保其安全性与合法性，对非法的请求予以实时阻断，为Web应用提供防护。
![RUNOOB](https://www.cloudflare.com/img/learning/ddos/glossary/waf/waf.png)
