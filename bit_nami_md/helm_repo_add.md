# bitnami and fedora
~~~
helm repo add bitnami oci://registry-1.docker.io/bitnamichartsmy_user@fedora:~/bn20250807$ 
my_user@fedora:~/bn20250807$ helm repo add bitnami oci://registry-1.docker.io/bitnamicharts
bash: helm: command not found...
Install package 'helm' to provide command 'helm'? [N/y] N


my_user@fedora:~/bn20250807$ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
my_user@fedora:~/bn20250807$ ls -la
total 12
drwxr-xr-x. 1 my_user my_user    22 Aug  7 11:30 .
drwx------. 1 my_user my_user  5804 Aug  7 11:11 ..
-rw-r--r--. 1 my_user my_user 11913 Aug  7 11:30 get_helm.sh
my_user@fedora:~/bn20250807$ chmod 700 get_helm.sh
my_user@fedora:~/bn20250807$ ./get_helm.sh
Downloading https://get.helm.sh/helm-v3.18.4-linux-amd64.tar.gz
Verifying checksum... Done.
Preparing to install helm into /usr/local/bin
[sudo] password for my_user: 
helm installed into /usr/local/bin/helm
my_user@fedora:~/bn20250807$ helm version
version.BuildInfo{Version:"v3.18.4", GitCommit:"d80839cf37d860c8aa9a0503fe463278f26cd5e2", GitTreeState:"clean", GoVersion:"go1.24.4"}
my_user@fedora:~/bn20250807$ helm repo add bitnami oci://registry-1.docker.io/bitnamicharts
Error: looks like "oci://registry-1.docker.io/bitnamicharts" is not a valid chart repository or cannot be reached: failed to perform "FetchReference" on source: invalid reference
my_user@fedora:~/bn20250807$ helm version
version.BuildInfo{Version:"v3.18.4", GitCommit:"d80839cf37d860c8aa9a0503fe463278f26cd5e2", GitTreeState:"clean", GoVersion:"go1.24.4"}
my_user@fedora:~/bn20250807$ docker --version
bash: docker: command not found...
Packages providing this file are:
'docker-cli'
'podman-docker'
my_user@fedora:~/bn20250807$ helm repo add bitnami https://charts.bitnami.com/bitnami
Error: looks like "https://charts.bitnami.com/bitnami" is not a valid chart repository or cannot be reached: failed to fetch https://charts.bitnami.com/bitnami/index.yaml : 403 Forbidden
my_user@fedora:~/bn20250807$ cat ~/exch/cmd.txt 
export ALL_PROXY="socks5://localhost:8090"
export HTTPS_PROXY="socks5://localhost:8090"
export HTTP_PROXY="socks5://localhost:8090"

helm repo add bitnami oci://re^C
my_user@fedora:~/bn20250807$ export ALL_PROXY="socks5://localhost:8090"
my_user@fedora:~/bn20250807$ export HTTPS_PROXY="socks5://localhost:8090"
my_user@fedora:~/bn20250807$ export HTTP_PROXY="socks5://localhost:8090"
my_user@fedora:~/bn20250807$ helm repo add bitnami https://charts.bitnami.com/bitnami
"bitnami" has been added to your repositories
my_user@fedora:~/bn20250807$ helm repo list
NAME   	URL                               
bitnami	https://charts.bitnami.com/bitnami

~~~
