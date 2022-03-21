## Openstack Neutron

### network segment

https://docs.openstack.org/neutron/latest/admin/config-network-segment-ranges.html

A set of `default` network segment ranges are created out of the values defined in the ML2 config file: `network_vlan_ranges` for ml2_type_vlan, `vni_ranges` for ml2_type_vxlan, `tunnel_id_ranges` for ml2_type_gre and `vni_ranges` for ml2_type_geneve. They will be reloaded when Neutron server starts or restarts. The `default` network segment ranges are `read-only`, but will be treated as any other `shared` ranges on segment allocation.

The administrator can use the default network segment range information to make shared and/or per-tenant range creation and assignment.