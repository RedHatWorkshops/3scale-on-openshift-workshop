# 3scale Gateway On OpenSHift

These are the steps I followed to in installing 3scale On Prem - largely following  [this](https://support.3scale.net/guides/infrastructure/onpremises20-installation)

## Background
Linux machine setup

* AWS: OHIO - TOM_ON_PREM_OFFICIAL
* Machine size: c4.2xlarge
* Machine host is: ec2-13-58-121-70.us-east-2.compute.amazonaws.com

## Login to machine.

```
ssh -i ./redhat-3scale-key-pair-ohio.pem ec2-user@ec2-13-58-121-70.us-east-2.compute.amazonaws.com
```
## Install Docker

Install docker package
```
sudo yum repolist all
sudo yum-config-manager --enable rhui-REGION-rhel-server-extras
sudo yum install docker docker-registry -y
```

Edit the `/etc/sysconfig/docker` file's option `INSECURE_REGISTRY` to look like this
```
INSECURE_REGISTRY='--insecure-registry 172.30.0.0/16'
```			

Restart docker
```
sudo systemctl start docker
sudo systemctl status docker
```
## Install Client tools

```
cd ~
sudo yum install wget -y
wget https://github.com/openshift/origin/releases/download/v1.4.1/openshift-origin-client-tools-v1.4.1-3f9807a-linux-64bit.tar.gz
tar xzvf openshift-origin-client-tools-v1.4.1-3f9807a-linux-64bit.tar.gz
sudo mv openshift-origin-client-tools-v1.4.1+3f9807a-linux-64bit/oc /usr/bin/
rm -rf openshift-origin-client-tools-v1.4.1*
```

Start the cluster
NOTES for the command below
1. : the hostname is the fully qualified host name of your instance
2. The routing suffix is the <PublicIP>.xip.io of your ec2 instance

```
sudo oc cluster up --public-hostname=ec2-13-58-121-70.us-east-2.compute.amazonaws.com --routing-suffix=13.58.121.70.xip.io
```

Web console: 	https://ec2-13-58-121-70.us-east-2.compute.amazonaws.com:8443

## Register nodes

Attach node(s) to the 3scale instance.

```
sudo subscription-manager register --username <yours>@redhat.com --password <yours>
```
Get pool id like this: 

```
sudo subscription-manager list --available --matches "*3scale API Management*"
```

Mine was `8a85f9815790899701579abe35b503c3`
```
sudo subscription-manager attach --pool=8a85f9815790899701579abe35b503c3
```
    
## Fix permissions issues

There are issues with permissions...fix those and install git
```
sudo su
yum-config-manager --disable rhel-7-server-rt-beta-rpms
yum-config-manager --save --setopt=rhel-7-server-htb-rpms.skip_if_unavailable=true
```

I had to do these 2 previous steps as the console advised me to - I was getting 403 permissions errors.

```
yum install git -y
```

# Storage

Configure persistent storage on a file system that supports multiple writes.

```
mkdir -p  /var/lib/docker/pv/{01..04}
chmod g+w /var/lib/docker/pv/{01..04}
chcon -Rt svirt_sandbox_file_t /var/lib/docker/pv/
```

locally - copy up pv.yml to AWS
```
scp -i ./redhat-3scale-key-pair-ohio.pem ./pv.yml ec2-user@ec2-13-58-121-70.us-east-2.compute.amazonaws.com:/home/ec2-user	
```

Make sure you are still sudo su - before running the following. Otherwise you might run into permissions problems.
```
oc login -u system:admin		
oc new-app --param PV=01 -f pv.yml
oc new-app --param PV=02 -f pv.yml
oc new-app --param PV=03 -f pv.yml
oc new-app --param PV=04 -f pv.yml
```
Verify with
```
oc get pv
```
## Enable access

Enable access to the `rhel-7-server-3scale-amp-2.0-rpms` repository using the subscription manager
	
```
sudo subscription-manager repos --enable=rhel-7-server-3scale-amp-2.0-rpms
sudo yum install 3scale-amp-template -y
```
Login using developer/developer
```
oc login
oc new-project 3scale-amp
```

NOTES before running below command:
1. The amp.yml is automatically downloaded
2. The IP referred to below is again the Public IP of your EC2 instance

```
oc new-app --file /opt/amp/templates/amp.yml --param WILDCARD_DOMAIN=amp.13.58.121.70.xip.io  --param ADMIN_PASSWORD=3scaleUser
```

## Notes

It can take a few minutes after all pods are started before it is ready. Make sure all pods are up by going to the openshift console before you login using the below link

When installation is complete, Login on https://3scale-admin.amp.13.58.121.70.xip.io 	as admin/3scaleUser
