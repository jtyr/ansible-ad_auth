ad_auth
=======

Role which helps to configure AD authentication.


Example
-------

```
---

- name: Exampple of how to use the ad_auth role
  hosts: myhost1
  vars:
    # You should configure krb
    krb_domain: "{{ ansible_domain }}"
    krb_server: "{{ krb_domain }}"
    krb_config_domain_realm__pre: .{{ ansible_hostname }} = {{ krb_domain | upper }}

    # You should configure nsswitch
    nsswitch__options: &nsswitch__options
      - files
      - sss
    nsswitch_automount: *nsswitch___options
    nsswitch_group: *nsswitch___options
    nsswitch_netgroup: *nsswitch___options
    nsswitch_passwd: *nsswitch___options
    nsswitch_services: *nsswitch___options
    nsswitch_shadow: *nsswitch___options

    # You might also need to customize the samba config
    samba_service_enable: no
    samba_service_start: no
    samba_config:
      global:
        client signing: "yes"
         client use spnego: "yes"
        kerberos method: secrets and keytab
        log file: /var/log/samba/%m.log
        netbios name: "{{ (ansible_hostname | hash('sha1'))[0:14] | upper }}"
        password server: "{{ sssd___primary_dc }}"
        realm: "{{ ansible_domain | upper }}"
        security: ads
        workgroup: "{{ ansible_domain | regex_replace('\\..*') | upper }}"

    # Configure the ad_auth role
    ad_auth___user: myuser
    ad_auth___password: myp4sw0rd
    ad_auth_join_cmd: >
      if [[ "x{{ ad_auth__force | default('') }}" != "x" ]]; then
        env rm -f /etc/krb5.keytab;
      fi;
      if [ ! -f /etc/krb5.keytab ]; then
        set -e;
        kdestroy;
        net ads leave -U '{{ ad_auth___user }}'%'{{ ad_auth___password }}' || true;
        env rm -f /tmp/krb5cc_*;
        service {{ samba_service }} start;
        net ads join -U '{{ ad_auth___user }}'%'{{ ad_auth___password }}' &&
        kinit -k -t /etc/krb5.keytab '{{ (ansible_hostname | hash('sha1'))[0:14] | upper }}$@{{ ansible_domain | upper }}' || false;
        service {{ samba_service }} stop;
      else
        exit 100;
      fi

    # You might also want to add some SSSD helpers
    oddjob_helper_pkgs:
      - oddjob-mkhomedir

    # You will need your own sssd config
    # (see: https://fedorahosted.org/sssd/wiki/Configuring_sssd_with_ad_server)
    sssd___primary_dc: myadserver01.{{ ansible_domain }}
    sssd_config:
      ...
  roles:
    - krb
    - nsswitch
    - samba
    - ad_auth
    - oddjob
    - sssd
```


Role variables
--------------

List of variables used by the role:

```
# Default authconfig options
ad_auth_authconfig_options__default:
  - "--enablesssd"
  - "--enablesssdauth"
  - "--enablemkhomedir"
  - "--enablelocauthorize"
  - "--update"
  - "--nostart"

# Custom authconfig options
ad_auth_authconfig_options__custom: []

# Final authconfig options
ad_auth_authconfig_options: "{{
  ad_auth_authconfig_options__default +
  ad_auth_authconfig_options__custom }}"

# Packages required to join the domain and/or get the machine token
# (explicit version can be specified here)
ad_auth_join_pkgs:
  # Contains kinit
  - krb5-workstation

# Command or script to join the domain and/or get the machine token
# (see an example in the README.md)
ad_auth_join_cmd: ""

# Expected return code from the command indicating no error
ad_auth_join_cmd_rc: 100

# Do no log the join command as it might contain passwords
ad_auth_join_no_log: yes
```


Dependencies
------------

- [krb](https://github.com/picotrading/ansible-krb) (optional)
- [nsswitch](https://github.com/picotrading/ansible-nsswitch) (optional)
- [oddjob](https://github.com/picotrading/ansible-oddjob) (optional)
- [samba](https://github.com/picotrading/ansible-samba) (optional)
- [sssd](https://github.com/picotrading/ansible-sssd) (optional)


License
-------

MIT


Author
------

Jiri Tyr
