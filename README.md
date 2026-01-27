# ubuntu-net-proxy-lab
Implementation of a virtual router and proxy on Ubuntu to simulate a private company server.

## 實作
在 Ubuntu 中實作 NET 模擬虛擬路由，並建 Proxy 模擬實際應用之公司的私服器，會避免外部連線和過濾內部像外的請求。

## 所建的架構
- 目前網路架構
    ```bash
    ip a
    ```
    <img src="/images/fig1.png" alt="示意圖" width="700">

- 在 network namespace 中的 ns1 執行 `ip a` 這個指令，查看他的網路介面和 IP 配置，這是模擬 veth0 路由下子網中的一台用戶端 ns1。
    ```bash
    sudo ip netns exec ns1 ip a
    ```
    <img src="/images/fig2.png" alt="示意圖" width="700">

- 查看 ns1 的路由
    ```bash
    sudo ip netns exec ns1 ip route
    ```
    <img src="/images/fig3.png" alt="示意圖" width="700">

## 實際運行狀況 (Demo)
即時查看 tinyproxy 服務 (Proxy) 的運行日誌，執行在左上終端機中，指令：
```bash
sudo journalctl –u tinyproxy –f
```
即時查看 MASQUERADE (NET) 運作，執行在左下終端機中，指令：
```bash
watch –n 1 sudo iptables –t nat –L –n –v
```
右邊終端機即將執行的指令，如：
```bash
sudo ip netns exec ns1 curl --interface 10.0.0.2 -x http://10.0.0.1:8888 http://example.com
```
**上面指令的解釋**：讓 ns1 的 client (10.0.0.2) 發送一個 HTTP 請求，先送到 tinyproxy (10.0.0.1:8888)，再由 tinyproxy 代替它去存取 example.com。

- 上述三個終端機視窗設置如下圖：

  <img src="/images/fig4.png" alt="示意圖" width="700">

【註】proxy 只有放行測試網 `http://example.com` 和校網 `https://www.cycu.edu.tw`

- (1) 在右邊終端機執行指令 (**連到測試網**)：
    ```bash
    sudo ip netns exec ns1 curl --interface 10.0.0.2 -x http://10.0.0.1:8888 http://example.com
    ```
    下圖為結果，可以看到左上終端機 Established connection (成功建立)，左下終端機的 pkts 從原本 (上圖) 的 4 增加到了 7，表示 NAT 有運作，右邊終端機有顯示回傳的結果，網頁 HTML 內容，表示有成功連上。

    <img src="/images/fig5.png" alt="示意圖" width="700">

- (2) 在右邊終端機執行指令 (**連到校網**)：
    ```bash
    sudo ip netns exec ns1 curl --interface 10.0.0.2 -x http://10.0.0.1:8888 https://www.cycu.edu.tw
    ```
    下圖為結果，可以看到左上終端機 Established connection (成功建立)，左下終端機的 pkts 從原本 (上圖) 的 7 增加到了 10，表示 NAT 有運作，右邊終端機有顯示回傳的結果，網頁 HTML 內容，表示有成功連上。

    <img src="/images/fig6.png" alt="示意圖" width="700">

- (3) 右邊終端機執行指令 (**連到 Google**)：
    ```bash
    sudo ip netns exec ns1 curl --interface 10.0.0.2 -x http://10.0.0.1:8888 https://google.com
    ```
    下圖為結果，可以看到左上終端機 refused on filtered domain (拒絕存取已過濾的域名)，左下終端機的 pkts 不變，從原本 (上圖) 的 10 沒變還是 10，表示 NAT 沒有運作，右邊終端機有顯示 proxy 回傳的結果，被 proxy 擋下來了，回傳 403 (拒絕請求)。

    <img src="/images/fig7.png" alt="示意圖" width="700">

- (4) 右邊終端機執行指令 (**連到 YouTube**)：
    ```bash
    sudo ip netns exec ns1 curl --interface 10.0.0.2 -x http://10.0.0.1:8888 https://www.youtube.com
    ```
    下圖為結果，可以看到左上終端機 refused on filtered domain (拒絕存取已過濾的域名)，左下終端機的 pkts 不變，從原本 (上圖) 的 10 沒變還是 10，表示 NAT 沒有運作，右邊終端機有顯示 proxy 回傳的結果，被 proxy 擋下來了，回傳 403 (拒絕請求)。

    <img src="/images/fig8.png" alt="示意圖" width="700">

## 環境
Ubuntu 22.04.5

## 網路路徑
外網經過 NET 到家用路由器，家用路由這個子網給 Ubuntu 位址，也就是 ns0，為了實作 NET 模擬子網，所以 NET 出了 ns1，這個虛擬路由器。
```
veth (virtual ethernet interface) 虛擬網卡介面
ns (network namespace) 獨立的網路堆疊環境

e.g. veth0 是「網卡」，ns0 是「網路空間」，可以把 veth0 接到 ns0 裡，讓 ns0 這個「虛擬電腦」有一張網卡使用
```
家用路由子網網域：192.168.1.0/24  

家用路由下 Ubuntu IP：192.168.1.130（實體網卡 enp0s5）  

模擬路由：10.0.0.1（veth0；ns0）  

模擬路由子網網域：10.0.0.0/24  

模擬子網網域下的某個用戶 IP：10.0.0.2（veth1；ns1）  

封包路徑：  
用戶端（ns1, 10.0.0.2）-> veth1 -> 模擬路由（veth0, 10.0.0.1） -> Tinyproxy（10.0.0.1:8888）-> NAT（MASQUERADE）-> Ubuntu（enp0s5, 192.168.1.141）-> 家用路由（192.168.1.0/24）-> NAT -> 公網（WAN）

note：模擬路由（veth0, 10.0.0.1）就是在模擬公司的總路由器。

## 設置指令
- 建 namespace + veth  
    ```bash
    sudo ip netns add ns1  
    sudo ip link add veth0 type veth peer name veth1  
    sudo ip link set veth1 netns ns1  
    ```

- 設定 10.0.0.1 Gateway，用於連接 veth1 ns1，並做啟動  
    ```bash
    sudo ip addr add 10.0.0.1/24 dev veth0  
    sudo ip link set veth0 up  
    ```

- 設置子網中的模擬設備 namespace client（10.0.0.2）  
    ```bash
    sudo ip netns exec ns1 ip addr add 10.0.0.2/24 dev veth1  
    sudo ip netns exec ns1 ip link set veth1 up  
    sudo ip netns exec ns1 ip link set lo up
    ```

- 讓 ns1 用 Ubuntu 當網關
    ```bash
    sudo ip netns exec ns1 ip route add default via 10.0.0.1
    ```

- 設置 NAT，用的是 MASQUERADE
    ```bash
    sudo sysctl -w net.ipv4.ip_forward=1
    sudo iptables -t nat -A POSTROUTING -o enp0s5 -j MASQUERADE
    ```

- 測試 ns1 能否上網
    ```bash
    sudo ip netns exec ns1 ping 8.8.8.8
    ```

- 查看 ns1 路由
    ```bash
    sudo ip netns exec ns1 ip route
    ```

- 查看 ns1 ip 相關資訊
    ```bash
    sudo ip netns exec ns1 ip a
    ```

- 安裝 Proxy，用的是 tinyproxy，並確認是否有檔案 tinyproxy.conf
    ```bash
    sudo apt install tinyproxy
    sudo nano /etc/tinyproxy/tinyproxy.conf
    ```

- tinyproxy 相關指令
    ```bash
    sudo systemctl start tinyproxy（啟動）
    sudo systemctl stop tinyproxy（關掉）
    sudo systemctl restart tinyproxy（重啟）
    sudo systemctl status tinyproxy（查看狀態）
    ```
- 開啟 tinyproxy.conf 檔，處理其內容，把註解拿掉、檢查或是修改內容（依序往下找相對應區塊）
    ```bash
    sudo nano /etc/tinyproxy/tinyproxy.conf
    ```
    - Port 區塊  
    Port 8888（預設，確認是 8888）

    - Listen 區塊   
    Listen 10.0.0.1（修改）

    - LogFile 區塊  
    LogFile "/var/log/tinyproxy/tinyproxy.log"（確認）

    - Syslog 區塊  
    Syslog On（確認）

    - LogLevel 區塊  
    LogLevel Connect（修改）

    - Allow 區塊  
    Allow 10.0.0.0/8（拿掉註解）

    - Filter 區塊  
    Filter "/etc/tinyproxy/filter"（拿掉註解）

    - FilterDefaultDeny 區塊  
    FilterDefaultDeny Yes（確認）

- 修改完 tinyproxy.conf 重啟
    ```bash
    sudo systemctl restart tinyproxy
    ```

- 建立 filter 檔案並開啟
    ```bash
    sudo nano /etc/tinyproxy/filter
    ```
    寫入  
    ```
    www.cycu.edu.tw  
    example.com  
    ```
    (只允許這兩個網站能成功連線)    

- 查看 Tinyproxy 日誌 log（終端機視窗一）
    ```bash
    sudo journalctl -u tinyproxy -f
    ```

- 查看 MASQUERADE 運作（終端機視窗二）
    ```bash
    watch -n 1 sudo iptables -t nat -L -n -v
    ```

- 網站連線測試（終端機視窗三）
    ```bash
    sudo ip netns exec ns1 curl --interface 10.0.0.2 -x http://10.0.0.1:8888 http://example.com

    sudo ip netns exec ns1 curl --interface 10.0.0.2 -x http://10.0.0.1:8888 https://example.com

    sudo ip netns exec ns1 curl --interface 10.0.0.2 -x http://10.0.0.1:8888 https://www.cycu.edu.tw

    sudo ip netns exec ns1 curl --interface 10.0.0.2 -x http://10.0.0.1:8888 https://google.com

    sudo ip netns exec ns1 curl --interface 10.0.0.2 -x http://10.0.0.1:8888 http://google.com

    sudo ip netns exec ns1 curl --interface 10.0.0.2 -x http://10.0.0.1:8888 https://www.youtube.com
    ```

- 刪掉建立的虛擬子網
    ```bash
    sudo ip netns delete ns1
    ```

## 設製成執行檔
- 因為虛擬路由、子網每次關機會被刪除  
- 建立 setup_ns1.sh 並開啟
    ```bash
    nano setup_ns1.sh
    ```
    寫入
    ```
    #!/bin/bash

    ip netns add ns1
    ip link add veth0 type veth peer name veth1
    ip link set veth1 netns ns1

    ip addr add 10.0.0.1/24 dev veth0
    ip link set veth0 up

    ip netns exec ns1 ip addr add 10.0.0.2/24 dev veth1
    ip netns exec ns1 ip link set veth1 up
    ip netns exec ns1 ip link set lo up

    ip netns exec ns1 ip route add default via 10.0.0.1

    sysctl -w net.ipv4.ip_forward=1

    iptables -t nat -A POSTROUTING -o enp0s5 -j MASQUERADE
    ```

- 設置執行權限
    ```bash
    chmod +x setup_ns1.sh
    ```
    
- 執行（每次開機直接執行這個，建立所有所需模擬）
    ```bash
    sudo ./setup_ns1.sh
    ```
    p.s. 記得 tinyproxy 要先開
