Sure thing. Let me tell you about a project where I helped a customer migrate their entire on-prem data center—everything from SQL Server clusters and web apps to an on-prem Kubernetes setup and Active Directory—over to Azure. It was a pretty big deal, so I’ll walk you through the major steps.

**Evaluation**
First off, we checked out their existing environment, their OS&kernel version, storage size, compute, network and etc. 
After we got a good picture of their setup, we are using Azure Migrate’s discovery tools to figure out which applications and databases depended on each other. 

**VM Size:**
We also worked closely with their architecture team to decide which VM sizes should use based on indicator like memory-to-vCPU ratio, disk I/O, throughput for each server role—like web servers, SQL servers, Exchange servers, and backend servers—so we could balance performance and cost. 
For example, we typically recommended: Web servers with moderate traffic: Standard D2s_v3 or B-series VMs, offering a balanced CPU/memory ratio. 
SQL servers that are memory-intensive: E-series or M-series VMs, providing higher memory-to-vCPU ratios ideal for large in-memory databases. Exchange servers: Carefully sized to handle spikes in email concurrency and I/O demands. 
Backend servers: Often benefit from the flexibility of the D-series for a mix of compute and memory.

**Storage**
In terms of storage, we evaluated Standard HDD, Premium SSD, or Ultra SSD based on each application’s IOPS and throughput needs, making sure each workload had the right disk tier to match its performance requirements.
 
**Network**
We set up a network in Azure, making sure the IP address ranges didn’t overlap with on-prem. We created separate subnets for web, data, and management layers just like their on-premise network environment, 
and we also integrated the customer’s local DNS zones into Azure DNS, ensuring that internal name resolution worked seamlessly in both environments. 
We also used Network Security Groups and Azure Firewall to make sure everything stayed locked down.

**Install mobility agent**
Then came the installation of the Mobility Agent on all the servers—both Windows and Linux. 
In some Linux servers, we found that the official installer would sometimes fail because of SELinux restrictions or certain file system attributes—like the /var partition being mounted with noexec—which prevented the agent from running scripts or installing properly. 
We then created a script that quickly updated all the affected servers to remount /var without the noexec flag, we actually used a simple shell script combined with an inventory list of all the servers. 
The script connected over SSH to each server automatically and updated the mount options, so it was effectively one command that iterated through every machine. That saved us a ton of time compared to doing it manually.

**Replication**
<这里，服务器正从本地传输数据到云端，可以后面介绍你怎么帮助客户在云上对recovery service vault(储存传输数据的地方)去配置diagnostic setting，写query去并且配合logic app等自动化报警系统>
通过diagnostic setting 收集 Windows/Linux 服务器日志、应用程序日志（IIS、Nginx、Tomcat）、网络流量监控（NSG Flow Logs） 以及 数据库查询性能（SQL Query Store），存储到 Log Analytics Workspace（LA） 进行处理。数据分析与可视化：

编写 Kusto Query Language（KQL） 查询规则，生成 业务健康监控报表、资源利用率趋势分析、异常日志筛选 等可视化数据面板。 通过 Azure Monitor Workbooks 构建 自定义监控面板，提供高层级的系统运行状态概览。 网络故障排查：

结合 Network Watcher、Packet Capture、Connection Monitor，分析 Azure 资源之间的网络连通性问题，如 ExpressRoute 连接丢包、VNet Peering 访问异常、NSG 端口封锁，提高整体网络稳定性。 监控 负载均衡器（ALB/NLB）、CDN 流量、VPN 网关性能，优化跨区域/跨站点的访问体验。

监控 Azure VM 运行状态、数据库响应时间、API 请求失败率，设置 Metrics-based Alerts，提前发现潜在问题。 结合 Action Groups，配置 多渠道告警通知（邮件、短信、Teams、ServiceNow），确保关键事件即时响应。 自动化事件响应
When it was time to actually replication, the customer was using Azure Private Link for the replication traffic, which meant all data flowed through private endpoints. 
Initially, some of their local DNS servers couldn’t resolve the private endpoint correctly, causing replication failures on a few machines. 
We caught this by running checks like tnc (Test-NetConnection), telnet, or ping to see if we could reach the endpoint. 
Then we dive deeper with Wireshark,  We filtered for DNS traffic (port 53) and saw that requests for the private endpoint domain were returning NXDOMAIN or failing to respond.  
Once we updated the DNS records for the private endpoints for the DNS server, replication started working for most servers.
However, a handful of servers still couldn’t connect securely. We also filtered traffic on TCP port 443 to examine the TLS handshake. 
We saw those servers only offered outdated ciphers (like RC4) and didn’t support TLS 1.2 cipher suites Azure expects. Once we enabled or installed these cipher suites on the older servers, the TLS handshake succeeded.
Meanwhile, since it will swamp their network bandwidth if all the servers are replicating at the same time. 
We split up the SQL databases into different replication groups based on their importance—critical workloads got replicated first, and the less urgent ones came later.

**Test Migration**
Once replication was going smoothly, we did a test migration. We fired up everything in Azure—the web apps, SQL clusters, and the AKS environment—and tested it all. 
That meant checking DNS resolution, hitting endpoints with Postman to test various HTTP methods (e.g., GET, POST) and confirm the correct status codes (like 200, 404, 500) as well as verify the JSON or HTML payloads. 
This let us compare responses against the on-prem environment to ensure functionality and data consistency, and making sure the SQL queries in Azure returned the same results as on-prem. 
For example, we ran queries like "SELECT COUNT(*)" on critical tables to compare row counts, "SELECT TOP 10 * FROM Orders ORDER BY OrderDate DESC" to check recent transactions, and executed stored procedures that the application relied on to confirm they produced identical results in both environments. 
By comparing these outputs, we verified data integrity, row consistency, and overall application functionality post-migration.
We did run into a hiccup where some domain-joined servers lost their trust relationship with AD after we failed them over, but we got that squared away by working with the AD team, rejoining them to the domain, and updating the DNS records. 
Another issue was network latency in certain regions, which we tracked down using packet captures and log analytics, and then fixed by adjusting some routing.

**Final Migration**
Once we were sure everything looked good, we scheduled the final cutover to avoid the customer’s peak business hours. Honestly, at that point, it was pretty smooth sailing. We hit our RTO of 2 hours, so no major drama.
 
<这里，服务器已经全部迁移上云，可以后面介绍你怎么负责帮客户搭建云上监控系统>
crash consistency(崩溃一致性)
app consistency(应用一致性)

