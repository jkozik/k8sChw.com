# k8sChw.com
Setup camptonhillsweather.com in a kubernetes cluster.  

Source image [InstallCHW.com](https://github.com/jkozik/InstallCHW.com)

# build image, put on docker hub
CamptonHillsWeather.com is running in a docker container on my host, directly -- not in a VM.  To make it run in my kubernetes cluster, I need to push the image to my dockerhub repository.  The kubernetes deployment resource has an "image" field that triggers a pull from dockerhub.
```
docker login
docker tag jkozik/chw.com jkozik/chw.com:v1
docker push jkozik/chw.com:v1
```

# Setup NFS share on host
The weather computer (outside of the cluster), runs a program called Cumulus.  It is designed to upload realtime weather data and graphes frequently to a website.  In my case, I have a login on a host dedicated to that website, user=weather. The path to the website is /home/weather/public_html. I want to make that data visible to the deployment that I want to run inside my cluster.  First step is to make an NFS share out of the data.  From the root login of my host (not in the cluster):
```
[root@dell2 public_html]# vi /etc/exports
/home/weather/public_html    192.168.100.0/24(ro,sync,no_root_squash,no_all_squash)
[root@dell2 public_html]# service nfs-server restart
Redirecting to /bin/systemctl restart nfs-server.service
```
# Create Persistent Volume
To make an NFS share visible to a deployment inside of a kubernetes cluster, one must create a persistent volume. The deployment accesses it by making a persistent volume claim. 
```
[jkozik@dell2 k8sChw.com]$ kubectl apply -f chwcom-pv.yml
persistentvolume/chwcom-persistent-storage created

[jkozik@dell2 k8sChw.com]$ kubectl get pv
NAME                           CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                                  STORAGECLASS   REASON   AGE
chwcom-persistent-storage      1Gi        ROX            Retain           Available                                          nfs                     2m42s

[jkozik@dell2 k8sChw.com]$ kubectl apply -f chwcom-pvc.yml
persistentvolumeclaim/chwcom-persistent-storage created

[jkozik@dell2 k8sChw.com]$ kubectl get pvc
NAME                           STATUS   VOLUME                         CAPACITY   ACCESS MODES   STORAGECLASS   AGE
chwcom-persistent-storage      Bound    chwcom-persistent-storage      1Gi        ROX            nfs            9s
```
Verify that the status is "bound." The key input to the application is the realtime weather data and charts.  That data is uploaded to /home/weather/public_html, shared over NFS, provisioned into my cluster as a resource (PV) and bound to my application (PVC).

# Create service and deployment
Create the chwcom service and deploy it.
```
[jkozik@dell2 k8sChw.com]$ kubectl apply -f chwcom-svc.yml
service/chwcom created
[jkozik@dell2 k8sChw.com]$ kubectl get service
NAME              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
chwcom            NodePort    10.105.158.243   <none>        80:31200/TCP   8s

[jkozik@dell2 k8sChw.com]$ kubectl apply -f chwcom-deploy.yml
deployment.apps/chwcom configured

[jkozik@dell2 k8sChw.com]$ kubectl get pods -o wide 
NAME                                   READY   STATUS    RESTARTS   AGE    IP             NODE       NOMINATED NODE   READINESS GATES
chwcom-6f48d9b7cc-jrc9j                1/1     Running   0          108s   10.68.77.142   kworker2   <none>           <none>
```
Note that the first time this deployment runs, it needs to download the image from the dockerhub repository.  It will take awhile for the above command to show that the deployment is running.
Next verify that the pod can see the realtime weather data.  Get a bash shell prompt in the pod and cd to the directory where the weather data sits and verify the timestamps.
```
[jkozik@dell2 k8sChw.com]$ kubectl exec chwcom-6f48d9b7cc-jrc9j -it /bin/bash
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl kubectl exec [POD] -- [COMMAND] instead.

root@chwcom-6f48d9b7cc-jrc9j:/var/www/html# cd mount
root@chwcom-6f48d9b7cc-jrc9j:/var/www/html/mount# ls
cumulus  davis  favicon.ico  favicon.ico.1  saratoga  webcam  x.tmp
root@chwcom-6f48d9b7cc-jrc9j:/var/www/html/mount# cd cumulus

root@chwcom-6f48d9b7cc-jrc9j:/var/www/html/mount/cumulus# ls
CumulusRealtime.swf  dbimages    images     monthlyrecord.htm  record.htm     thisyear.htm  trends.htm        wz_jsgraphics.js
Reports              gauges.htm  index.htm  realtime.txt       thismonth.htm  today.htm     weatherstyle.css  yesterday.htm

root@chwcom-6f48d9b7cc-jrc9j:/var/www/html/mount/cumulus# ls -lasth realtime.txt
4.0K -rw-r--r--. 1 1002 1002 267 Jul  5 12:33 realtime.txt
root@chwcom-6f48d9b7cc-jrc9j:/var/www/html/mount# exit
```

# Apply ingress resource
The chwcom service needs to map to the URL camptonhillsweather.com.  Apply the ingress rule.  
```
[jkozik@dell2 k8sChw.com]$ kubectl apply -f chwcom-ingress.yml
ingress.networking.k8s.io/chwcom-ingress configured
```
The ingress point to this cluster is the cluster's IP address and the NodePort number of the ingress controller.  To find this out, review the following:
```
[jkozik@dell2 k8sChw.com]$ kubectl get services -n ingress-nginx
NAME                                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             NodePort    10.97.70.29     <none>        80:30140/TCP,443:30023/TCP   10d

[jkozik@dell2 k8sChw.com]$ kubectl get ingress
NAME                      CLASS    HOSTS                     ADDRESS           PORTS   AGE
chwcom-ingress            <none>   camptonhillsweather.com   192.168.100.174   80      7h44m
nginx-wordpress-ingress   <none>   k8s.kozik.net             192.168.100.174   80      10d

[jkozik@dell2 k8sChw.com]$ kubectl describe ingress chwcom-ingress
Name:             chwcom-ingress
Namespace:        default
Address:          192.168.100.174
Default backend:  default-http-backend:80 (<error: endpoints "default-http-backend" not found>)
Rules:
  Host                     Path  Backends
  ----                     ----  --------
  camptonhillsweather.com
                           /   chwcom:80 (10.68.77.142:80)
Annotations:               nginx.ingress.kubernetes.io/rewrite-target: /
Events:                    <none>

```
So, on my home LAN http://192.168.100.174:30410 is where incoming web traffic enters.  The ingress controller parses for camptonhillsweather.com and redirects the traffic to the chwcom service. I have an external IP address for my home LAN that gets NAT'd by my home router to 192.168.100.174.  On that NAT box, I map port 80 to port 30410. As you can see from the get ingress command, I also am running a wordpress application with another URL, but also mapped to the same IP address and port number.  The ingress control does a reverse proxy function and splits the wordpress traffic off to the nginx-wordpress-ingress resource. 
