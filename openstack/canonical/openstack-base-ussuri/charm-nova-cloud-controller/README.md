# Overview

Cloud controller node for OpenStack nova. Contains nova-schedule, nova-api,
nova-network and nova-objectstore.

If console access is required then console-proxy-ip should be set to a client
accessible IP that resolves to the nova-cloud-controller. If running in HA mode
then the public vip is used if console-proxy-ip is set to local. Note: The
console access protocol is baked into a guest when it is created, if you change
it then console access for existing guests will stop working

# Usage

## High availability

When more than one unit is deployed with the [hacluster][hacluster-charm]
application the charm will bring up an HA active/active cluster.

There are two mutually exclusive high availability options: using virtual IP(s)
or DNS. In both cases the hacluster subordinate charm is used to provide the
Corosync and Pacemaker backend HA functionality.

See [OpenStack high availability][cdg-ha-apps] in the [OpenStack Charms
Deployment Guide][cdg] for details.

## Spaces

This charm supports the use of Juju Network Spaces, allowing the charm to be
bound to network space configurations managed directly by Juju.  This is only
supported with Juju 2.0 and above.

API endpoints can be bound to distinct network spaces supporting the network
separation of public, internal and admin endpoints.

Access to the underlying MySQL instance can also be bound to a specific space
using the shared-db relation.

To use this feature, use the --bind option when deploying the charm:

    juju deploy nova-cloud-controller --bind \
       "public=public-space \
        internal=internal-space \
        admin=admin-space \
        shared-db=internal-space"

Alternatively, these can also be provided as part of a Juju native bundle
configuration:

```yaml
    nova-cloud-controller:
      charm: cs:xenial/nova-cloud-controller
      num_units: 1
      bindings:
        public: public-space
        admin: admin-space
        internal: internal-space
        shared-db: internal-space
```

NOTE: Spaces must be configured in the underlying provider prior to attempting
to use them.

NOTE: Existing deployments using os-*-network configuration options will
continue to function; these options are preferred over any network space
binding provided if set.

## Default Quota Configuration

This charm supports default quota settings for projects. This feature is only
available from OpenStack Icehouse and later releases.

The default quota settings do not overwrite post-deployment CLI quotas set by
operators. Existing projects whose quotas were not modified will adopt the new
defaults when a config-changed hook occurs. Newly created projects will also
adopt the defaults set in the charm's config.

By default, the charm's quota configs are not set and OpenStack projects have
the below default values:

    quota-instances : 10
    quota-cores : 20
    quota-ram : 51200
    quota-metadata_items : 128
    quota-injected_files : 5
    quota-injected_file_content_bytes : 10240
    quota-injected_file_path_length : 255
    quota-key_pairs : 100
    quota-server_groups : 10 (available since Juno)
    quota-server_group_members : 10 (available since Juno)

## SSH knownhosts caching

This section covers the option involving the caching of SSH host lookups
(knownhosts) on each nova-compute unit. Caching of SSH host lookups speeds up
deployment of nova-compute units when first deploying a cloud, and when adding
a new unit.

There is a Boolean configuration key `cache-known-hosts` that ensures that any
given host lookup to be performed just once. The default is `true` which means
that caching is performed.

> **Note**: A cloud can be deployed with the `cache-known-hosts` key set to
  `false`, and be set to `true` post-deployment. At that point the hosts will
  have been cached. The key only controls whether the cache is used or not.

If the above key is set, a new Juju action `clear-unit-knownhost-cache` is
provided to clear the cache. This can be applied to a unit, service, or an
entire nova-cloud-controller application. This would be needed if DNS
resolution had changed in an existing cloud or during a cloud deployment. Not
clearing the cache in such cases could result in an inconsistent set of
knownhosts files.

This action will cause DNS resolution to be performed (for
unit/service/application), thus potentially triggering a relation-set on the
nova-cloud-controller unit(s) and subsequent changed hook on the related
nova-compute units.

The action is used as follows, based on unit, service, or application,
respectively:

    juju run-action nova-cloud-controller/0 clear-unit-knownhost-cache target=nova-compute/2
    juju run-action nova-cloud-controller/0 clear-unit-knownhost-cache target=nova-compute
    juju run-action nova-cloud-controller/0 clear-unit-knownhost-cache

In a high-availability setup, the action must be run on all
`nova-cloud-controller` units.

## Policy Overrides

Policy overrides is an **advanced** feature that allows an operator to override
the default policy of an OpenStack service. The policies that the service
supports, the defaults it implements in its code, and the defaults that a charm
may include should all be clearly understood before proceeding.

> **Caution**: It is possible to break the system (for tenants and other
  services) if policies are incorrectly applied to the service.

Policy statements are placed in a YAML file. This file (or files) is then (ZIP)
compressed into a single file and used as an application resource. The override
is then enabled via a Boolean charm option.

Here are the essential commands (filenames are arbitrary):

    zip overrides.zip override-file.yaml
    juju attach-resource nova-cloud-controller policyd-override=overrides.zip
    juju config nova-cloud-controller use-policyd-override=true

See appendix [Policy Overrides][cdg-appendix-n] in the [OpenStack Charms
Deployment Guide][cdg] for a thorough treatment of this feature.

# Bugs

Please report bugs on [Launchpad][lp-bugs-charm-nova-cloud-controller].

For general charm questions refer to the OpenStack [Charm Guide][cg].

<!-- LINKS -->

[cg]: https://docs.openstack.org/charm-guide
[cdg]: https://docs.openstack.org/project-deploy-guide/charm-deployment-guide
[cdg-appendix-n]: https://docs.openstack.org/project-deploy-guide/charm-deployment-guide/latest/app-policy-overrides.html
[lp-bugs-charm-nova-cloud-controller]: https://bugs.launchpad.net/charm-nova-cloud-controller/+filebug
[cdg-ha-apps]: https://docs.openstack.org/project-deploy-guide/charm-deployment-guide/latest/app-ha.html#ha-applications
[hacluster-charm]: https://jaas.ai/hacluster
