# Network-Namespace-Replay
本项目希望能够在一个完全独立的Linux Network Namespace中回放流量：
1. 抓取流量，通过pcap获取当前Network Namespace中的指定流量，可以是全部
2. 获取当前Network Namespace中的所有网卡信息
3. 创建独立Network Namespace，创建设置相关的网卡
4. 根据pcap文件，通过不同的网卡给独立Network Namespace发送流量
