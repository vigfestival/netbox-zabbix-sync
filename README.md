A script to sync the Netbox device inventory to Zabbix.

## Requires pyzabbix and pynetbox.

### Script settings
#### Enviroment variables

* ZABBIX_HOST="https://zabbix.local"
* ZABBIX_USER="username"
* ZABBIX_PASS="Password"
* NETBOX_HOST="https://netbox.local"
* NETBOX_TOKEN="secrettoken"

#### Flags
|  Flag | Option  |  Description |
| ------------ | ------------ | ------------ |
|  -c | cluster | For clustered devices: only add the primary node of a cluster and use the cluster name as hostname. |
|  -H | hostgroup | Create non-existing hostgroups in Zabbix. Usefull for a first run to add all required hostgroups. |
|  -t | tenant | Add the tenant name to the hostgroup format (Tenant/Site/Manufacturer/Role) |
|  -v | Verbose | Log with debugging on. |


#### Logging
Logs are generated under sync.log, use -v for debugging.

#### Hostgroups: manual mode

In case of omitting the -H flag, manual hostgroup creation is required for devices in a new category.

This is in the format:
{Site name}/{Manufacturer name}/{Device role name}
And with tenants (-t flag):
{Tenant name}/{Site name}/{Manufacturer name}/{Device role name}

Make sure that the Zabbix user has proper permissions to create hosts.
The hostgroups are in a nested format. This means that proper permissions only need to be applied to the site name hostgroup and cascaded to any child hostgroups.

### Netbox settings
#### Custom fields
Use the following custom fields in Netbox to map the Zabbix URL:
* Type: Integer
* Name: zabbix_hostid
* Required: False
* Default: null
* Object: dcim > device

And this field for the Zabbix template

* Type: Text
* Name: zabbix_template
* Required: False
* Default: null
* Object: dcim > device_type

#### Set interface parameters within Netbox
When adding a new device, you can set the interface type with custom context.
Due to Zabbix limitations of changing interface type with a linked template, changing the interface type from within Netbox is not supported and the script will generate an error.

For example when changing a SNMP interface to an Agent interface:
```
Netbox-Zabbix-sync - WARNING - Device: Interface OUT of sync.
Netbox-Zabbix-sync - ERROR - Device: changing interface type to 1 is not supported.
```

To configure the interface parameters you'll need to use custom context. Custom context was used to make this script as customizable as posible for each environment. For example, you could:
 * Set the custom context directly on a device
 * Set the custom context on a label, which you would add to a device (for instance, SNMPv3)
 * Set the custom context on a device role
 * Set the custom context on a site or region

##### Agent interface configuration example
```json
{
    "zabbix": {
        "interface_port": 1500,
        "interface_type": 1
    }
}
```
##### SNMPv2 interface configuration example
```json
{
    "zabbix": {
        "interface_port": 161,
        "interface_type": 2,
        "snmp": {
            "bulk": 1,
            "community": "SecretCommunity",
            "version": 2
        }
    }
}
```
##### SNMPv3 interface configuration example
```json
{
    "zabbix": {
        "interface_port": 1610,
        "interface_type": 2,
        "snmp": {
            "authpassphrase": "SecretAuth",
            "bulk": 1,
            "securitylevel": 1,
            "securityname": "MySecurityName",
            "version": 3
        }
    }
}
```
Note: Not all SNMP data is required for a working configuration. [The following parameters are allowed ](https://www.zabbix.com/documentation/current/manual/api/reference/hostinterface/object#details_tag "The following parameters are allowed ")but are not all required, depending on your environment.

#### Permissions
Make sure that the user has proper permissions for device read and modify (modify to set the Zabbix HostID custom field) operations.

#### Custom links
To make the user experience easier you could add a custom link that redirects users to the Zabbix latest data.

* Name: zabbix_latestData
* Text: {% if obj.cf["zabbix_hostid"] %}Show host in Zabbix{% endif %}
* URL: {ZABBIX_URL} /zabbix.php?action=latest.view&filter_hostids[]={{ obj.cf["zabbix_hostid"] }}&filter_application=&filter_select=&filter_set=1