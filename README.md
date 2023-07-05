# ldap_custom_mapping - LDAP user search

A [ColdFront](https://coldfront.readthedocs.io/en/latest/) plugin that extends the functionality of the existing [`ldap_user_search`](https://github.com/ubccr/coldfront/tree/main/coldfront/plugins/ldap_user_search) plugin, allowing the user to create a custom mapping from LDAP attributes to ColdFront user attributes. Also adds TLS support.

## Requirements

- `pip install python-ldap ldap3`

## Installation

If you're using a virtual environment (following ColdFront's deployment instructions should have you make and use a virtual environment), make sure you're in the virutal environment first.

`pip install git+https://github.com/cecilialau6776/ldap_custom_mapping`

## Usage

To enable this plugin set the following applicable environment variables:

```env
PLUGIN_LDAP_USER_SEARCH=True
LDAP_USER_SEEACH_SERVER_URI=ldap://example.com
LDAP_USER_SEARCH_BASE="dc=example,dc=com"
LDAP_USER_SEARCH_BIND_DN="cn=Manager,dc=example,dc=com"
LDAP_USER_SEARCH_BASE="dc=example,dc=com"
LDAP_USER_SEARCH_USE_SSL=True
LDAP_USER_SEARCH_USE_TLS=True
LDAP_USER_SEARCH_CACERT_FILE=/path/to/cacert
LDAP_USER_SEARCH_CERT_FILE=/path/to/cert
LDAP_USER_SEARCH_PRIV_KEY_FILE=/path/to/key
```

For custom attributes, set the Django variable `LDAP_USER_SEARCH_ATTRIBUTE_MAP` in ColdFront's [local settings](https://coldfront.readthedocs.io/en/latest/config/#configuration-files). This dictionary maps from ColdFront User attributes to LDAP attributes:
```py
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

