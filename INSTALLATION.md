# Installation

Installing Infranetes is currently a relatively manual affair.  

For these instructions we assume you can already create a kubernetes cluster (and these instructions assume it is created via `cluster/kube-up.sh` on aws from kubernetes)

1. Build infranetes and vmserver

2. Create CA certs and keys for use by the GRPC communication between `infranetes` and `vmserver` 

3. Create a base linux image to be used as pod host that starts Docker and `vmserver` on boot
 
4. Changing kubelet on an existing node to use infranetes as its container runtime via the CRI

5. Labeling and taininting the node to ensure that only pods meant to be scheduled via infranetes are scheduled to this node

6. Modify AWS VPC to work on the Intenet

## 1. Building infranetes and vmserver

 This is fairly straight forward go build
    
 ```bash
 $ go build ./cmd/ifranetes/infranetes.go
 $ go build ./cmd/vmserver/vmserver.go
 ```

## 2. Creating CA Certs and keys

1. Create a CA public/private key pair

 ```bash
 $ openssl genrsa -aes256 -out ca-key.pem 4096
 $ openssl req -new -x509 -days 365 -key ca-key.pem -sha256 -out ca.pem
 ```

2. Create a server key pair 
 
 ```bash
 $ openssl genrsa -out key.pem 4096
 ```

3. Create a certificate signing request for it

 ```bash
 $ openssl req -subj "/CN=*" -sha256 -new -key key.pem -out server.csr
 ```

4. Sign it with the CA key created above

 ```bash
 $ echo subjectAltName = IP:127.0.0.1 > extfile.cnf

 $ openssl x509 -req -days 365 -sha256 -in server.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out cert.pem -extfile extfile.cnf
 ```
 this will result in 3 files we care about for infranetes usage `ca.pem`, `key.pem`, and `cert.pem`

 * Note the above IP in the subjectAltName isn't the IP of the VM instance, however, all that TLS cares is that `infranetes` thinks it should be 127.0.0.1 and vmserver claims it is 127.0.0.1 and the certificate chain verifies to the CA 

## 3. Creating the base image

In amazon, the way we currently create the base image  

1. boot a regular ubuntu instance provided by AWS

2. scp `vmserver`, init files and server key/cert to the new instance

 `$ scp -i <ec2-key location> vmserver key.pem cert.pem vmserver.init ubuntu@<ec2 instance ip>:/tmp`

3. ssh to the instance and move files to the appropriate location

 ```bash
 $ ssh -i <ec2-key location> ubuntu@<ec2_instance_ip>

 <connect to vm>

 $ sudo -s
 # mv /tmp/vmserver /usr/local/sbin
 # mv /tmp/*.pem /root
 # mv /tmp/vmserver.init /etc/init.d/vmserver
 # systemctl daemon-reload
 ```

4. use aws to image this VM and one can name this image infranetes-base.  This AMI will be the image infrantes boot to act as a pod host

## 4. Modify an existing node to act as an infranetes node.

This section currently assumes that one created an aws kubernetes cluster with `cluster/kube-up.sh` from the kubernetes source repository
 
1. create a `vars.sh` file that will contain one's AWS keys

 ```bash
 export AWS_ACCESS_KEY_ID=<fill in>
 export AWS_SECRET_ACCESS_KEY=<fil in>
 ```
 
2. a) Creates a new subnet within the kubernetes vpc for use by infrantes.

   b) Add it to the kubernetes route table
    
   c) Configure it to auto assign a public ip address

2. create an `aws.json` that corresponds to one's kubernetes cluster configuration

 ```json
 {
  "Ami":"<ami-id created above>",
  "RouteTable":"<rtb-id created by kube-up>",
  "Region":"<region running in",
  "SecurityGroup":"<sg-id created by kube-up>",
  "Vpc":"<vpc-id created by kube-up>",
  "Subnet":"<subnet-id created above>",
  "SshKey":"<key used to connect to ubuntu account>"
 }
 ```

3. copy `infranetes`, `ca.pem`, `vars.sh` and `aws.json` to the node being modified and move to `/root`

4. modify `kubelet` via `/etc/sysconfig/kubelet` to use `infranetes` via the cri
  
 ```bash
 $ vi /etc/sysconfig/kubelet

 change DAEMON_ARGS to

 DAEMON_ARGS="--api-servers=https://172.20.0.9 --enable-debugging-handlers=true --cloud-provider=aws --config=/etc/kubernetes/manifests --allow-privileged=True --v=4 --cluster-dns=10.0.0.10 --cluster-domain=cluster.local --non-masquerade-cidr=10.0.0.0/8 --cgroup-root=/ --babysit-daemons=true --experimental-cri --container-runtime=remote --container-runtime-endpoint=/tmp/infra --feature-gates StreamingProxyRedirects=true --experimental-cgroups-per-qos=true"
 ```

5. on the node, run infranetes as root (I currently use a screen/tmux session for this)

 ```bash
 # ./infranetes -alsologtostderr -listen /tmp/infra -podprovider aws -master-ip 172.20.0.9 -base-ip <base ip of subnet created above>
 ```
 
 With AWS created via kube-up, we can autodetect the master and subnet base ip, so can simplify startup to
 
  ```bash
  # ./infranetes -alsologtostderr -listen /tmp/infra -podprovider aws
  ```
 

6. on the node restart kubelet to use the new configuration

 ```bash
 # systemctl restart kubelet
 ```

## 5. Modify Master Component to be CRI aware

add --feature-gates StreamingProxyRedirects=true to apiserver cmd line

## 6. Label/Taint the new Infranetes node

on a macine that can use kubectl to manage the kubernetes cluster label and taint this node

 ```bash
 $ kubectl taint node <name> infranetes=true:NoSchedule
 $ kubectl label node <name> infrantes=true
 ```
---
Congratulations, you should know have a working kubernetes cluster that can selected pods into independent VMs

## 6. AWS Configuration modifications

In order for these VMs to be usable on the internet, i.e. to fetch docker immage, the kubernetes vpc they use has to be configured to get an ipv4 IP

namely, in the vpc/subnet configuration, one selects the vpc-kubernetes subnet and uses modify auto-assign ip-settings to ensure its enabled.
