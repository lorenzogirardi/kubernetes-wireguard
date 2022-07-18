# kubernetes-wireguard  

# YA vpn service in kubernetes

## Table of contents
1. [Key generation](#keygen)
2. [Kubernetes deployment](#example2)
3. [Test and results](#third-example)


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


