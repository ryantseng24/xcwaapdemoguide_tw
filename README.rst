==================================================

.. contents:: Table of Contents


目標
####################
使用本指南和提供的示範應用程式和工具，探索 F5 Distributed Cloud (XC)的 Web 應用程式和 API 保護功能 (WAAP)。這將幫助您入門 WAAP 的幾個使用案例：

- Web 應用程式防火牆 (WAF)
- API 發現和保護
- 負載平衡器服務策略
- 機器人緩解
- DoS/DDoS 緩解
  
情境
####################
我們將以一個典型的客戶應用情境為例：一個星級評分應用程式。這個應用程式允許用戶對各種物品（例如電子商務產品或客戶服務）進行評分和評論。該應用程式運行在 Kubernetes 容器平台上，可以部署在各種雲端環境中。然而，為了本次演示指南的目的，我們將使用預先在 XC 虛擬 Kubernetes（vK8s）上部署的應用程式，並使用 XC WAAP 進行保護。

情境架構
#######################
F5 XC WAAP 是一套基於 SaaS 的安全服務，為分佈式應用程式服務提供全面的保護。在我們的情境中，應用程式服務部署在不同 XC vK8s 區域的多個 vK8s pod 上，由 F5 全球 PoP 提供服務，以表示分散的工作負載；同樣的模型適用於在任何雲端上進行類似部署。

這個實驗的目的是透過在 Docker 容器中部署一個典型的範例應用程式，再透過 XC 服務進行管理，以自給自足的方式快速熟悉 XC 平台。同時還包括一個單獨特制的"實用工具"服務，該服務提供生成模擬用戶流量和攻擊（如 WAF 或機器人）所需的工具，以幫助說明不同的 WAAP 使用情境。

*Docker 容器應用程式*: 包含了 Star Ratings 應用程式，由一個簡單的後端服務組成，該服務公開了一個 API，以及一個使用該 API 的前端。

*測試工具*: 基於 Web 的服務，包含了專門用於測試 **部署的示範應用程式** 狀態的腳本和工具。

**注意：此工具僅用於此WAAP演示指南，僅接受包含有效的 Star Ratings 示範應用程式部署的 URL，並且此部署必須托管在 F5 Distributed Cloud 上（主機名以'.ac.vh.ves.io'結尾）。此工具無法用於在此演示指南以外的任何其他網站/網頁應用程式上運行測試。**

在這個指南的範圍內，測試工具可用於啟動/停止 WAF 測試、機器人測試，以及檢查 WAAP / Bot Defense 的保護狀態。

.. figure:: assets/waap_overview.png

登入 F5 Distributed Cloud 控制台
##########################################
1. 一旦您啟動了 UDF 部署，就會觸發一個工作流程，為您在 f5-sales-demo 租戶中創建一個用戶帳戶。您應該已經收到一封電子郵件，要求您設定此帳戶的密碼。按照電子郵件中的說明設置您帳戶的密碼。

2. 如果系統要求您輸入 XC 租戶域名，請輸入f5-sales-demo並點擊 **Next** 。

.. figure:: assets/xc-domain.png
   :width: 600px

3. 使用您的電子郵件和剛剛設定的密碼登入：

.. figure:: assets/xc-login.png
   :width: 600px

4. 如果系統要求，請查看並接受 **Terms of Service** 和 **Privacy Policy** 。

5. 當要求您識別自己時，選中所有核取方塊，然後點擊 **Next** 。

6. 點擊 **Advanced** ，然後點擊 **Get Started** 。

7. 一旦您成功登入租戶，導航到 **Multi-Cloud App Connect** 。

8. 在 URL 中，您將找到為您隨機生成的 Namespace：

.. figure:: assets/xc-namespace.png
   :width: 800px

9. 記下上述 Namespace，因為您將在隨後的步驟中需要它。

設置 HTTP 負載平衡器
******************************

接下來，我們需要通過配置我們應用程式的 HTTP 負載平衡設置，使我們的示範應用程式工作負載可訪問。我們將為服務創建一個源池。源池包括端點、叢集、路由和宣告策略，這些都是發布應用程式至網際網路所需的元素。

返回到 F5 Distributed Cloud 控制台，導航到服務選單中的 **Multi-Cloud App Connect** 服務。

.. figure:: assets/load_balancer_navigate.png
   :width: 600px

選擇 **HTTP Load Balancers**。

.. figure:: assets/load_balancer_navigate_menu.png
   :width: 500px

點擊 **Add HTTP Load Balancer** 按鈕以打開 HTTP 負載平衡器創建表單。

.. figure:: assets/load_balancer_create_click.png
   :width: 600px

接著輸入負載平衡器的名稱。

.. figure:: assets/httplb_set_name.png

接下來，我們需要為我們的工作負載提供一個域名：域名可以委派給 F5，以便可以快速創建域名服務（DNS）紀錄，加速部署和路由流量到我們的工作負載。在這個演示中，我們指定 **star-ratings-(您的學生編號).sales-demo.f5demos.com** 。

委派的域名已事先設定好，您可以直接使用 **Automatically Manage DNS Records** 。

.. figure:: assets/httplb_set_domain.png

之後，讓我們創建一個新的源池，它將用於我們的負載平衡器。源池是將一組端點配置為一個資源池，該資源池用於負載平衡器配置。點擊 **Add Item** 以打開源池創建表單。

.. figure:: assets/httplb_pool_add.png

然後打開下拉選單，點擊 **Add Item** 。

.. figure:: assets/httplb_pool_add_create.png

首先，讓我們給這個池一個名稱。

.. figure:: assets/httplb_pool_name.png

現在點擊 **Add Item** 以開始新增一個源站伺服器

.. figure:: assets/httplb_pool_origin_add.png

Let's now configure origin server. First open the drop-down menu to specify the type of origin server. For this demo select **K8s Service Name of Origin Server on given Sites**. 
Then specify service name indicating the service we deployed in the corresponding namespace. Please note that it follows the format of **servicename.namespace**. We use **star-ratings-app.yournamespace** for this demo where **yournamespace** is the name of your namespace. 
After that we need to select the **Virtual Site** type and select **shared/ves-io-all-res**. 
Finally, the last step to configure the origin server is specifying network on the site. Select **vK8s Network on Site**.
完成後，點擊 **Apply** 。

.. figure:: assets/httplb_pool_origin_configure.png

Next we need to configure the port (the end point service/workload available on this port). In this demo it's Port **8080**.

.. figure:: assets/httplb_pool_port.png

Then just click **Continue** to move on.

.. figure:: assets/httplb_pool_continue.png

Once done, click **Apply** to apply the origin pool to the load balancer configuration. This will return to the load balancer configuration form.

.. figure:: assets/httplb_pool_confirm.png

Take a look at the load balancer configuration and finish creating it by clicking **Save and Exit**.

.. figure:: assets/httplb_save_and_exit.png

We will need a CNAME record in order to generate traffic and to run attacks on our app. For the purposes of this guide, you can use the generated CNAME value as shown in the image below. However, should you want to use your own domain, you can; there are several ways that you can delegate your DNS domain to F5 Distributed Cloud Services. A reference on how to do so is here:  `Domain Delegation <https://docs.cloud.f5.com/docs/how-to/app-networking/domain-delegation>`_.

.. figure:: assets/httplb_cname.png

Now let's open the website to see if it's working. You can use CNAME or your domain name to do that.

.. figure:: assets/website.png

Great, your sample app should be live and you should be ready to go through the WAAP use-cases.

WAAP Use-Case Demos
####################

At this point, whether you used the manual approach in *PATH 1* or Ansible automation in *PATH 2*, you should have a working sample app. You can now start running through the WAAP use-cases. Again, you can follow the steps below to proceed with the use-cases manually, or you may choose to use corresponding sections in the Ansible guide to automate what's done manually. 

APP Protection
**************

A skilled attacker will use automation and multiple tools to explore various attack vectors. From simple Cross-Site Scripting (XSS) that leads to website defacement to more complex zero-day vulnerabilities, the range of attacks continues to expand and there isn’t always a signature to match!

The combination of signatures, threat intelligence, behavioral analysis, and machine learning capabilities built into F5 Distributed Cloud WAF enables detection of known attacks and mitigation of emerging threats from potentially malicious users. This provides effective and easy-to-operate security for apps delivered across clouds and architectures.

In the **App Protection** use-case we will see how easy it is to create an effective WAF policy using F5’s Distributed Cloud to quickly secure our application front-end. We already have user traffic of our sample app flowing through the HTTP Load Balancer within F5 Distributed Cloud, routing requests to the app instance(s) running in Amazon AWS. To protect this traffic, we will edit the HTTP Load Balancer we created earlier by configuring App Firewall. 

First, let's test our app and see if it's vulnerable to attacks. For that we are going to use Test Tool which sends attacks to the apps based on its CNAME. 

Follow the link `<https://test-tool.sr.f5-cloud-demo.com>`_, then paste the CNAME copied one step before and click **SEND ATTACKS**. In the box under it you will see attack types and site status - our app is vulnerable to them. Now let's go ahead and protect the app by creating and configuring Firewall. Then we will test the app once again to see the result of protection.

.. figure:: assets/test_waf_1.png

Back in the F5 Distributed Cloud Console, open the service menu and navigate to **Web App & API Protection**. 

.. figure:: assets/waf_navigate.png
   :width: 600px

Then proceed to the **HTTP Load Balancers** section.

.. figure:: assets/waf_navigate_menu.png
   :width: 500px

Open HTTP Load Balancer properties and select **Manage Configuration**.

.. figure:: assets/httplb_popup.png
   :width: 850px

Click **Edit Configuration** in the right top corner to start editing the HTTP load balancer. 

.. figure:: assets/httplb_edit.png

In the **Web Application Firewall** section first enable **App Firewall** in the drop-down menu, and then click **Add Item** to configure a new WAF object.

.. figure:: assets/waf_create.png

First, give the Firewall a name.

.. figure:: assets/waf_name.png

Then specify enforcement mode in the dropdown menu. The default is **Monitoring**, meaning that the Distributed Cloud WAF service won't block any traffic but will alert on any request that is found to be violating the WAF policy. **Blocking** mode means that the Distributed Cloud WAF will take mitigation action on offending traffic. Select the **Blocking** mode option.

.. figure:: assets/waf_enforcement_mode.png

Next, we will specify detection settings. Default settings are recommended for mitigating malicious traffic with a low false positive rate. But we will select **Custom** detection settings, in order to override and customize preset policy detection defaults. 

.. figure:: assets/waf_detection_custom.png

Select **Custom** attack type in the drop-down menu and proceed to specifying **Disabled Attack Types**. Select **Command Execution** attack type. Command execution is an attack against web applications that targets Operating System commands to gain access to it. 

.. figure:: assets/waf_attack_types.png

The next property **Signature Selection by Accuracy** allows us to disable some attack types and use different signature sets for optimal accuracy. For this demo let's enable **High, Medium and Low** accuracy signatures.

.. figure:: assets/waf_signature.png

After that we will edit Disabled Violations list. This enables detection of various violation types like malformed data and illegal filetypes. For this use-case, we will select **Custom** violations, and then specify **Bad HTTP Version**. 

.. figure:: assets/waf_violatations.png

Next we will specify blocking response page. To do that, select **Custom** and indicate **403 Forbidden** as response code. By default the Distributed Cloud WAF looks for specific query parameters like "card" or "password" to prevent potentially sensitive information such as account credentials or credit card numbers from appearing in security logs. This can be customized through a Blocking Response Page that can include a custom body in ASCII or base64.

.. figure:: assets/waf_adv_config.png

Now that we’re done with all the settings, just click **Continue**.

.. figure:: assets/waf_continue.png

Click **Save and Exit** to save the HTTP Load Balancer settings.

.. figure:: assets/waf_save_lb.png

Now we are ready to test and see if our app is still vulnerable to the attacks. Follow the link  `<https://test-tool.sr.f5-cloud-demo.com>`_, and click **SEND ATTACKS**. In the box under it you will see attack types and their statuses - they are now all blocked and our app is safe. 

.. figure:: assets/test_waf_2.png

Next let’s look at some of the visibility and security insights provided by F5 Distributed Cloud WAAP. Navigate to **Dashboards**, select **Security Dashboard** and click on our load balancer.

.. figure:: assets/waf_dashboard_navigate.png

Here we will see app dashboard. The dashboard provides detailed info on all the security events, including location, policy rules hit, malicious users, top attack types and other crucial information collected through F5 Distributed Cloud WAAP correlated insights.

.. figure:: assets/waf_dashboard_events.png

Now navigate to **Security Events** and then open one of the security events to drill into it. 

.. figure:: assets/waf_requests.png

Let’s look at the specifics of the **Java code injection** attack. Here we can not only see its time, origin and src IP, but also drill down to see very detailed information.

.. figure:: assets/waf_request_details.png

After having a look at the attack, it is possible to block the client. To do that, open the menu and select **Add to Blocked Clients**. 

.. figure:: assets/waf_block_options.png

F5 Distributed Cloud WAF provides security through Malicious User Detection as well. Malicious User Detection helps identify and rank suspicious (or potentially malicious) users. Security teams are often burdened with alert fatigue, long investigation times, missed attacks, and false positives. Retrospective security through Malicious User Detection allows security teams to filter noise and to identify actual risks and threats through actionable intelligence, without manual intervention.

WAF rules hit, forbidden access attempts, login failures, request and error rates -- all create a timeline of events that can suggest malicious activity. Users exhibiting suspicious behavior can be automatically blocked, and exceptions can be made through allow lists.

The screenshot below represents how the malicious user can look like.

.. figure:: assets/waf_malicious_user.png


API Protection 
**************

Protecting API resources is a critical piece of a holistic application security strategy. API Security helps us analyze and baseline normal levels of traffic, response rates, sizes and data being shared via APIs. 

Without API protection all traffic goes directly to the server and can be harmful. Let's take a look at an attack on our sample app and then protect its API.

Go back to the Test Tool  `<https://test-tool.sr.f5-cloud-demo.com>`_, and switch to the **API Security in Action** tab. Then click **SEND ATTACKS**. In the box under it you will see the status which shows that API is vulnerable. Now let's go ahead and protect API.

.. figure:: assets/test_api_1.png

Distributed Cloud API Security helps protect API resources based on an Open API specification, typically captured in a Swagger file. The API Security service supports the upload of an Open API specification file, which contains API routes that can be protected by the Web App Firewall, as well as methods that can be enabled and disabled. 

To start API protection configuration, go back to the F5 Distributed Cloud Console, select **Swagger Files** and click **Add Swagger File**. 

.. figure:: assets/swagger_navigate.png

Give swagger file a name and then upload it. Once it's uploaded, click **Save and Exit**.
   
.. figure:: assets/swagger_upload_file.png

Now over to creating API Definition. Navigate to **API Definition** and then click the **Add API Definition** button.

.. figure:: assets/api_definition_navigate.png

Enter a name in the metadata section. Then go to **Swagger Specs** section and open the drop-down menu. Select the swagger spec added earlier, then click **Save and Exit** to create API definition object.

.. figure:: assets/api_definition_create.png

Now we need to attach the created API definition to our HTTP load balancer. Navigate to **Load Balancers** and select **HTTP Load Balancers**. The HTTP Load Balancer we created earlier will appear. Open its menu and select **Manage Configuration**.

.. figure:: assets/api_definition_lb_popup.png

Click **Edit Configuration** to start editing.

.. figure:: assets/api_definition_lb_edit.png

In the **API Protection** section enable **API Definition** and then select the API Definition created earlier. 

.. figure:: assets/api_definition_select_api_def.png

Now we need to a create a new Service Policy with a set of Custom Rules that will specify either an Allow or Deny rule action for specific API resources contained in our Swagger file. This approach uses the combination of Service Policies and Custom Rules to fine-tune and provide granular control over how our application API resources are protected.

Scroll to the **Common Security Controls** section and select **Apply Specified Service Policies**. Then click **Configure**. 

.. figure:: assets/api_definition_policy.png

Click on the **Select Item** field and select **Add Item** option.

.. figure:: assets/api_definition_policy_create.png

Enter a name for the policy in the metadata section and go to the **Rules** section. Select **Custom Rule List** and click **Configure**.

.. figure:: assets/api_definition_policy_create_rules.png

Let's now add rules: click **Add Item**.
   
.. figure:: assets/api_definition_rule_add.png

The first rule will deny all except the API. Enter a name in the metadata section and scroll down. 

.. figure:: assets/api_definition_rule_add_details.png

Next configure HTTP Path. Click **Configure** in the **HTTP Path** section.

.. figure:: assets/api_definition_rules_path.png

And fill in the path - **/api/v1/** for this demo. Then click **Apply**.

.. figure:: assets/api_definition_rules_prefix.png

Scroll down to **Advanced Match** section and click **Configure** for the API Group Matcher field.

.. figure:: assets/api_definition_rules_api_matcher.png

In the API Group Matcher screen, select an exact value. 

.. figure:: assets/api_definition_rules_matcher_select_api_def.png

Tick the **Invert String Matcher** option and click **Apply** to add the matcher. 

.. figure:: assets/api_definition_matcher_tick.png

 Click another **Apply** to add the rule specification. 

.. figure:: assets/api_definition_policy_apply.png

Click **Apply** to add the rule.

.. figure:: assets/api_definition_add_rule.png

Create one more rule to 'allow-other' using the **Add Item** option in the rules section. 

.. figure:: assets/api_definition_second_rule.png

First, enter a name in the metadata section.
   
.. figure:: assets/api_definition_second_rule_details.png

Next, select **Allow** for Action field in the Action section.

.. figure:: assets/api_definition_second_rule_allow.png

Click **Apply** to add the rule specification.

.. figure:: assets/api_definition_second_rule_apply.png

Click **Apply** to add the second rule.

.. figure:: assets/api_definition_second_rule_add.png

Take a look at the rules created and click **Apply**. 

.. figure:: assets/api_definition_rule_list_apply.png

Click **Continue** to add the service policy to the load balancer and then **Apply**.

.. figure:: assets/api_definition_continue.png

.. figure:: assets/api_definition_def_policy_apply.png

The last step is to look the configuration through and save the edited HTTP load balancer. Once you click **Save and Exit** at the end, the Load Balancer will update with the API security settings and our API resources will be protected!

.. figure:: assets/api_definition_lb_save.png

Well done! The API of our sample Rating App is protected based on the spec in the uploaded Swagger file. Let's try and see that the access is forbidden.

Go back to the Test Tool  `<https://test-tool.sr.f5-cloud-demo.com>`_, and click **SEND ATTACKS**. In the box under it we will see **protected** status, so our API is safe now.  

.. figure:: assets/test_api_2.png

In cases where API specifications are not known or well documented, the F5 Distributed Cloud API Security provides a machine learning (ML)-based, dynamic API Discovery service.

API Discovery analyzes traffic that flows to and from API endpoints and constructs a visual graph to detail API path relationships. It may be difficult for an organization to keep track of APIs, as they typically change frequently. Over time F5 Distributed Cloud can baseline normal API behavior, usage, and methods, detecting anomalies and helping organization detect shadow APIs that bring unintended risk.

In the screenshot below we can see the percent of requests, learned schema for a specific endpoint, and even download an automatically-generated Swagger file based on discovered APIs.

.. figure:: assets/api_auto_discovery.png 

Bot Protection
**************

F5 Distributed Cloud Bot Defense helps us identify attacks and allow us then to easily block them! Our sample rating app could definitely benefit from Distributed Cloud Bot Defense. So let’s see how easy it actually is to set up and use the service!

First let's generate some bot traffic to our app. Go back to the Test Tool  `<https://test-tool.sr.f5-cloud-demo.com>`_, and switch to the **Bot Defense in Action** tab. Click **GENERATE BOT TRAFFIC**. In the box under it we will see that all the bot traffic passed. Now let's go ahead and block it by setting up a resilient anti-automation solution that will be attached to the HTTP Load Balancer that processes the traffic to our app. We will then test it to see how Bot Protection works.

.. figure:: assets/test_bot_1.png

Navigate to **HTTP Load Balancers**, open the menu of the load balancer we created earlier and select **Manage Configuration**.

.. figure:: assets/bot_lb_popup.png

Click **Edit Configuration** to start editing the load balancer.

.. figure:: assets/bot_lb_edit.png

Go to the **Bot Protection** section and enable Bot Defense. The Regional Endpoint is **US** due to its closer proximity to our sample app user base. Click **Configure** to configure Bot Defense Policy.

.. figure:: assets/bot_config.png

Next, we need to configure an App Endpoint, which will contain the policies and actions to protect the specific resource in our app that’s used for adding ratings. To do that click **Configure**.

.. figure:: assets/bot_config_endpoint.png

Click **Add Item** to start adding an endpoint.

.. figure:: assets/bot_config_endpoint_add.png

Name the endpoint and then select HTTP Methods. Let's pick **PUT** and **POST** for this demo. Scroll down and fill in the path - **/api/v1/**.
Then set Bot Traffic Mitigation options to **Block** action for identified bot traffic, and select **403 Forbidden** status. 
Go ahead and click **Apply** to complete the App Endpoint setup.

.. figure:: assets/bot_full_config.png

We’ve just defined the policy to protect our vulnerable Rating app resource with Bot Defense enabled. Now, click **Apply** to confirm.

.. figure:: assets/bot_endpoint_apply.png

Click **Apply** to apply the configured Bot Defense Policy.

.. figure:: assets/bot_config_apply.png

To complete the configuration of load balancer, click **Save and Exit**.

.. figure:: assets/bot_lb_save.png

Now we can test and see the end-result of our setup. Go back to the Test Tool  `<https://test-tool.sr.f5-cloud-demo.com>`_, and click **GENERATE BOT TRAFFIC**. This time we will see **blocked** status.  

.. figure:: assets/test_bot_2.png

Now let’s have a look at the Security analytics for the HTTP Load Balancer where we configured Bot Defense. Navigate to **Dashboards**, then **Security Dashboard** and click on the load balancer name.

.. figure:: assets/bot_dashboard_0.png

Navigate to the **Bot Defense** tab. Here we will see key info breaking down: which bots are making the most malicious requests, which endpoints are attacked the most, and which automation types are being used the most. 

.. figure:: assets/bot_dashboard_1.png

Then move on to the **Security Events** tab. Here we can go into detail on the HTTP Load Balancer traffic from the point of view of Bot traffic analytics. From transactions per minute for a specified timeframe, to detail of every HTTP request with inference of whether it is a legitimate user or automation traffic.

.. figure:: assets/bot_dashboard_2.png


DDoS Protection
***************

F5 Distributed Cloud WAAP is monitoring traffic and is able to identify multiple types of security events, including DDoS attacks directed towards our application as DDoS Security Events. This provides critical intelligence of your app security at your fingertips.

In this demo we will configure DDoS protection by specifying IP Reputation and rate limiting for the sample app. Then we will add DDoS mitigation rule to block users by IP source defining expiration timestamp. 

Navigate to **HTTP Load Balancers**, open the menu of the load balancer we created earlier and select **Manage Configuration**. 

.. figure:: assets/ddos_lb_popup.png

Click **Edit Configuration** to start editing the load balancer.

.. figure:: assets/ddos_lb_edit.png


In the **Common Security Controls** section enable **IP Reputation** and choose IP threat categories. We select **Spam Sources, Denial of service, Anonymous Proxies, Tor Proxy** and **Botnets** for this demo.

.. figure:: assets/ddos_ip_reputation.png

In order to configure rate limiting, select **Custom Rate Limiting Parameters** in the drop-down menu of rate limiting and click **View Configuration**.

.. figure:: assets/ddos_rate_limiting_select.png

First specify number, then burst multiplier. For this use-case we specify **10** and **5** respectively. Click **Apply** to proceed. 

.. figure:: assets/ddos_rate_limit_config.png

In the **DoS Protection** section enable DDoS detection in the drop-down menu and click **Configure** to add a new rule.

.. figure:: assets/ddos_detection.png


Next click the **Add Item** button to open the form where we will create an ‘IP Source’ mitigation rule.

.. figure:: assets/ddos_mitigation_add.png

Give rule a name, specify IP we want to block - **203.0.113.0/24** and indicate the expiration time stamp. Finally, click the **Apply** which will create our DDoS Mitigation rule.

.. figure:: assets/ddos_mitigation_rule.png

Click **Apply** to apply the rule we've created.

.. figure:: assets/ddos_mitigation_rule_apply.png

And finally we need to click **Save and Exit** to save these changes and allow the F5 Distributed Cloud WAF engine to start enforcing our newly created DDoS Mitigation rule and blocking the malicious IP.

.. figure:: assets/ddos_save_lb.png

See how easy that was! This should definitely help with the performance and uptime of our application!

We have created the service policy to block that malicious IP. Now let’s have a look at the reporting and analytics for the HTTP Load Balancer where we configured the policy for our app. 

Here we can see all of our app's critical security info in one place. Take a look at the **Security Events** section in the screenshot below showing all the events including the DDoS ones. Notice in the **DDoS Security Events** section we now see blocked traffic from the IP address we specified earlier. We can also see the map of security events giving clear visual security event distribution.

.. figure:: assets/ddos_demo_1.png

In the screenshot below you can see the analytics for our simulated traffic and attacks. See the impact of attacks on each endpoint by hovering over an endpoint on the map. We can also get insights into Top IPs, Regions, ASNs and TLS FPs. 

.. figure:: assets/ddos_demo_2.png

Wrap-Up
#######

At this stage you should have set up a sample app and sent traffic to it. You've configured and applied F5 Distributed Cloud WAAP services in order to protect both the Web & API of the app from malicious actors & bots. We also looked at the telemetry and insights from the data in the various Dashboards & security events.

We hope you have a better understanding of the F5 Distributed Cloud WAAP services and are now ready to implement it for your own organization. Should you have any issues or questions, please feel free to raise them via GitHub. Thank you!
