# Installing 3scale AMP 2.1 on OCP 3.6 on ec2

The instructions are borrowed from Dari's [gist](https://gist.github.com/mayorova/6fb9733416870986b775dff225248fef) and changed per OCP 3.6 installation


You will need:

- a running instance with 8GB RAM minimum (recommended 16GB) and RHEL

- **`<PUBLIC_DNS>`**: (e.g. `ec2-54-123-456-78.compute-1.amazonaws.com`)

- **`<PUBLIC_IP>`**: (e.g. `54.123.456.78`)

## Set up OpenShift cluster 

### References

- https://github.com/openshift/origin/blob/master/docs/cluster_up_down.md#linux

(thanks to Toni Syvänen)
- https://github.com/openshift-evangelists/oc-cluster-wrapper
- https://access.redhat.com/downloads/content/290/ver=3.4/rhel---7/3.4.1.10/x86_64/product-software

### Install and run Docker

```
sudo yum-config-manager --enable rhui-REGION-rhel-server-extras
sudo yum install docker docker-registry -y
```

`/etc/sysconfig/docker`:
```
INSECURE_REGISTRY='--insecure-registry 172.30.0.0/16'
```
```
sudo systemctl start docker
sudo systemctl status docker
```

### Install OC tools

**OCP:** https://mirror.openshift.com/pub/openshift-v3/clients/3.6.173.0.21/linux/oc.tar.gz

Example:
```
sudo yum install wget -y
wget https://mirror.openshift.com/pub/openshift-v3/clients/3.6.173.0.21/linux/oc.tar.gz
tar xzvf oc.tar.gz
mv oc /usr/bin/
rm -rf oc.tar.gz
```

### Start the cluster

```
oc cluster up --public-hostname=<PUBLIC_DNS> --routing-suffix=<PUBLIC_IP>.xip.io --version=latest --host-data-dir=/home/ec2-user/myoccluster1
```

Check out the console:
`https://<PUBLIC_DNS>:8443`

## Deploy 3scale AMP

### Create persistent volumes

```
sudo su

mkdir -p  /var/lib/docker/pv/{01..04}
chmod g+w /var/lib/docker/pv/{01..04}
chcon -Rt svirt_sandbox_file_t /var/lib/docker/pv/
```

(`pv.yml` - copy from resources folder)

```
oc login -u system:admin

oc new-app --param PV=01 -f pv.yml
oc new-app --param PV=02 -f pv.yml
oc new-app --param PV=03 -f pv.yml
oc new-app --param PV=04 -f pv.yml

oc get pv
```

(`amp.yml` - copy from resources folder)


### Login as developer and start AMP with template


```
oc login https://<PUBLIC_DNS>:8443 --insecure-skip-tls-verify

oc new-project 3scale-amp

oc new-app --file amp.yml --param WILDCARD_DOMAIN=<PUBLIC_IP>.xip.io
```

```
--> Deploying template "3scale-amp/system" for "amp.yml" to project 3scale-amp

     system
     ---------
     Login on https://3scale-admin.<PUBLIC_IP>.xip.io as admin/gu8edykg    <===== LOGIN with these credentials

     * With parameters:
        * AMP_RELEASE=er3
        * ADMIN_PASSWORD=gu8edykg # generated
        * ADMIN_USERNAME=admin
        * APICAST_ACCESS_TOKEN=rthdeuql # generated
        * ADMIN_ACCESS_TOKEN=4o2txf0v4e3wgvtw # generated
        * WILDCARD_DOMAIN=<PUBLIC_IP>.xip.io
        * SUBDOMAIN=3scale
        * MySQL User=mysql
        * MySQL Password=qfnt75jf # generated
        * MySQL Database Name=system
        * MySQL Root password.=7dhquse7 # generated
        * SYSTEM_BACKEND_USERNAME=3scale_api_user
        * SYSTEM_BACKEND_PASSWORD=a3i3n7by # generated
        * REDIS_IMAGE=rhscl/redis-32-rhel7:3.2-5.3
        * SYSTEM_BACKEND_SHARED_SECRET=s4wpndxj # generated
```

### Test

Log in to the portal using the credentials above, configure the API, deploy APIcast staging adn production.

### Configure emails (optional)

```
oc env dc/system-app --overwrite SMTP_ADDRESS=smtp.gmail.com SMTP_USER_NAME=<SMTP_USERNAME> SMTP_PASSWORD=<SMTP_PASSWORD>
oc env dc/system-redis --overwrite SMTP_ADDRESS=smtp.gmail.com SMTP_USER_NAME=<SMTP_USERNAME> SMTP_PASSWORD=<SMTP_PASSWORD>
oc env dc/system-sidekiq --overwrite SMTP_ADDRESS=smtp.gmail.com SMTP_USER_NAME=<SMTP_USERNAME> SMTP_PASSWORD=<SMTP_PASSWORD>
```

Note: the emails will be sent from the user specified in `<SMTP_USERNAME>`

## Set up additional APIcast on OpenShift cluster

### Create Access Token

Create an access token for Account Management API (Read permission is enough) (`<ACCESS_TOKEN>`)

### Deploy APIcast pointing to the AMP backend

```
oc login https://<PUBLIC_DNS>:8443 --insecure-skip-tls-verify

oc new-project "apicast" --display-name="new-apicast-gateway" --description="3scale apicast gateway"

oc secret new-basicauth apicast-configuration-url-secret --password=https://<ACCESS_TOKEN>@3scale-admin.<PUBLIC_IP>.xip.io

oc new-app --file apicast.yml

oc env dc/apicast --overwrite BACKEND_ENDPOINT_OVERRIDE=https://backend.<PUBLIC_IP>.xip.io

```

### Create a route in OpenShift for the new APIcast

- Create a new Public Base URL in the Integration page
- Add the corresponding route to the `apicast` service
