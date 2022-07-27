# kubernetes-wireguard  

# YA vpn service in kubernetes

## Table of contents
1. [Why](#why)
2. [Key generation](#keygen)
3. [Kubernetes deployment](#kubeconf)
4. [Test and results](#results)



<br/><br/>

## Why <a name="why"></a>  
Honestly i was scared to dismiss my beloved ipsec based on [strongswan](https://github.com/lorenzogirardi/kubernetes-strongswan),  
however on of my colleagues shared me how low is the overhead of wireguard, so i was interested to check myself.  

<br/><br/>

## Key generation <a name="keygen"></a>

First you have to install wg somewhere , however if u are using a mac , it's enough   
```brew install wireguard-tools```   
  

Second step is generate the keys   

Server:  
```
wg genkey > server_privatekey  
wg pubkey < server_privatekey > server_publickey_me
```

Client:   
```
wg genkey | tee me_privatekey | wg pubkey > me_publickey   
```

<br/><br/>

## Kubernetes deployment <a name="kubeconf"></a>  

Let's create a dedicated namespace , i'm not a fan of default  
```
apiVersion: v1
kind: Namespace
metadata:
  name: wireguard
  labels:
    name: wireguard
```
    

Since i'm under a vps i will use the nodePort  
```
apiVersion: v1
kind: Service
metadata:
  name: wireguard
  namespace: wireguard
spec:
  type: NodePort
  ports:
    - port: 51820
      nodePort: 31820
      protocol: UDP
      targetPort: 51820
  selector:
    name: wireguard
```
in this example the udp will be shown on public_ip:31820  


Now ... we have multiple options , mount a persisten volume , use secrets , use configmaps etc etc  
here just for a test purpose i've used the secrets   
```
apiVersion: v1
kind: Secret
metadata:
  name: wireguard
  namespace: wireguard
type: Opaque
stringData:
  wg0.conf.template: |
    [Interface]
    Address = 172.16.18.0/20
    ListenPort = 51820
    PrivateKey = <server_privatekey>
    PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -A FORWARD -o %i -j ACCEPT; iptables -t nat -A POSTROUTING -o ENI -j MASQUERADE
    PostUp = sysctl -w -q net.ipv4.ip_forward=1
    PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -D FORWARD -o %i -j ACCEPT; iptables -t nat -D POSTROUTING -o ENI -j MASQUERADE
    PostDown = sysctl -w -q net.ipv4.ip_forward=0

    [Peer]
    #me
    PublicKey = <me_publickey>
    AllowedIPs = 172.16.18.10
```

where ```72.16.18.0/20``` is the road warrior tunnel and   
```172.16.18.10``` is the client ip assigned to me.  

Last the deployment  
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wireguard
  namespace: wireguard
spec:
  selector:
    matchLabels:
      name: wireguard
  template:
    metadata:
      labels:
        name: wireguard
    spec:
      initContainers:
        - name: "wireguard-template-replacement"
          image: "busybox"
          command: ["sh", "-c", "ENI=$(ip route get 8.8.8.8 | grep 8.8.8.8 | awk '{print $5}'); sed \"s/ENI/$ENI/g\" /etc/wireguard-secret/wg0.conf.template > /etc/wireguard/wg0.conf; chmod 400 /etc/wireguard/wg0.conf"]
          volumeMounts:
            - name: wireguard-config
              mountPath: /etc/wireguard/
            - name: wireguard-secret
              mountPath: /etc/wireguard-secret/
      containers:
        - name: "wireguard"
          image: "linuxserver/wireguard:latest"
          ports:
            - containerPort: 51820
          env:
            - name: "TZ"
              value: "Europe/Rome"
            - name: "PEERS"
              value: "example"
          volumeMounts:
            - name: wireguard-config
              mountPath: /etc/wireguard/
              readOnly: true
          securityContext:
            privileged: true
            capabilities:
              add:
                - NET_ADMIN
      volumes:
        - name: wireguard-config
          emptyDir: {}
        - name: wireguard-secret
          secret:
            secretName: wireguard
      imagePullSecrets:
        - name: docker-registry
```

if everithing is done correctly you will se the following  
```kubectl get pods -n wireguard```    
```
NAME                         READY   STATUS    RESTARTS   AGE
wireguard-74ff66988d-ltwkq   1/1     Running   0          108m
```    
and  

```# kubectl exec -n wireguard -it deployment/wireguard -- bash```   
```
root@wireguard-74ff66988d-ltwkq:/# wg show
interface: wg0
  public key: Er7V4vxMEVBNZbqbDzHgXlYnZjSwrJYtwds86oOLLEg=
  private key: (hidden)
  listening port: 51820

peer: dfJjw5rdVNhmcIlDyFAXZI0rBQydsw9uqlh4kFJxBa0I=
  endpoint: 10.0.254.135:33851
  allowed ips: 172.16.18.10/32
  latest handshake: 5 minutes, 16 seconds ago
  transfer: 30.63 MiB received, 6.55 MiB sent
```


For the client i strongly ... STRONGLY suggest to import a configuration 

```$ cat wireconf.conf```
```
[Interface]
Address = 172.16.18.10/32
PrivateKey = <me_privatekey>
DNS = 1.1.1.1

[Peer]
PublicKey = <server_publickey_me>
AllowedIPs = 0.0.0.0/0, ::/0
Endpoint = <your-public-ip>:31820
```

<br/><br/>


## Test and results <a name="results"></a>

From mobile  
[![wirecardmobile](https://res.cloudinary.com/ethzero/image/upload/c_scale,w_240/v1658175857/misc/wirecardmobile.png)](https://res.cloudinary.com/ethzero/video/upload/v1658175554/misc/wireguard-mobile.mp4 "wirecardmobile")   

And what about the resources usage with 1g file download...   
![](https://res.cloudinary.com/ethzero/image/upload/v1658176773/misc/wireguardesktop.png)



<br/><br/>

So i'm really impressed by the efficency of this tool ... i was reading a lot of good mention about it,  
however today i discovered another tool for my swiss army knife



<br/><br/>

### UPDATE   
Added a monitoring service as a sidecar container based on prometheus exporter https://github.com/MindFlavor/prometheus_wireguard_exporter   


```
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/path: "/metrics"
        prometheus.io/port: "9586"
```

this annotation is based of kubernetes service discovery in prometheus server   

```
        - name: "wg-exporter"
          image: "mindflavor/prometheus-wireguard-exporter:3.6.3"
          args: ["--prepend_sudo=true"]
          ports:
            - containerPort: 9586
          securityContext:
            privileged: true
            capabilities:
              add:
                - NET_ADMIN
                - NET_BIND_SERVICE
```   

Grafana

![](https://res.cloudinary.com/ethzero/image/upload/v1658901252/misc/grafana-wireguard.png)



