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

The same model is adapted for openstack proxying.

## Create a hosts file for the Bright cluster
It needs an entry for the master and the controllers.  Something like this should do.
Note that the ansible rules need to run as root.  Typically admins have sudo
on the master but may not have sudo on the compute nodes.  One work around is
to put the user running ansible's ssh public key in the root account of the
controller.

## Commands to run to enable proxy config

The first step is to update the openstack dashboard on the controllers
```
ansible-playbook -i hosts horizon_newroot.yaml
```

The next step is to update the http hosting configuration on the master to
reach the updated dashboard configuration.
```
ansible-playbook -i hosts proxy-services.yaml -K -b
```
