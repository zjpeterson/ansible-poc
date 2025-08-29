# Custom Credential Types

Commonly-needed [Credential Types](https://docs.ansible.com/automation-controller/latest/html/userguide/credential_types.html) for Ansible Controller that don't come with the base install.

Most of the Input sections are pretty similar to one another. The important part is the **Injector**, which needs to expose login secrets in a way that is expected by the modules you intend to use. See each Collection's documentation to find out what your options are. I prefer Environment methods when possible - this avoids a need to put anything into your playbook code for authentication.

## Generic
### Input
```yaml
fields:
  - id: username
    type: string
    label: Username
  - id: password
    type: string
    label: Password
    secret: true
required:
  - username
  - password
```
### Injector
```yaml
extra_vars:
  cred_username: '{{ username }}'
  cred_password: '{{ password }}'
```

## ServiceNow
### Input
```yaml
fields:
  - id: host
    type: string
    label: Host
  - id: username
    type: string
    label: Username
  - id: password
    type: string
    label: Password
    secret: true
required:
  - host
  - username
  - password
```
### Injector
```yaml
env:
  SN_HOST: "{{ host }}"
  SN_USERNAME: "{{ username }}"
  SN_PASSWORD: "{{ password }}"
```

## Cisco ACI
Notes:
- You can also use the built-in `Network` type for ACI, but with that you will not be able to build in values for `host` or `validate_certs`.
- This is a basic implentation for username/password auth. The `cisco.aci` collection supports key/cert auth as well, you would just need to update the Credential Type to supply those values.
### Input
```yaml
fields:
  - id: host
    type: string
    label: Host
  - id: username
    type: string
    label: Username
  - id: password
    type: string
    label: Password
    secret: true
  - id: validate_certs
    type: boolean
    label: Verify SSL
required:
  - username
  - password
```
### Injector
```yaml
env:
  ACI_HOST: "{{ host }}"
  ACI_USERNAME: "{{ username }}"
  ACI_PASSWORD: "{{ password }}"
  ACI_VALIDATE_CERTS: "{{ validate_certs }}"
```

## F5
### Input
```yaml
fields:
  - id: server
    type: string
    label: Server
  - id: username
    type: string
    label: Username
  - id: password
    type: string
    label: Password
    secret: true
  - id: validate_certs
    type: boolean
    label: Verify SSL
required:
  - username
  - password
```
### Injector
```yaml
env:
  F5_SERVER: "{{ server }}"
  F5_USER: "{{ username }}"
  F5_PASSWORD: "{{ password }}"
  F5_VALIDATE_CERTS: "{{ validate_certs }}"
```

## Netbox
### Input
```yaml
fields:
  - id: api
    type: string
    label: API Endpoint
  - id: token
    type: string
    label: API Token
    secret: true
required:
  - api
  - token
```
### Injector
```yaml
env:
  NETBOX_API: '{{ api }}'
  NETBOX_TOKEN: '{{ token }}'
```
