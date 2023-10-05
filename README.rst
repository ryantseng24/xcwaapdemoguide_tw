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

現在讓我們配置源伺服器。首先打開下拉選單，指定源伺服器的類型。對於這個 LAB，請選擇 **Public IP of Origin Server**。
然後，指定源站 IP 名稱 **54.157.200.74** (已提前部建好的應用服務)。
完成後，點擊 **Apply** 。

.. figure:: assets/httplb_pool_origin_configure2.png

接下來，我們需要配置源站服務監聽的埠號。在這個 LAB 中，請使用 8080 埠。

.. figure:: assets/httplb_pool_port.png

然後只需點擊 **Continue** 以繼續。

.. figure:: assets/httplb_pool_continue.png

完成後，點擊 **Apply** 以將源池應用於負載平衡器配置。這將返回到負載平衡器配置表單。

.. figure:: assets/httplb_pool_confirm.png

查看負載平衡器配置，然後點擊 **Save and Exit** 以完成創建。

.. figure:: assets/httplb_save_and_exit.png

為了生成流量並對我們的應用程式進行攻擊，我們需要一個可以透過網際網路存取服務的 FQDN 或是 IP。對於本指南的目的，您可以使用如下圖所示的生成的 CNAME 值。

.. figure:: assets/httplb_cname.png

現在讓我們打開網站來檢查它是否正常運作。您可以使用 CNAME 或您的域名來執行此操作。

.. figure:: assets/website.png

太好了，您的示範應用程式已經上線，您現在應該已經準備好進行 WAAP 使用案例的操作了。

WAAP Use-Case Demos
####################

在此階段，無論您是選擇在*路徑1*中使用手動方式，或在*路徑2*中使用Ansible自動化，您應該都已經有一個運作中的範例應用程式。您現在可以開始執行WAAP的使用案例。再次提醒，您可以選擇手動跟隨以下步驟進行這些使用案例，或者選擇在Ansible指南中使用對應的部分來自動執行手動完成的步驋。

應用程式保護
**************

熟練的攻擊者將使用自動化和多種工具來探索各種攻擊向量。從導致網站被篡改的簡單跨站腳本攻擊（XSS）到更複雜的零日漏洞，攻擊範圍持續擴大，並且並非所有攻擊都有對應的簽名！

F5分散式雲端WAF內置的簽名、威脅情報、行為分析和機器學習能力的結合，使其能夠檢測已知攻擊並緩解來自可能惡意用戶的新興威脅。這為跨雲端和架構提供的應用程式提供了有效且易於操作的安全性。

在**應用程式保護**使用案例中，我們將看到如何使用F5的分散式雲端來創建有效的WAF政策，快速保護我們的應用程式前端。我們已經有了我們範例應用程式的用戶流量，這些流量透過F5分散式雲端內的HTTP負載平衡器流動，將請求路由到在Amazon AWS中運行的應用程式實例。為了保護這些流量，我們將編輯我們早先創建的HTTP負載平衡器，並配置App Firewall。

首先，讓我們測試我們的應用程式，看看它是否容易受到攻擊。為此，我們將使用測試工具，該工具根據其CNAME向應用程式發送攻擊。

請按照以下連結 `<https://test-tool.sr.f5-cloud-demo.com>`_，然後粘貼一步之前複製的CNAME，並點擊 **發送攻擊**。在它下面的框中，你將看到攻擊類型和站點狀態--我們的應用程式對它們是脆弱的。現在讓我們開始保護應用程式，創建和配置防火牆。然後我們將再次測試應用程式，以查看保護的結果。

.. figure:: assets/test_waf_1.png

回到F5分散式雲端控制台，打開服務菜單並導航到**Web應用程式和API保護**。

.. figure:: assets/waf_navigate.png
   :width: 600px

然後前往**HTTP負載平衡器**部分。

.. figure:: assets/waf_navigate_menu.png
   :width: 500px

打開HTTP負載平衡器屬性並選擇**管理配置**。

.. figure:: assets/httplb_popup.png
   :width: 850px

在右上角點擊**編輯配置**以開始編輯HTTP負載平衡器。

.. figure:: assets/httplb_edit.png

在**Web應用程式防火牆**部分，首先在下拉菜單中啟用**App防火牆**，然後點擊**新增項目**以配置新的WAF對象。

.. figure:: assets/waf_create.png

首先，為防火牆取一個名稱。

.. figure:: assets/waf_name.png

然後在下拉菜單中指定強制模式。預設為**監控**，這意味著分散式雲端WAF服務不會阻擋任何流量，但會對任何被發現違反WAF政策的請求進行警告。**阻擋**模式意味著分散式雲端WAF將對觸犯的流量採取緩解行動。選擇**阻擋**模式選項。

.. figure:: assets/waf_enforcement_mode.png


接下來，我們將指定檢測設置。預設設置被推薦用於減輕惡意流量，並具有低假陽性率。但我們將選擇**自訂**檢測設置，以覆蓋和自訂預設的政策檢測預設值。

.. figure:: assets/waf_detection_custom.png

在下拉菜單中選擇**自訂**攻擊類型，然後進行指定**已禁用的攻擊類型**。選擇**命令執行**攻擊類型。命令執行是針對Web應用程式的攻擊，目標是操作系統命令以獲取對其的訪問。

.. figure:: assets/waf_attack_types.png

下一個屬性**按準確性選擇簽名**允許我們禁用一些攻擊類型並使用不同的簽名集合以獲得最佳準確性。對於這個演示，讓我們啟用**高，中和低**準確性的簽名。

.. figure:: assets/waf_signature.png

之後我們將編輯已禁用違規的列表。這可以檢測到各種類型的違規，如格式錯誤的數據和非法文件類型。對於這個使用案例，我們將選擇**自訂**違規，然後指定**錯誤的HTTP版本**。

.. figure:: assets/waf_violatations.png

接下來我們將指定阻擋響應頁面。要做到這一點，選擇**自訂**並指定**403 Forbidden**作為響應碼。預設情況下，分散式雲端WAF會尋找特定的查詢參數，如"卡"或"密碼"，以防止可能的敏感信息，如帳戶憑證或信用卡號碼出現在安全日誌中。這可以通過一個可以包含ASCII或base64的自訂體的阻擋響應頁面來自訂。

.. figure:: assets/waf_adv_config.png

現在我們已經完成所有設置，只需點擊繼續。

.. figure:: assets/waf_continue.png

點擊儲存並退出以儲存HTTP負載平衡器設置。

.. figure:: assets/waf_save_lb.png

現在我們已經準備好測試並查看我們的應用程式是否仍然容易受到攻擊。按照此鏈接 <https://test-tool.sr.f5-cloud-demo.com>_，並點擊發送攻擊。在其下方的框中，您將看到攻擊類型及其狀態 - 它們現在都被阻擋，我們的應用程式是安全的。

.. figure:: assets/test_waf_2.png

接下來，讓我們看看F5分散式雲端WAAP提供的一些可見性和安全洞察。導航到儀表板，選擇安全儀表板，然後點擊我們的負載平衡器。

.. figure:: assets/waf_dashboard_navigate.png

在這裡，我們將看到應用程式儀表板。該儀表板提供了所有安全事件的詳細信息，包括位置，政策規則命中，惡意用戶，主要攻擊類型以及通過F5分散式雲端WAAP相關洞察收集的其他關鍵信息。

.. figure:: assets/waf_dashboard_events.png

現在導航到安全事件，然後打開其中一個安全事件以深入了解。

.. figure:: assets/waf_requests.png

讓我們看看Java代碼注入攻擊的具體情況。在這裡，我們不僅可以看到其時間，起源和源IP，還可以深入查看非常詳細的信息。

.. figure:: assets/waf_request_details.png

在查看攻擊之後，可以阻止客戶端。要做到這一點，打開菜單並選擇添加到被阻擋的客戶端。

.. figure:: assets/waf_block_options.png

F5分散式雲端WAF也通過惡意用戶檢測提供安全性。惡意用戶檢測有助於識別和排名可疑（或可能惡意）的用戶。安全團隊經常被警報疲勞、長時間的調查、錯過的攻擊以及假陽性所困擾。通過惡意用戶檢測的回溯性安全允許安全團隊過濾噪音並通過可操作的情報識別實際風險和威脅，無需手動干預。

WAF規則命中，禁止訪問嘗試，登錄失敗，請求和錯誤率 -- 都創建了一個事件時間線，這可能表明存在惡意活動。表現出可疑行為的用戶可以被自動阻擋，並且可以通過允許列表進行例外處理。

下面的屏幕截圖表示惡意用戶可能的外觀。

.. figure:: assets/waf_malicious_user.png


API Protection 
**************

保護API資源是整體應用安全策略的關鍵部分。API安全性幫助我們分析並確定通過API共享的流量、響應率、大小和數據的正常水平。

如果沒有API保護，所有流量都會直接流向伺服器，這可能是有害的。讓我們來看看我們的示例應用的一次攻擊，然後保護其API。

回到測試工具 `<https://test-tool.sr.f5-cloud-demo.com>`_，並切換到 **API 安全實踐** 頁籤。然後點擊 **發送攻擊**。在它下面的框中，你將看到顯示API存在漏洞的狀態。現在讓我們去保護API。

.. figure:: assets/test_api_1.png

分散式雲API安全性基於Open API規範來幫助保護API資源，通常在Swagger文件中捕獲。API安全服務支持上傳Open API規範文件，該文件包含可以由Web應用防火牆保護的API路由，以及可以啟用和禁用的方法。

要開始配置API保護，請返回到F5分散式雲控制台，選擇 Swagger文件，然後點擊 添加Swagger文件。

.. figure:: assets/swagger_navigate.png

給Swagger文件命名，然後上傳它。一旦上傳完成，點擊 保存並退出。

.. figure:: assets/swagger_upload_file.png

現在轉到創建API定義。導航到 API定義，然後點擊 添加API定義 按鈕。

.. figure:: assets/api_definition_navigate.png

在元數據部分輸入一個名稱。然後轉到 Swagger規範 部分並打開下拉菜單。選擇先前添加的swagger規範，然後點擊 保存並退出 以創建API定義對象。

.. figure:: assets/api_definition_create.png

現在我們需要將創建的API定義附加到我們的HTTP負載平衡器。導航到 負載平衡器 並選擇 HTTP負載平衡器。我們先前創建的HTTP負載平衡器將會出現。打開其菜單並選擇 管理配置。

.. figure:: assets/api_definition_lb_popup.png

點擊 編輯配置 開始編輯。

.. figure:: assets/api_definition_lb_edit.png

在 API保護 部分啟用 API定義，然後選擇先前創建的API定義。

.. figure:: assets/api_definition_select_api_def.png

現在我們需要創建一個新的服務政策，該政策包含一套自定義規則，這些規則將為我們Swagger文件中包含的特定API資源指定允許或拒絕的規則動作。這種方法使用服務政策和自定義規則的組合來微調並提供對我們的應用程式API資源保護方式的細粒度控制。

滾動到 通用安全控制 部分並選擇 應用指定的服務政策。然後點擊 配置。

.. figure:: assets/api_definition_policy.png

在 選擇項目 欄位上點擊，並選擇 添加項目 選項。

.. figure:: assets/api_definition_policy_create.png

在元數據部分為政策輸入一個名稱，並轉到 規則 部分。選擇 自定義規則列表 並點擊 配置。

.. figure:: assets/api_definition_policy_create_rules.png

現在讓我們添加規則：點擊 添加項目。

.. figure:: assets/api_definition_rule_add.png

第一條規則將拒絕所有除API之外的。在元數據部分輸入一個名稱並向下滾動。

.. figure:: assets/api_definition_rule_add_details.png

接下來配置HTTP路徑。在 HTTP路徑 部分點擊 配置。

.. figure:: assets/api_definition_rules_path.png

並填寫路徑 - 對於這次演示為 /api/v1/。然後點擊 應用。

.. figure:: assets/api_definition_rules_prefix.png

滾動到 進階匹配 部分，並在API群組匹配器欄位中點擊 配置。

.. figure:: assets/api_definition_rules_api_matcher.png

在API群組匹配器屏幕中，選擇一個確切的值。

.. figure:: assets/api_definition_rules_matcher_select_api_def.png


勾選 反轉字串匹配器 選項，然後點擊 應用 以添加匹配器。

.. figure:: assets/api_definition_matcher_tick.png

點擊另一個 應用 以添加規則規範。

.. figure:: assets/api_definition_policy_apply.png

點擊 應用 以添加規則。

.. figure:: assets/api_definition_add_rule.png

在規則部分使用 添加項目 選項創建一個 'allow-other' 的規則。

.. figure:: assets/api_definition_second_rule.png

首先，在元數據部分輸入一個名稱。
   
.. figure:: assets/api_definition_second_rule_details.png

接下來，在動作部分的動作欄位中選擇 **允許**。

.. figure:: assets/api_definition_second_rule_allow.png

點擊 應用 以添加規則規範。

.. figure:: assets/api_definition_second_rule_apply.png

點擊 應用 以添加第二條規則。

.. figure:: assets/api_definition_second_rule_add.png

查看創建的規則，然後點擊 **應用**。

.. figure:: assets/api_definition_rule_list_apply.png

點擊 **繼續** 以將服務政策添加到負載均衡器，然後點擊 **應用**。

.. figure:: assets/api_definition_continue.png

.. figure:: assets/api_definition_def_policy_apply.png

最後一步是查看並保存已編輯的HTTP負載均衡器的配置。一旦在最後點擊 **保存並退出**，負載均衡器將更新API安全設置，我們的API資源將得到保護！

.. figure:: assets/api_definition_lb_save.png

做得好！我們的示例評分應用的API根據上傳的Swagger文件中的規範得到了保護。讓我們試試看，現在應該是禁止訪問的。

回到測試工具 `<https://test-tool.sr.f5-cloud-demo.com>`_，然後點擊 **發送攻擊**。在它下面的框中，我們將看到 **受保護** 的狀態，所以我們的API現在是安全的。

.. figure:: assets/test_api_2.png

在API規範未知或文檔記載不清的情況下，F5分布式雲API安全提供了一種基於機器學習（ML）的動態API發現服務。

API發現分析流向API端點的流量，並構建一個視覺圖表來詳細說明API路徑關係。對於一個組織來說，跟蹤API可能會很困難，因為它們通常會頻繁變化。隨著時間的推移，F5分布式雲可以對正常的API行為、使用和方法進行基線劃定，檢測異常，並幫助組織檢測帶來意外風險的影子API。

在下面的截圖中，我們可以看到請求的百分比，特定端點的學習模式，甚至可以下載基於發現的API自動生成的Swagger文件。
.. figure:: assets/api_auto_discovery.png 



Wrap-Up
#######

在這個階段，您應該已經設置了一個示例應用並向其發送了流量。您已經配置並應用了F5分布式雲WAAP服務，以保護應用的Web和API免受惡意用戶和機器人的攻擊。我們還查看了各種儀表板和安全事件中數據的遙測和洞察。

我們希望您對F5分布式雲WAAP服務有了更好的理解，並且現在已經準備好為您自己的組織實施它。如果您有任何問題或疑問，請隨時通過GitHub提出。謝謝！
