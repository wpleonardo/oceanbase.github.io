---
slug: Eccang
title: 'Eccang × OceanBase: Building a Full-Ecosystem SaaS Platform for New Retail in Cross-Boarder Ecommerce'
tags:
  - User Case
---


Some say South China is the country's land of cross-border e-commerce, with Shenzhen city being the capital. In 2021, China's cross-border e-commerce market reached a new high of CNY 14.2 trillion, maintaining a growth rate of 13.6% year-on-year. Over the past five years, the country's cross-border e-commerce has grown nearly tenfold. As the industry flourishes, a dark horse based in Shenzhen has cut a dashing figure.

  

Shenzhen Eccang Technology Co., Ltd. ("Eccang" for short), founded in 2013, has been dedicated to developing efficient and controllable cross-border management models. It was the first service provider in the industry with a full-featured mid-end platform. On February 8, 2021, Eccang secured USD 40 million in Series B funding, setting a new record for the highest single investment in the software as a service (SaaS) sector of the cross-border e-commerce industry. Lying low for nearly a decade, Eccang has now partnered with many cross-border sellers, overseas warehouse service providers, and logistics giants around the globe. Its overseas Warehouse Management System (WMS) has a market share of nearly 70%, positioning it at the forefront of the industry.

  

In this article, Cheng Han, the database administrator (DBA) responsible for database architecture and optimization at Eccang, shares insights on database selection and operation when facing massive cross-border business data, so as to help companies reduce tenant costs and improve operational efficiency while enhancing product experience and supporting customer success.

  

  

1. More About Eccang
--------

  

SaaS for cross-border e-commerce came into bud in 2014, and has experienced fast growth through brand building, harsh times during COVID, and rational adjustments under the influence of black swan events. Eccang, established in 2013, seized the opportunity of SaaS development and became an industry leader through years of hardworking.

  

While many believe that Eccang is a full-ecosystem software service provider in the cross-border industry, I prefer to see us as a new retail SaaS provider.

  

![1](/img/blogs/users/Eccang/image/4a98be75-7b94-47b2-99dc-676590975a67.png)

  

Why? Eccang is restructuring the cross-border ecosystem using the internet and big data technologies, connecting overseas buyers to sellers, logistics providers, carriers, and factory suppliers here in China. Eccang offers a diverse product mix, including ERP for cross-border e-commerce companies, TMS for international logistics providers, WMS for overseas warehousing services, and M2B for cross-border distributors. Maintaining tens of thousands of tenants for customers, we are the first SaaS provider that integrates the upstream and downstream of the entire supply chain in the cross-border industry.

  

  

2. Inherent Multitenancy of a SaaS Architecture
-----------------

  

When it comes to SaaS, tenant is a must-mention. I bet peer companies in the cross-border industry are all stressed in handling high costs per tenant. In a conventional architecture, multitenancy is often implemented in the following methods.

  

![2](/img/blogs/users/Eccang/image/2a8bb069-b0f8-4e24-bc5a-b68551bdb7e3.png)

  

Each of these methods has its pros and cons. In view of the challenging market conditions today, Eccang aims to provide quality service to each tenant at minimal costs. We use methods 1 and 3 flexibly depending on the specific business scenarios of different customers.

  

  

3. The Same Choice of a Multitenant Architecture
-----------

In fact, each tenant in our SaaS architecture is associated with the business of a specific customer, which aligns exactly with the multitenancy concept of OceanBase Database. Eccang provides tenants with varying resource sharing modes for customers of all sizes and budgets. Top-tier customers may use dedicated resources. Mid-tier customers may use resource pools shared among a limited number of tenants. Lower-tier customers may use shared resource pools accessible to a larger number of tenants. Free-plan customers and trial users have access to shared resource pools of a larger cluster containing an even greater number of tenants.

  

![3](/img/blogs/users/Eccang/image/1f78321f-b123-40f8-bfee-7fe65861ead7.png)

  

Resources dedicated to a tenant are exclusively provided to the tenant, including resources of the web, application, and database layers. Shared resources may be allocated from separate web, application, and database layers. Apparently, it is impossible to manually allocate dedicated or shared resources for each tenant. Instead, our system automatically activates corresponding services based on the products customers purchase. If, for example, a customer purchases a dedicated tenant, the system automatically activates a database for this customer, and a new customer signs up to a shared tenant is automatically bound to a shared database pool.

  

Previously, all Eccang tenants - whether free, paid, small, or large - were physically isolated at the database level. Large tenants may be isolated at the instance level. With such an architecture, managing numerous tenants, we would likely stress out in tackling challenges caused by a massive number of servers and databases, petabytes of stored data, and rocketing traffic peaks.

  

  

4. Vulnerabilities of Our Previous Database Architecture
------------

As China's cross-border e-commerce grew rapidly, Eccang experienced a rocketing increase in business volume, with numerous customers generating petabytes of data. How should we design our database architecture?

  

The following figure shows our previous primary-standby database architecture. Data was transferred from databases through an extract-transform-load (ETL) layer to an offline or real-time computing module for business use. To support our cross-border business, we had deployed servers both at home and abroad.

  

![4](/img/blogs/users/Eccang/image/afae8668-753c-41af-a202-9c4eac71a26a.png)

**▋ Pain point 1: Massive data and tables, low operational efficiency, and database performance bottlenecks**

  

After nearly 10 years of development, we had numerous database instances and massive stored data. Initially, a single tenant had over 1,000 tables, and the number could be as large as 3,000 today.

  

Massive data and tables made operations extremely challenging. We needed a database O&M platform for task scheduling, as well as the monitoring, alerting, basic maintenance, and self-healing of all instances. The O&M of cloud databases was kind of easy, but building a monitoring system to do intelligent O&M of self-managed on-premises databases could be a hard nut to crack when you had servers in the US, Russia, and EU countries.

  

To maximize resource utilization, we deployed as many tenants as possible in a single database instance. For example, we might deploy 10 tenants in an instance for mid-tier customers, and 100 tenants for lower-tier customers, and the numbers might be 500 or even 1,000 for free-plan customers. If a single instance served 1,000 tenants with 3,000 tables each, it would have to handle millions of tables - an enormous challenge for any database.

  

The most serious database bottleneck back then was the excessive number of tables. All resources, physical or cloud based, were exhausted. Millions of tables might translate to thousands of physical files, potentially overwhelming all physical machines. With swelling customer data volumes, database performance bottlenecks became increasingly intolerable.

  

Most database systems on the market could hardly meet our needs, and even if they provided required features, other factors might turn them down - for example, huge data backups would directly challenge the operating system.

  

**▋ Pain point 2: Complex big data synchronization and SLA challenge**

  

Eccang's data is sourced from sellers on various overseas platforms, selling products to buyers in North America, Europe, Southeast Asia, and other regions. Data is transferred from these platforms to China. In addition, a seller may run manufacturing plants in China, so the product delivery may involve domestic and international logistics services. This results in complex and lengthy links in our ETL layer, making it hard to manage stability, availability, and real-time performance.

  

To aggregate data from various platforms, we must create numerous instances for big data synchronization links, demanding high database timeliness and stability. If you had a hundred RDS instances or MySQL databases, you must create and maintain a hundred big data synchronization links, making it a daunting challenge to meet the SLA requirements.

  

Just one link instability or data loss event in a month would destroy the service stability. With thousands of instances, we had to figure out how to ensure stable big data services and zero loss in real-time data synchronization.

  

**▋ Pain point 3: Increasing customer data storage costs and significant resource waste**

  

Eccang only stores data of customers, who actually own the data. Many customers have used our services for years. Assuming they can secure 10 million orders a year on average, that would amount to 30 million orders after three years, occupying a great storage space. Technically, their storage costs would increase. From the customer's perspective, however, their annual order volume remains roughly the same. Why should they pay more for data storage? As a SaaS provider with subscription-based pricing, Eccang has no grounds to charge them for incremental data storage.

  

As more customers subscribed to our services, their data volume grew fast, too, leading to a considerable annual rise in our storage and server costs.

  

Additionally, there was resource waste. Anyone familiar with MySQL or databases knows that many resources, especially cloud resources, should be pre-paid regardless of whether you have used it. For example, you bought a server with 8 CPU cores, 16 GB of memory, and 1 TB of disk space, and your business might use 70% of the CPU resources and 80% of the memory, leaving 20-30% of the resources wasted. When you have only 5 to 10 servers, this waste might not be a serious concern, but with thousands of servers, the financial impact would be significant.

  

  

5. Benefits of a Native Distributed Database
------------

We've been looking for a database that could address those pain points. We tried TiDB and researched various cloud-native and distributed databases, and eventually picked OceanBase Database for its tenant management mechanism fits our specific needs the best.

  

Cross-border business folks knew that small- and medium-sized customers might embrace a blasting increase in order volumes during promotional events like Double 11 and Double 12 shopping festivals, Christmas sales, and Black Friday. DBAs must be prepared to handle these situations, to maintain customers' business stability through traffic peaks.

  

So we needed a database that could aggregate resources and support tenant resource isolation, shared instances, and automatic scaling. To be specific, the database should schedule physical resources among tenants, and rapidly scale up a tenant when its resource usage exceeds the upper thresholds due to traffic spikes, and scale down a tenant when its resource usage drops below the lower thresholds. OceanBase Database provides what we were looking for.

  

Previously, 100 tenants might require 10 to 20 RDS instances. Now, a single OceanBase Database instance suffices. This way, the resource waste could be mitigated significantly. OceanBase Database supports multiple tenants in a large cluster, substantially improving database resource utilization and DBAs' work efficiency.

  

**Reduced storage costs**: OceanBase Database boasts a remarkably high compression ratio. A 1.2 TB database migrated to OceanBase Database occupies only 140 GB, achieving a space saving of 88%, which greatly slashes our data storage costs.

  

![6](/img/blogs/users/Eccang/image/3b69270b-a210-4eb3-89ea-ef6a54ea5c5a.png)

  

**Big data synchronization**: Previously, we managed 10 to 30 synchronization links between servers distributed around the world. In the case of a network error, we would have to pull the error data back to China for analysis. That's why we are very demanding concerning the link quality and real-time performance. Now, within a single OceanBase cluster, we can guarantee the stability of all tenants by maintaining only one synchronization link.

  

**Hybrid transaction and analytical processing (HTAP)**: Long-time customers might query data quarters or even years old. We used to provide big data solutions to take care of those simple needs, but we also need HTAP capabilities to handle their real-time queries.

  

So far, OceanBase Database has demonstrated impressive HTAP capabilities. If we use TiDB, PolarDB, or ClickHouse, additional components are required for data synchronization. In addition, columnar storage and row-based storage are physically separated in TiDB. Unlike that, OceanBase Database supports hybrid row-column storage. We can simply create a table and import data, and OceanBase Database then automatically separates rowstore and columnstore data. Mind-blowing, right?

  

**Postscript**

  

**He Zhiyong** | OceanBase Solution Architect

  

I'm honored to provide technical support and services for Eccang, and through this opportunity, I've witnessed the flourishing growth of the SaaS market in cross-border e-commerce. Having honed its expertise in the industry for years, Eccang stands out as a leader. It has served countless customers with constantly updated products and has grown into a full-stack service provider in the cross-border industry.

  

Against the backdrop of global cross-border e-commerce, customer expectations for digital services are increasingly high. Companies must keep enhancing their system robustness, stability, efficiency, security, and reliability, to provide high-quality services and help customers optimize costs while maintaining cost advantages in their respective industries. Eccang has never stopped its exploration in cutting-edge technologies to break through performance bottlenecks of conventional architectures. Since the first half of this year, Eccang has launched a series of tests on OceanBase Database, and finally integrated its business systems with OceanBase Database after comprehensive business adaptation and verification.

  

I believe that Eccang can further strengthen its business ecosystem with capabilities of OceanBase Database and will attract more customers to create greater shared success.

  

Follow us in the [OceanBase community](https://open.oceanbase.com/blog). We aspire to regularly contribute technical information so we can all move forward together.

  

Search 🔍 DingTalk group 33254054 or scan the QR code below to join the OceanBase technical Q&A group. You can find answers to all your technical questions there.

  

![7](https://gw.alipayobjects.com/zos/oceanbase/f4d95b17-3494-4004-8295-09ab4e649b68/image/2022-08-29/00ff7894-c260-446d-939d-f98aa6648760.png)