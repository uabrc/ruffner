# Project for devcluster

Document the configuration and install of the development
cluster for the RCS.

## Proxy Service configuration

Follow the [Bright KB config](http://kb.brightcomputing.com/faq/index.php?action=artikel&cat=9&id=291)
to provide access to cluster services
via standard port 443 by setting up proxy configs to the
default port maps for bright-view, userportal, and openstack services.

The KB addresses bright-view and userportal via a rewrite rule
to move requests for the service to https transport and then
maps https requests to a proxy config to the backend. This default config
leverages the stock Bright httpd config that inherits port 80 configuration
from the server-level config. It has an explicit ssl.conf via a VirtualHost
section.  Because the re-write rules implictly exist in the server level (port
80) config these rules are not triggered on the ssl VirtualHost leaving only
the ProxyPass config.  Because the userportal and bright-view are exposed
via http and https by default via cmd the ProxyPass works seemlessly against
the ssl-based backend.

The same model is adapted for openstack proxying.  In order for this model to
work for openstack from bcm, the openstack service must first be hosted as
an SSL service via haproxy so that the ProxyPass rules in Apache maintain the
https protocol restriction.  In addition, the OpenStack dashboard django app
must be updated to be served under the /openstack/ url prefix.  This involves
changes to [nginx config](http://nginx.org/en/docs/http/ngx_http_core_module.html#alias)
 that serves the openstack-dashboard app so nginx recognizes the path prefix and builds correct file paths for static content, the [local_settings for the openstack_dashboard](https://docs.djangoproject.com/en/2.2/topics/settings/#django-settings) so django returns prefixed urls,  and the [urlpatterns defined as part of django app dispatcher config](https://docs.djangoproject.com/en/1.11/topics/http/urls/) so that the app recognizes the new path prefix.



## Create a hosts file for the Bright cluster
It needs an entry for the master and the controllers.  Something like this should do.
Note that the ansible rules need to run as root.  Typically admins have sudo
on the master but may not have sudo on the compute nodes.  One work around is
to put the user running ansible's ssh public key in the root account of the
controller.

A host file like the following can work:
```
[master]
master.example.org

[controller]
c1 ansible_host=master.example.org+controller_host ansible_user=root
```

In the event that you want to run the playbook in a `chroot`, for example, apply to the OpenStack controller node image (assuming that the repo has been staged to `/cm/images/<image>/root/uabrc/ruffner`):

```shell
$ sudo mkdir /cm/images/<image>/dev/shm
$ sudo mount --bind /dev/shm /cm/images/<image>/dev/shm
$ sudo chroot /cm/images/<image>
# cd /root/uabrc/ruffner
# cat << EOF > hosts
[controller]
localhost ansible_connection=local
EOF

# ansible-playbook  -i hosts -l "localhost,"  horizon-newroot.yaml

# exit
$ sudo umount /cm/images/<image>/dev/shm
$ sudo rmdir /cm/images/<image>/dev/shm
```

## Define an SSL certificate bundle

Serving content via SSL requires a cert bundle for haproxy
and other services.  You can build a cert bundle with your host key and any
intermidiate certificates up to the root CA.  For an InCommon CA key this file
can be created in the root users directory of the master node.
```
cat host.key host.crt InCommon_RSA_Server_CA.pem USERTrust_RSA_Certification_Authority.pem AddTrust_External_CA_Root.pem > cert-bundle.pem
```
The ansible tasks will refernence this at the path `/root/cert-bundle.pem`.

## Commands to run to enable proxy config

The first step is to update the openstack dashboard on the controllers
```
ansible-playbook -i hosts horizon-newroot.yaml
```

The next step is to update the http hosting configuration on the master to
reach the updated dashboard configuration.
```
ansible-playbook -i hosts proxy-services.yaml -K -b
```

## Set publicURL to a FQDN

If the publicURL is an IP it can be impossible to reach the service from multiple networks.
Setting it to a FQDN allows name resolution to determine the correct IP.

Update the image and running cluster nodes the the desired FQDN.
```
ansible-playbook -i hosts add-publicurl-build.yaml
ansible-playbook -i hosts fix-publicurl-ops.yaml
```


Update the openstack configuration. Make sure you have admin privileges
```
openstack endpoint list --interface public -f value -c ID -c URL | sed -e 's/192\.168\.16\.10/my.FQDN/'
```

Then simply replace the urls with a loop of commands like:
```
openstack endpoint set --url <endpoint_url> <endpoint_id>
```
