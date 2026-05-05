---
title: "Increase your e-commerce website reliability using chaos engineering and AWS Fault Injection Simulator"
url: "https://aws.amazon.com/blogs/devops/increase-e-commerce-reliability-using-chaos-engineering-with-aws-fault-injection-simulator/"
date: "Wed, 16 Jun 2021 22:42:44 +0000"
author: "Bastien Leblanc"
feed_url: "https://aws.amazon.com/blogs/devops/tag/aws-fault-injection-simulator/feed/"
---
<p>Customer experience is a key differentiator for retailers, and improving this experience comes through speed and reliability. An e-commerce website is one of the first applications customers use to interact with your brand.</p> 
<p>For a long time, testing an application has been the only way to battle-test an application before going live. Testing is very effective at identifying issues in an application, through processes like unit testing, regression testing, and performance testing. But this isn’t enough when you deploy a complex system such as an e-commerce website. Planning for unplanned events, circumstances, new deployment dependencies, and more is rarely covered by testing. That’s where chaos engineering plays its part.</p> 
<p>In this post, we discuss a basic implementation of chaos engineering for an e-commerce website using <a href="https://aws.amazon.com/fis/">AWS Fault Injection Simulator</a>.</p> 
<p><span id="more-9305"></span></p> 
<h2>Chaos engineering for retail</h2> 
<p>At AWS, we help you build applications following the <a href="https://aws.amazon.com/architecture/well-architected/">Well-Architected Framework</a>. Each pillar has a different importance for each customer, but the reliability pillar has consistently been valued as high priority by retailers for their e-commerce website.</p> 
<p>One of the recommendations of this pillar is to run <a href="https://wa.aws.amazon.com/wat.concept.gameday.en.html">game days</a> on your application.</p> 
<p>A game day simulates a failure or <a href="https://wa.aws.amazon.com/wat.concept.event.en.html">event </a>to test systems, processes, and team responses. The purpose is to perform the actions the team would perform as if an exceptional <a href="https://wa.aws.amazon.com/wat.concept.event.en.html">event</a> happened. These should be conducted regularly so that your team builds muscle <a href="https://wa.aws.amazon.com/wat.concept.memory.en.html">memory</a> of how to respond. Your game days should cover the areas of <a href="https://wa.aws.amazon.com/wat.pillar.operationalExcellence.en.html">operations</a>, <a href="https://wa.aws.amazon.com/wat.pillar.security.en.html">security</a>, <a href="https://wa.aws.amazon.com/wat.concept.c-reliability.en.html">reliability</a>, <a href="https://wa.aws.amazon.com/wat.pillar.performance.en.html">performance</a>, and <a href="https://wa.aws.amazon.com/wat.pillar.costOptimization.en.html">cost</a>.</p> 
<p>Chaos engineering is the practice of stressing an application in testing or production environments by creating disruptive events, such as a sudden increase in CPU or memory consumption, observing how the system responds, and implementing improvements. E-commerce websites have increased in complexity to the point that you need automated processes to detect the unknown unknowns.</p> 
<p>Let’s see how retailers can run game days, applying chaos engineering principles using AWS FIS.</p> 
<h2>Typical e-commerce components for chaos engineering</h2> 
<p>If we consider a typical e-commerce architecture, whether you have a monolith deployment, a well-known e-commerce software, or a microservices approach, all e-commerce websites contain critical components. The first task is to identify which components should be tested using chaos engineering.</p> 
<p>We advise you to consider specific criteria when choosing which components to prioritize for chaos engineering. From our experience, the first step is to look at your critical customer journey:</p> 
<ul> 
 <li>Homepage</li> 
 <li>Search</li> 
 <li>Recommendations and personalization</li> 
 <li>Basket and checkout</li> 
</ul> 
<p>From these critical components, consider focusing on the following:</p> 
<ul> 
 <li><strong>High and peak traffic</strong>: Some components have specific or unpredictable traffic, such as slots, promotions, and the homepage.</li> 
 <li><strong>Proven components</strong>: Some components have been tested and don’t have any existing issues. If the component isn’t tested, chaos engineering isn’t the right tool. You should return to unit testing, QA, stress testing, and performance testing and fix the known issues, then chaos engineering can help identify the unknown unknowns.</li> 
</ul> 
<p>The following are some real-world examples of relevant e-commerce services that are great chaos engineering candidates:</p> 
<ul> 
 <li><strong>Authentication</strong> – This is customer-facing because it’s part of every critical customer journey buying process</li> 
 <li><strong>Search</strong> – Used by most customers, search is often more important than catalog browsing</li> 
 <li><strong>Products</strong> – This is a critical component that is customer-facing</li> 
 <li><strong>Ads</strong> – Ads may not be critical, but have high or peak traffic</li> 
 <li><strong>Recommendations</strong> – A website without recommendations should still be 100% functional (to be checked with hypothesis during experiments), but without personal recommendations, a customer journey is greatly impacted</li> 
</ul> 
<h2>Solution overview</h2> 
<p>Let’s go through an example with a simplified recommendations service for an e-commerce application. The application is built with microservices, which is a typical target for chaos experiments. In a microservices architecture, unknown issues are potentially more frequent because of the distributed nature of the development. The following diagram illustrates our simplified architecture.</p> 
<div class="wp-caption aligncenter" id="attachment_9367" style="width: 731px;">
 <img alt="Recommendations Service architecture: ECS, DynamoDB, Personalize, SSM, Elasticsearch " class="wp-image-9367 size-full" height="497" src="https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2021/06/14/recommendations-1.png" width="721" />
 <p class="wp-caption-text" id="caption-attachment-9367">Recommendations Service Architecture</p>
</div> 
<p>&nbsp;</p> 
<p>Following the <a href="https://principlesofchaos.org/">principles of chaos engineering</a>, we define the following for each scenario:</p> 
<ul> 
 <li>A steady state</li> 
 <li>One or multiple hypothesis</li> 
 <li>One or multiple experimentations to test these hypotheses</li> 
</ul> 
<p>Defining a steady state is about knowing what “good” looks like for your application. In our recommendations example, steady state is measured as follows:</p> 
<ul> 
 <li>Customer latency at p90 between 0.3–0.5 seconds (end-to-end latency when asking for a recommendations)</li> 
 <li>A success rate of 100% at the 95 percentile</li> 
</ul> 
<p>For the sake of simplification of this article, we use a simplified version of a steady state than what is done in a real environment. You could go deeper by checking latency, for example (such as if the answer is fast but wrong). You could also analyze the metrics with an <a href="https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Anomaly_Detection.html">anomaly detection band</a> instead of fixed metrics.</p> 
<p>We could test the following situations and what should occur as a result:</p> 
<ul> 
 <li>What if <a href="https://aws.amazon.com/dynamodb/">Amazon DynamoDB</a> isn’t accessible from the recommendations engine? In this case, the recommendations engine should fall back to using <a href="https://aws.amazon.com/elasticsearch-service/">Amazon Elasticsearch</a> (Amazon ES) only.</li> 
 <li>What if <a href="https://aws.amazon.com/personalize/">Amazon Personalize</a> is slow to answer (over 2 seconds)? Recommendations should be served from a cache or reply with empty responses (which the front end should handle gracefully)</li> 
 <li>What if failures occur in <a href="http://aws.amazon.com/ecs">Amazon Elastic Container Service</a> (Amazon ECS), such as instances in the cluster failing or not being accessible? Scaling should kick in and continue serving customers.</li> 
</ul> 
<p>Chaos experiments run the hypotheses and check the outcomes. Initially, we run the experiments individually to avoid any confusion, but going forward we can run these experiments regularly and concurrently (for example, what happens if you introduce failing tasks on Amazon ECS and DynamoDB).</p> 
<h2>Create an experiment</h2> 
<p>We measure observability and metrics through X-Ray and Amazon CloudWatch metrics. The service is fronted by a load balancer so we can use the native CloudWatch metrics for the customer-facing state. Based on our definitions, we include the metrics that matter for our customer, as summarized in the following table.</p> 
<table style="border-color: #000000;"> 
 <thead> 
  <tr> 
   <th>Metric</th> 
   <th>Steady state</th> 
   <th>CloudWatch Namespace</th> 
   <th>Metric Name</th> 
  </tr> 
 </thead> 
 <tbody> 
  <tr> 
   <td style="text-align: center;">Latency</td> 
   <td style="text-align: center;">&lt; 0.5 seconds</td> 
   <td style="text-align: center;">AWS/X-Ray</td> 
   <td style="text-align: center;">ResponseTime</td> 
  </tr> 
  <tr> 
   <td style="text-align: center;">Success Rate</td> 
   <td style="text-align: center;">100% at 95 percentile</td> 
   <td style="text-align: center;">AWS/X-Ray</td> 
   <td style="text-align: center;">OkRate</td> 
  </tr> 
 </tbody> 
</table> 
<p>Now that we have ways to measure a steady state, we implement the hypothesis and experiments in AWS FIS. For this post, we test what happens if failures occur in Amazon ECS.</p> 
<p>We use the action <code>aws:ecs:drain-container-instances</code>, which targets the cluster running the relevant task.</p> 
<p>Let’s aim for 20% of instances that are impacted by the experiment. You should modify this percentage based on your environment, striking a balance between enough disturbance without failing the entire service.</p> 
<p>1. On the AWS FIS console, choose Create experiment template to start creating your experiment.</p> 
<p><img alt="FIS Home page -&gt; create experiment template" class="alignnone wp-image-9308 size-full" height="401" src="https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2021/06/11/1.jpg" width="1308" /></p> 
<p>Configure the experiment with an action <code>aws:ecs:drain-container-instances</code></p> 
<div class="wp-caption aligncenter" id="attachment_9372" style="width: 859px;">
 <img alt="add action for experiment, drainage 30%, duration: 600sec" class="wp-image-9372 size-full" height="779" src="https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2021/06/14/Capture1.png" width="849" />
 <p class="wp-caption-text" id="caption-attachment-9372">Setting up the experiment action using ECS drain instances</p>
</div> 
<p>Configure the targeted ECS cluster(s) you want to include in your chaos experiment, we recommend to use tags to easily target a component without changing the experiment again.</p> 
<div class="wp-caption aligncenter" id="attachment_9373" style="width: 855px;">
 <img alt="set target as resource tag, key=chaos, value=true" class="wp-image-9373 size-full" height="812" src="https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2021/06/14/Capture2.png" width="845" />
 <p class="wp-caption-text" id="caption-attachment-9373">Definition target for the chaos experiment</p>
</div> 
<p>Before running an experiment, we have to define the stop conditions. It’s usually a combination of multiple CloudWatch alarms, which could be a manual stop (a specific alarm that can be set to the ALARM state to stop the experiment), but more importantly alarms on business metrics that you define as criteria for the applications to serve your customers. For an e-commerce website, this could be the following:</p> 
<ul> 
 <li>Error rate over 5%</li> 
 <li>Search errors over x%</li> 
 <li>Order or minimum decreased by more than 15% than baseline (for more information about using the anomaly detection prediction band for business metrics, see <a href="https://aws.amazon.com/blogs/mt/how-to-set-up-cloudwatch-anomaly-detection-to-set-dynamic-alarms-automate-actions-and-drive-online-sales/">How to set up CloudWatch Anomaly Detection to set dynamic alarms, automate actions, and drive online sales</a>)</li> 
</ul> 
<p>For this post, we focus on error rate.</p> 
<p>2. Create a CloudWatch alarm for error rate on the service.</p> 
<p><img alt="CW graphs : X-ray responsetime to p50 and a second one on p90" class="alignnone wp-image-9309 size-full" height="301" src="https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2021/06/11/2.jpg" width="1076" /></p> 
<p><img alt="Clouwatch alarm conditions : static, greater than 0.5" class="alignright wp-image-9310 size-full" height="553" src="https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2021/06/11/3.jpg" width="801" /></p> 
<p>3. Configure this alarm in AWS FIS as a stop condition.</p> 
<p><img alt="FIS Stop conditions = RecommendationResponseTime" class="aligncenter wp-image-9311 size-full" height="365" src="https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2021/06/11/4.jpg" width="706" /></p> 
<h2>Run the experiment</h2> 
<p>We’re now ready to run the experiment. Let’s generate some load on the e-commerce website and see how it copes before and after the experiment. For the purpose of this post, we assume we’re running in a performance or QA environment without actual customers, so the load generated should be representative of the typical load on the application.</p> 
<p>In our example, we ingest the load using the open-source tool <a href="https://github.com/tsenart/vegeta">vegeta</a>. Some general load is generated using a command similar to the following:</p> 
<pre><code class="lang-bash">echo "GET http://xxxxx.elb.amazonaws.com/recommendations?userID=aaa&amp;amp;currentItemID=&amp;amp;numResults=12&amp;amp;feature=home_product_recs&amp;amp;fullyQualifyImageUrls=1" | vegeta attack -rate=5 -duration 0 &amp;gt; add-results.bin</code></pre> 
<p>We created a dedicated CloudWatch dashboard to monitor how the recommendations service is serving customer workload. The steady state looks like the following screenshot.</p> 
<p><img alt="Dashboard - steady state" class="alignnone wp-image-9312 size-full" height="792" src="https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2021/06/11/5.jpg" width="1610" /></p> 
<p>The p90 latency is under 0.5 seconds, p90 of success is greater than x% , the number of requests varies, but the response time is steady.</p> 
<p>Now let’s start the experiment on AWS FIS console.</p> 
<p><img alt="FIS - start the experiment" class="alignnone wp-image-9313 size-full" height="639" src="https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2021/06/11/6.jpg" width="1579" /></p> 
<p>After a few minutes, let’s check how the recommendations service is running.</p> 
<p><img alt="Dashboard - 1st experiment, Responsetime &lt; SLA, CPU at 80%" class="alignnone wp-image-9314 size-full" height="772" src="https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2021/06/11/7.jpg" width="1599" /></p> 
<p>The number of tasks running on the ECS cluster has decreased as expected, but the service has enough room to avoid any issue due to losing part of the ECS cluster. However, the average CPU usage starts to go over 80%, so we can suspect that we’re close to saturation.</p> 
<p>AWS FIS helped us prove that even with some degradation in the ECS cluster, the service-level agreement was still met.</p> 
<p>But what if we increase the impact of the disruption and confirm this CPU saturation assumption? Let’s run the same experiment with more instances drained from the ECS cluster and observe our metrics.</p> 
<p><img alt="Dashboard - breached SLA on response time, 100% CPU" class="alignright wp-image-9370 size-full" height="777" src="https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2021/06/14/8-1.jpg" width="1601" /></p> 
<p>With less capacity available, the response time has largely exceeded the SLA, and we have reached the limit of the architecture. We would recommend to explore optimizing the architecture with concepts like auto scaling, or <a href="https://wa.aws.amazon.com/wellarchitected/2020-07-02T19-33-23/wat.concept.cache.en.html">caching.</a></p> 
<h2>Going further</h2> 
<p>Now that we have a simple chaos experiment up and running, what are the next steps? One way of expanding on this is by increasing the number of hypotheses.</p> 
<p>As a second hypothesis, we suggest adding network latency to the application. Network latency, especially for a distributed application, is a very interesting use case for chaos engineering. It’s not easy to test manually, and often applications are designed with a “perfect” network mindset. We use the action<code> arn:aws:ssm:send-command/AWSFIS-Run-Network-Latency</code> to target the instances running our application.</p> 
<p>For more information about actions, see <a href="https://docs.aws.amazon.com/fis/latest/userguide/actions-ssm-agent.html">SSM Agent for AWS FIS actions</a>.</p> 
<p>However, having only technical metrics (such as latency and success code) lacks a customer-centric view. When running an e-commerce website, customer experience matters. Think about how your customers are using your website and how to measure the actual outcome for a customer.</p> 
<h2>Conclusion</h2> 
<p>In this post, we covered a basic implementation of chaos engineering for an e-commerce website using AWS FIS. For more information about chaos engineering, see <a href="https://principlesofchaos.org/">Principles of Chaos Engineering</a>.</p> 
<p>Amazon Fault Injection Simulator is now generally available, you can use it to run chaos experiments today. <a href="https://aws.amazon.com/fis">Click here to learn more</a></p> 
<p>To go beyond these first steps, you should consider increasing the number of experiments in your application, targeting crucial elements, starting with your development and environments and moving gradually to run experiments in production.</p> 
<p>&nbsp;</p> 
<h3>Author bio</h3> 
<p><img alt="Bastien Leblanc - Profile Photo" class="alignleft wp-image-9323 size-full" height="240" src="https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2021/06/11/bastien-HD-1.jpeg" width="160" />Bastien Leblanc is the AWS Retail technical lead for EMEA. He works with retailers focusing on delivering exceptional end-user experience using AWS Services. With a strong background in data and analytics he helps retailers transform their business with cutting-edge AWS technologies.</p>
