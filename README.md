# multicloud-loadbalancer
跨雲端的 Load balancer

## 動機
想試試看不同 cloud 之間的 Load balance ，例如 GCP AWS DO 都開
像我們上次 demo 作法只能都在 DO 內
* 可能技術
    * Dynamic DNS
        * 每有一次 IP 更動，就改變 DNS 上的紀錄
        * 可是每次只能指定一個 IP，不符合 load balance 需求
    * Round-Robin DNS
        * 在 DNS A 紀錄 中，在同 subdomain 指定多行不同值
        * 缺點：沒辦法控制 client 要去列表中第幾個
    * GeoDNS
        * 對來自不同地方的使用者，DNS Server 回應不同 IP
        * [GeoDNS vs Anycast by mattgadient.com](https://mattgadient.com/i-tested-geodns-vs-anycast-for-websites-so-which-one-is-better/)、[Cloudflare vs Route 53 by mattgadient.com](https://mattgadient.com/geo-dns-cloudflare-vs-route-53-a-look-and-short-test-results/)
    * Anycast
        * 多台設備在不同地點，卻有同一個 IP
        * 使用 BGP routing protocol 在不同的地點和鄰近 ISP 進行路由交換
        * 每個地點都公佈相同的 AS number 及交換的路由
        * 由路由器依路由表來決定連到那一部設備
        * 例如：我們常用的 8.8.8.8、1.1.1.1、168.95.1.1 都是有用到這技術
        * 成本較高，需要擁有 ASN、有能力廣播 BGP
            * PS: 這場 SITCON 2021 議程 - [網路維運，在台灣怎麼玩？打造全台最大家用網路！ by 海豹](https://sitcon.org/2021/agenda/3fe74d93-2ab5-4c42-8b49-b5b92973f4aa)，講者有去申請到他自己的 ASN，我很期待聽到這場演講

## 實作跨雲端的 Load balance
* 架構圖
    ![](https://i.imgur.com/45uPz3j.png)
    > [設計原始檔](https://gist.github.com/jiazheng0609/d4d0f6c775a87d7e38cc3d47d5ef0a47)，[網頁版分享連結](https://viewer.diagrams.net/?highlight=0000ff&edit=_blank&layers=1&nav=1&title=LSAproject.drawio#Uhttps%3A%2F%2Fdrive.google.com%2Fuc%3Fid%3D114Y5djarxGvDSCLjlKrGBQAyl1xb753j%26export%3Ddownload)，用 app.diagrams.net 繪製，DigitalOcean Icon 取自[這裡](https://www.digitalocean.com/brand/)和[這裡](https://do.co/diagram-kit)
* 利用 DNS，根據地理位置提供不同的路由政策
* 預備資源
    * 自己的網域名稱，這裡用從 Gandi 註冊的 `jiazhengdev.tw`
        * 我是在參加 [COSCUP](https://coscup.org/) 時拿到的免費一年優惠券
    * DNS name server，這裡使用 AWS Router 53
        * 也能自己架 [BIND + GeoDNS](https://geoip.site/) 來達成，但用 AWS 的 C/P 值比較高
    * 兩個雲端服務供應商的帳號，這裡使用 DigitalOcean 和 AWS
        * DigitalOcean 註冊需要信用卡，透過 [BT 的邀請連結](https://m.do.co/c/7a14e0e84052)可拿點數
        * [AWS Educate](https://aidea-web.tw/files/AWS_Educate_for_Students.pdf) 學生方案可不用信用卡，可拿點數，需要數天審核
    * 在不同雲端 data center 中的多台虛擬機，這裡 DO 開 4 台、AWS 開 2 台
        * DO Droplet: 2 * keepalived + 2 * nginx
        * AWS EC2: 2 * nginx
### 步驟
1. 在 AWS Router 53 建立 `jiazhengdev.tw` 的託管區域
2. 前往域名註冊商 Gandi，把 nameserver (NS) 改指向 AWS Router 53
3. 這裡假設已經有根據[之前報告](https://hackmd.io/pLrUczzjQ_CUhcnPX2Xwhw?view#DEMO)，把 DigitalOcean Float IP、keepalived、nginx 架好了
4. 在兩台 AWS EC2 網頁伺服器的前端，我用了 AWS Elastic Load Balancer 來 load balance
    * 我是用之中的 Network Load Balancer 
    * 演算法是用 flow hash，參考自[官方文件](https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/how-elastic-load-balancing-works.html)
5. 在 Route 53 設定 DNS 規則，參考[官方教學](https://docs.aws.amazon.com/zh_tw/Route53/latest/DeveloperGuide/dns-failover-types.html)
    * 示意圖
    ![](https://i.imgur.com/JBxfoG2.png)
    > [設計原始檔](https://gist.github.com/jiazheng0609/d4d0f6c775a87d7e38cc3d47d5ef0a47)，用 app.diagrams.net 繪製
    * 「容錯移轉->主要」路由裡要設定「運作狀態檢查」的對象，要去另外一個界面新增規則
        * 規則截圖
        ![](https://i.imgur.com/9mmCdCl.png)
        * 但若要監測的是 AWS 內部資源（例如我這裡的 AWS Network Load Balancer）的別名，就只要把「評估目標運作狀態」打開，「運作狀態檢查」留空，他就會自動去偵測了
    * 為了方便辨識，我也把所有虛擬機 IP 用 A 紀錄起來了
    * [所有 DNS 紀錄列表](https://i.imgur.com/RCUgWE8.png)
    


## 監控 Monitoring
* 以下僅供系統管理員人工判斷，不直接參與以上機器之自動化決策
### Prometheus + Grafana
* 目的：自動定期監測，並整合、視覺化
* [成品公開展示頁面](http://monitor.jiazhengdev.tw:3000/d/6LB6zeR7z/watch-lsa?orgId=1&refresh=5s)
    ![](https://i.imgur.com/5jUBteG.png)
* 找公正第三方來定期健康檢查
    * 這裡用的是額外的機器，與上面有開的機器不重複
    * 這個環節其實也能冗餘做高可用，但目前我先不做
    * 監測點架在 DigitalOcean 新加坡，所以目前 `lsa.jiazhengdev.tw` 只能看到亞洲路由
* 安裝以下元件
    * [First steps | Prometheus](https://prometheus.io/docs/introduction/first_steps/) 負責統整來自 exporter 們的資料，再發給 Grafana
    * [Blackbox prober exporter](https://github.com/prometheus/blackbox_exporter) 用 HTTP 請求做健康檢查的工具，再提供資料給 Prometheus
    * [Download Grafana | Grafana Labs](https://grafana.com/grafana/download) 讓 Prometheus 的資料漂漂亮亮的圖形化呈現
* Prometheus 完整設定檔
```yaml
# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.


    static_configs:
    - targets: ['localhost:9090']

  - job_name: 'black_box'
    metrics_path: /probe
    scrape_interval: 5s # 可以自訂每間隔多久要去檢查一次
    params:
      module: [http_2xx]  # Look for a HTTP 200 response.
    static_configs:
      - targets:
        - http://lsa.jiazhengdev.tw    # Target to probe with http.
        - http://failover-sgp.jiazhengdev.tw
        - http://failover-us.jiazhengdev.tw
        - http://lsalb-d596176d69ccaeb9.elb.us-west-1.amazonaws.com/ # aws-lb
        - http://do-float-ip.jiazhengdev.tw
        - http://do-keepalived-master.jiazhengdev.tw/
        - http://do-keepalived-backup.jiazhengdev.tw/
        - http://do-nginx-primary.jiazhengdev.tw/
        - http://do-nginx-secondary.jiazhengdev.tw/
        - http://aws-us1.jiazhengdev.tw/
        - http://aws-us2.jiazhengdev.tw/
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 127.0.0.1:9115  # The blackbox exporter's real hostname:port.

```
### 在自己的 shell 上肉眼觀察
* `ping lsa.jiazhengdev.tw`
* `watch -n1 <command>` 每秒鐘執行一次指令
    * `watch -n1 dig lsa.jiazhengdev.tw` 檢查 DNS 狀態
    * `watch -n1 curl http://lsa.jiazhengdev.tw` 檢查網頁內容
### Alertmanager
* 目的：當 metrics 不如預期時，發出警告通知系統管理員，用簡訊、電子郵件、即時通訊軟體等方式
* 尚未完成，可能使用 [ metalmatze / alertmanager-bot ](https://github.com/metalmatze/alertmanager-bot)
* 預期成果：一個 Telegram Bot，出事時自動傳訊息給人
    ![](https://i.imgur.com/bWHSir9.png)

## Reference Link
* 跨雲端的 load balance
    * [Lien, M. (2018). Tellus One - 基於地理空間，為即時應用軟體設計的全球自組織伺服器 P2P 負載平衡系統 (Master's thesis).](http://ir.ncnu.edu.tw:8080/handle/310010000/11332)
    * [Global Server Load Balance (Global Traffic Managemnet) Enterprise Cloud Knowledge Center documentation | ecl.ntt.com](https://ecl.ntt.com/en/documents/service-descriptions/rsts/gslb/gslb.html)
    * https://docs.aws.amazon.com/zh_tw/Route53/latest/DeveloperGuide/dns-failover-types.html
    * [Looking Glass - Hurricane Electric (AS6939)](http://lg.he.net/) 檢查從各地路由的工具
    * [Anycast DNS by Vincent Bernat](https://vincent.bernat.ch/en/blog/2011-dns-anycast)
    * [TW DNS ANYCAST by TWNIC 許乃文](https://www.hcrc.edu.tw/media/education/DNS_0.pdf) 解釋 anycast 的中文簡報
    * https://grafana.com/blog/2020/02/25/step-by-step-guide-to-setting-up-prometheus-alertmanager-with-slack-pagerduty-and-gmail/
