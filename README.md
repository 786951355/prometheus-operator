#这是个部署prometheus-operator的yaml文件   

#版本号：0.26.0  
#部署方式：yaml文件手工，非helm  
#监控有：  

        monitoring/alertmanager/0 (3/11 active targets)  
        monitoring/kube-apiserver/0 (1/1 active targets)  
        monitoring/kube-controller-manager/0 (1/17 active targets)  
        monitoring/kube-dns/0 (1/17 active targets)  
        monitoring/kube-etcd/0 (3/17 active targets)  
        monitoring/kube-scheduler/0 (1/17 active targets)  
        monitoring/kube-state-metrics/1 (1/11 active targets)  
        monitoring/kubelet/0 (2/17 active targets)  
        monitoring/kubelet/1 (2/17 active targets)  
        monitoring/node-exporter/0 (2/11 active targets)  
        monitoring/prometheus-operator/0 (1/11 active targets)  
        monitoring/prometheus/0 (2/11 active targets)  
        
部署时候有一些坑的说明：  
#1.添加dns的监控
		
		在kube-dns的service.yaml里添加一下内容，千万不要盲目添加dns的service，会把整个k8s集群dns无法解析的！
		labels:   #因为servicemonitor用这个标签来寻找service
   			 app: exporter-kube-dns
    		 component: kube-dns
    	和		
	    - name: http-metrics-dnsmasq 
          port: 10054 
          protocol: TCP 
          targetPort: 10054 
        - name: http-metrics-skydns 
          port: 10055 
          protocol: TCP 
          targetPort: 10055 
        selector: 
          k8s-app: kube-dns 
          
#2.添加etcd的监控：
       我的etcd集群是在k8s外面二进制部署的，要监控得在k8s上添加endpoints，service，servicemonitor
       修改etcd/endpoints.yaml的ip地址及端口，协议，按自己的需求看情况
       我用的是http方式，因为测试https没成功把TSL证书传进prometheus里，所以修改了etcd的配置打开了http的metrics
       
       etcd的配置添加：  --listen-metrics-urls=http://192.168.79.130:2222 \ 
       请重载重启etcd后访问能不能获取metrics值
  
**curl http://192.168.79.131:2222/metrics**

		[root@k8s-master1 etcd]# curl http://192.168.79.131:2222/metrics |less
 		 % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
		# HELP etcd_debugging_mvcc_db_compaction_keys_total Total number of db keys compacted.
		# TYPE etcd_debugging_mvcc_db_compaction_keys_total counter
		etcd_debugging_mvcc_db_compaction_keys_total 120688
		# HELP etcd_debugging_mvcc_db_compaction_pause_duration_milliseconds Bucketed histogram of db compaction pause duration.
		# TYPE etcd_debugging_mvcc_db_compaction_pause_duration_milliseconds histogram
		etcd_debugging_mvcc_db_compaction_pause_duration_milliseconds_bucket{le="1"} 0
		etcd_debugging_mvcc_db_compaction_pause_duration_milliseconds_bucket{le="2"} 0
		etcd_debugging_mvcc_db_compaction_pause_duration_milliseconds_bucket{le="4"} 0
		etcd_debugging_mvcc_db_compaction_pause_duration_milliseconds_bucket{le="8"} 0
		etcd_debugging_mvcc_db_compaction_pause_duration_milliseconds_bucket{le="16"} 0
		etcd_debugging_mvcc_db_compaction_pause_duration_milliseconds_bucket{le="32"} 0
		etcd_debugging_mvcc_db_compaction_pause_duration_milliseconds_bucket{le="64"} 0
		etcd_debugging_mvcc_db_compaction_pause_duration_milliseconds_bucket{le="128"} 0
		etcd_debugging_mvcc_db_compaction_pause_duration_milliseconds_bucket{le="256"} 0
		etcd_debugging_mvcc_db_compaction_pause_duration_milliseconds_bucket{le="512"} 0
		etcd_debugging_mvcc_db_compaction_pause_duration_milliseconds_bucket{le="1024"} 0
		etcd_debugging_mvcc_db_compaction_pause_duration_milliseconds_bucket{le="2048"} 0
		etcd_debugging_mvcc_db_compaction_pause_duration_milliseconds_bucket{le="4096"} 0
		..............
       
       有这些metrics就配置对了
		
#3.无法发现kube-controller-manager和kube-scheduler的Endpoint
![](/Users/guleng/Desktop/WX20190110-182332.png)

		原因都是k8s自带service没有kube-controller-manager和kube-scheduler没有labels导致找不到Endpoint
		我们添加新的Endpoint及service(原因是这俩个组件是二进制方式部署的)，servicemonitor
		部署controller-scheduler目录下的yaml就可以了，但是请看自己的场景是不是master集群，
		按照自己的集群修改配置文件的ip地址，要是HA模式就添加三个ip地址！！！
		
#4.配置alertmanager的报警：
	
		修改alertmanager/alertmanager-secret.yaml的base64编码段，把自己的配置编码一节更改即可，
		要是修改错会alertmanager会起不来
		
		我的个人的报警配置：支持微信，邮件，钉钉(专门起了个pod  webhook方式发送消息)
		global:
		  resolve_timeout: 2s
		route:
		  group_by: ['alertname']   #根据label标签进行匹配,如果是多个,就要多个都匹配
		  #continue: true
		  group_wait: 1s            #等待该组的报警,看有没有一起拼车的
		  group_interval: 1s        #下一次报警开车时间,alert不会分不同的指标挨个儿发一遍,而是全部汇总发完后等一段时间后再拼车一次发
		  repeat_interval: 1s       #重复报警时间(同一个指标的下次报警时间间隔): group_interval+repeat_interval
		  #receiver: wechat
		  #receiver: live-monitoring   #下面的匹配的接受项
		  receiver: 'live-monitoring'
		  routes:   #多个报警路径的配置
		  - match:
		      severity: critical        #告警的级别是critical时发报警渠道是或后报警的人事live-monitoring
		    receiver: 'live-monitoring'
		  - match_re:
		      severity: ^(warning|info)$ #waining或info时通过这个wechat
		    receiver: 'wechat'
		    #continue: true   #递归发报警
		    group_by: ['alertname']
		    #receiver: 'live-monitoring'
		    #routes:   #多个报警路径的配置
		  - receiver: 'wechat'
		    group_by: ['alertname']
		    continue: true
		  - receiver: 'webhook'
		    group_by: ['alertname']
		    continue: false
		receivers:
		- name: 'wechat'
		  wechat_configs:
		  - api_secret: 'SHGXwaE8**Zzy3EfXVnD******8awNkpw-cpTgKDotkBY'
		    corp_id: 'wx26**1a45b*****42'
		    agent_id: '1000003'   
		    send_resolved: true #是否通知已解决的警报 
		    to_user: 'guleng'    #接受者
		    to_party: '1'     #接受组
		    to_tag: ''
		- name: 'live-monitoring'
		  email_configs:
		  - send_resolved: true
		    smarthost: 'smtp.exmail.qq.com:465'
		    from: 'developer@qq.net.cn'
		    require_tls: false
		    auth_username: 'developer@qq.net.cn'
		    auth_password: '101**Pp!'
		    to: 'guleng@qq.net.cn'
		    headers: { Subject: "[WARN] 报警邮件"} # 接收邮件的标题
		- name: 'webhook'
		  webhook_configs:
		  - url: 'http://prometheus-webhook-dingtalk:5358/dingtalk/webhook1/send'

       
   
