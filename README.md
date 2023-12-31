# ldap_custom_mapping - LDAP user search

A [ColdFront](https://coldfront.readthedocs.io/en/latest/) plugin that extends the functionality of the existing [`ldap_user_search`](https://github.com/ubccr/coldfront/tree/main/coldfront/plugins/ldap_user_search) plugin, allowing the user to create a custom mapping from LDAP attributes to ColdFront user attributes. Also adds TLS support.

## Requirements

- `pip install python-ldap ldap3`

## Installation

If you're using a virtual environment (following ColdFront's deployment instructions should have you make and use a virtual environment), make sure you're in the virutal environment first(`source /srv/coldfront/venv/bin/activate`).

`pip install git+https://github.com/cecilialau6776/ldap_custom_mapping`

To enable this plugin, set the following applicable variables in ColdFront's [local settings](https://coldfront.readthedocs.io/en/latest/config/#configuration-files):

```py
# /etc/coldfront/local_settings.py
INSTALLED_APPS += ["coldfront_plugin_ldap_custom_mapping"]
```

## Configuration
Also in local settings, set the following applicable variables.
| Option | Default | Description |
| --- | --- | --- |
| `LDAP_USER_SEARCH_SERVER_URI` | N/A | URI for the LDAP server, required |
| `LDAP_USER_SEARCH_BASE` | N/A | Search base, required |
| `LDAP_USER_SEARCH_BIND_DN` | None | Bind DN |
| `LDAP_USER_SEARCH_BIND_PASSWORD` | None | Bind Password |
| `LDAP_USER_SEARCH_CONNECT_TIMEOUT` | 2.5 | Time in seconds before the connection times out |
| `LDAP_USER_SEARCH_USE_SSL` | True | Whether or not to use SSL |
| `LDAP_USER_SEARCH_USE_TLS` | False | Whether or not to use TLS |
| `LDAP_USER_SEARCH_SASL_MECHANISM` | None | One of `"EXTERNAL"`, `"DIGEST-MD5"`, `"GSSAPI"`, or `None` |
| `LDAP_USER_SEARCH_SASL_CREDENTIALS` | None | SASL authorization identity string. If you don't have one and `None` doesn't work, try `""`. |
| `LDAP_USER_SEARCH_PRIV_KEY_FILE` | None | Path to the private key file |
| `LDAP_USER_SEARCH_CERT_FILE` | None | Path to the certificate file |
| `LDAP_USER_SEARCH_CACERT_FILE` | None | Path to the CA certificate file |
| `LDAP_USER_SEARCH_ATTRIBUTE_MAP` | `{"username": "uid", "last_name": "sn", "first_name": "givenName", "email": "mail"}` | A mapping from ColdFront user attributes to LDAP attributes. |
| `LDAP_USER_SEARCH_MAPPING_CALLBACK` | See below. | Function that maps LDAP search results to ColdFront user attributes. See more below. |

`LDAP_USER_SEARCH_MAPPING_CALLBACK` default:
```py
def parse_ldap_entry(attribute_map, entry_dict):
    user_dict = {}
    for user_attr, ldap_attr in attribute_map.items():
        user_dict[user_attr] = entry_dict.get(ldap_attr)[0] if entry_dict.get(ldap_attr) else ''
    return user_dict
```


For custom attribute mapping, set the Django variable `LDAP_USER_SEARCH_ATTRIBUTE_MAP` in local settings.
```py
# /etc/coldfront/local_settings.py
# default
LDAP_USER_SEARCH_ATTRIBUTE_MAP = {
    "username": "uid",
    "last_name": "sn",
    "first_name": "givenName",
    "email": "mail",
}
```

To set a custom mapping, define an `LDAP_USER_SEARCH_MAPPING_CALLBACK` function with parameters `attr_map` and `entry_dict` that returns a dictionary mapping ColdFront User attributes to their values. `attr_map` is just `LDAP_USER_SEARCH_ATTRIBUTE_MAP`, and `entry_dict` is further explained below.

For example, if your LDAP schema provides a full name and no first and last name attributes, you can define `LDAP_USER_SEARCH_ATTRIBUTE_MAP` and `LDAP_USER_SEARCH_MAPPING_CALLBACK` as follows:

```py
# /etc/coldfront/local_settings.py
LDAP_USER_SEARCH_ATTRIBUTE_MAP = {
    "username": "uid",
    "email": "mail",
    "full_name": "cn",
}

def LDAP_USER_SEARCH_MAPPING_CALLBACK(attr_map, entry_dict):
    user_dict = {
        "username": entry_dict.get(attr_map["username"])[0],
        "email": entry_dict.get(attr_map["email"])[0],
        "first_name": entry_dict.get(attr_map["full_name"])[0].split(" ")[0],
        "last_name": entry_dict.get(attr_map["full_name"])[0].split(" ")[-1],
    }
    return user_dict
```

`entry_dict` is provided as a dictionary mapping from the LDAP attribute to a list of values.
```py
entry_dict = {
    'mail': ['jane.emily.doe@example.com'],
    'cn': ['Jane E Doe'],
    'uid': ['janedoe1234']
}
```

If this was the input to the above callback, `user_dict` would look like this:
```py
user_dict = {
    "username": "janedoe1234",
    "email": "jane.emily.doe@example.com",
    "first_name": "Jane",
    "last_name": "Doe",
}
```

