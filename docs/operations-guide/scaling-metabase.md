# Scaling Metabase

Metabase is scalable, battle-tested software used by tens of thousands of companies. It supports high availability via horizontal scaling, and high performance via vertical scaling. Plus it's efficient out of the box: a single core machine with 4 gigs of RAM can scale Metabase to hundreds of users.

This article provides guidance on how to keep Metabase running smoothly in production as the numbers of users and data sources increase. 

We'll cover:

- [**Factors that impact Metabase performance and availability.**](#factors-that-impact-metabase-performance-and-availability)
- [**Vertical scaling.**](#vertical-scaling) Running a single instance of Metabase with more cores and memory. Vertical scaling can improve Metabase application performance.
- [**Horizontal scaling.**](#horizontal-scaling) Running multiple instances of Metabase. Horizontal scaling can improve availability/reliability of your Metabase application.
- [**Data warehouse.**](#data-warehouse) Way out-of-scope for this article, but we'll discuss some general strategies for data warehousekeeping.
- [**Metabase application best practices.**](#metabase-application-best-practices) Tuning dashboards and questions to improve performance.
- [**Hosted Metabase.**](#hosted-metabase) if you don't want to deal with the care and feeding of your application: Metabase also offers a [hosted solution](https://www.metabase.com/hosting/) that takes operating Metabase off your plate so you can focus on your data.

Because the domain is so complex, and each system is different, we can only discuss scaling strategies at a high level, so you'll have to translate these strategies to your particular environment and usage.

## Factors that impact Metabase performance and availability

Metabase scales well (both vertically and horizontally), but Metabase is only one component of your data warehouse, and the overall performance of your system will depend on the composition of your system and your usage patterns.

Major factors that impact your experience using Metabase include:

- The number of databases connected to Metabase.
- The number of tables in each database.
- The efficiency of your data warehouse.
- The number of questions in your dashboards.

For example, it won't matter how many instances of Metabase you run if a question needs to grab records from thousands of tables in a database. That's just going to take a while. The solution in that case is either to re-evaluate your need for that data (do you really need all that info every time?), or to find ways to improve the performance of your database, such as reorganizing, indexing, or caching your data.

But first, let's make sure our Metabase application is well-tuned to scale.

## Vertical scaling

Vertical scaling is the brute force approach. Give Metabase more cores and memory, and it will have more resources available to do its work. If you are experiencing performance issues related to the application itself (i.e. unrelated to the breadth and magnitude of your databases), running Metabase on a more powerful machine can help improve performance.

Metabase is already efficient out of the box. For example, for a starter Metabase instance on AWS, we recommend running Metabase using Elastic Beanstalk on a `t2.small` instance, and scaling up from there. That's a single core machine with 2 gigabytes of RAM. Machines with with 4-8 gigs of RAM should handle hundreds of users, and you can bump the number of cores and gigabytes of memory if needed.

## Horizontal scaling

Horizontal scaling involves running multiple instances of Metabase and using a load balancer to direct traffic to the instances. The primary use cases for horizontal scaling is to improve reliability (a.k.a. "high availability").

Metabase is set up for horizontal scaling out of the box, so you don't need any special configuration to run multiple instances of Metabase. The user session data is stored in the external application database, so users don't have to worry about losing saved work if one machine goes down, and administrators don't have to deal with configuring sticky sessions to make sure users are connected to the right Metabase instance. The load balancer will route users to an available instance so users can keep right on working.

When scaling horizontally, however, you must use an external relational database like PostgreSQL to store your application data (all of your questions, dashboards, logs, and other Metabase data), so that all instances of Metabase can share a common database. Though you can maintain your own application database, we recommend using a managed database solution with high availability, such as [AWS's Relational Database Service (RDS)](https://aws.amazon.com/rds/), [Google Cloud Platform's Cloud SQL](https://cloud.google.com/sql), [Microsoft Azure's SQL Database](https://azure.microsoft.com/en-us/services/sql-database/), or similar offering, so your application database will always be available for your Metabase instances. Managed database solutions are especially useful for Enterprise customers who take advantage of Metabase's [auditing functionality](../enterprise-guide/audit.md).

### Time-based horizontal scaling

Some customers adjust the number of Metabase instances based on the time of day. For example, customers will spin up multiple instances of Metabase in the morning to handle a burst of traffic when users log in and run their morning dashboards, then spin the extra Metabase instances down in the afternoon (or at night, or on the weekends) to save money on cloud spend.

### Configuring the load balancer

Setting up a load balancer is pretty straightforward. Metabase's API exposes a health check endpoint `/api/health` that load balancers can use to determine whether or not a Metabase instance is up and responding to requests. If the instance is healthy, the endpoint will return `{"status":"ok"}`.

See our guide to [running Metabase on AWS Elastic Beanstalk](running-metabase-on-elastic-beanstalk.md) to see an example of setting up a load balancer to use the `/api/health` endpoint.

## Data warehouse

Architecting a data warehouse is beyond the scope of this article, but your queries in Metabase will only be as fast as your databases can return data. If you have questions that ask for data that takes your database a long time to retrieve, those query times will impact your experience, regardless of how fast Metabase is.

In general, there are three ways to improve data warehouse performance:

- **Structure your data in a way that anticipates the questions your users will ask.** Identify your usage patterns and store your data in a way that make its easy to return results for questions common in your organization. Compose ETLs to create new tables that bring together frequently queried data from multiple sources.
- **Tune your databases.** Read up on the documentation for your databases to learn how to improve their performance via indexing and caching.
- **Improve the precision of your questions**. Filter your data as much as you can, and add limits to your queries when composing them. You should also take advantage of Metabase's data exploration tools (including record previews) so you only query data relevant to the question you're trying to answer.

## Metabase application best practices

Here are some strategies to get the most out of your Metabase application:

- [Use an external database to store you Metabase application data](#use-an-external-database-to-store-your-metabase-application-data)
- [Upgrade to the latest version of Metabase](#upgrade-to-the-latest-version-of-metabase)
- [Only ask for the data you need](#only-ask-for-the-data-you-need)
- [Cache your queries](#cache-your-queries)
- [Look for bottlenecks](#look-for-bottlenecks)
- [Keep dashboards to 7 questions or fewer](#keep-dashboards-to-7-questions-or-fewer)
- [Update your browser](#update-your-browser)

### Use an external database to store your Metabase application data

The application database stores all of your questions, dashboards, collections, permissions, and other data related to the Metabase application. We recommend you use an external database (like PostgreSQL or MySQL) to manage your application database. You can also use a managed relational database, like AWS RDS, which will auto-scale for your needs.

### Upgrade to the latest version of Metabase

If you haven't already, we recommend you update to the latest Metabase version to get the most recent performance improvements.

### Only ask for the data you need

In general, you should only query the data you need. If you set up a dashboard that you'll be viewing daily, you can reduce load times by limiting the number of records your queries return.

If you have many users running queries that return a lot of records, it won't matter that Metabase is fast: the users will get their data only as fast as your database(s) can return the requested records.

Take advantage of Metabase's data exploration tools to learn about your data and preview records in tables so you can dial in on only the records you need to answer your question.

### Cache your queries

You can [configure caching](14-caching.md) on questions to store their results. Metabase will show users the timestamp for the results, and users can manually refresh the results if they want to rerun the query. Caching is a great way to save time (and money) for results that do not update frequently.

### Look for bottlenecks

Metabase's Enterprise Edition offers [auditing tools](audit.md) for you to monitor the usage and performance of your application. You can, for example, see how many questions are being asked, by whom, and how long the questions took to run, which can help identify any bottlenecks that need attention.

### Keep dashboards to 7 questions or fewer

Sometimes people go overboard with dashboards, loading them up with 50 questions or more. When a dashboard with 50 questions loads, it sends 50 simultaneous requests asking for data. And depending on the size of that database and the number of tables in that database, it can be quite some time before those records return to answer all of those questions.

7 is an arbitrary number, there will of course be times where adding a lot of questions to a dashboard makes sense. but in general encourage your users to keep their dashboards focused on telling a story about your data with just a handful of questions. Think essays, or short stories, not books.

### Update your browser

Metabase is a web application, and can benefit from the latest and greatest versions of browsers like Firefox, Chrome, Edge, and Safari.

## Supported deployments

### Hosted Metabase

Metabase now offers a [hosted solution](https://www.metabase.com/hosting/), where we handle scaling Metabase for you. You'll still have to ensure your data sources are performant, but you'll no longer have to manage running the Metabase application.

### AWS Elastic Beanstalk

Check out our [guide to setting up Metabase on Elastic Beanstalk](running-metabase-on-elastic-beanstalk.md). We use Elastic Beanstalk to host our internal Metabase application.

### Heroku

See [running Metabase on Heroku](running-metabase-on-heroku.md).

### Docker & Kubernetes

See [running Metabase on Kubernetes](running-metabase-on-kubernetes).

### Other cloud providers

Google Cloud Platform, Microsoft Azure, Digital Ocean, and other cloud providers offer other great alternatives for hosting your Metabase application.