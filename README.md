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
