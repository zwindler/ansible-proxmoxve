# Installation manuelle

https://pve.proxmox.com/wiki/Install_Proxmox_VE_on_Debian_Stretch

## Dans la console scaleway

* Créer une VM de type S
* Dans les options, retirer l'IPv6
* Modifier le fichier host

```bash
cat /etc/hosts
10.16.166.25    proxmox01 #IP interne
51.15.217.222   proxmox01.zwindler.fr #IP externe
127.0.0.1       localhost #remove proxmox01 from here
::1             localhost ip6-localhost ip6-loopback
ff02::1         ip6-allnodes
ff02::2         ip6-allrouters
```

echo "deb http://download.proxmox.com/debian/pve stretch pve-no-subscription" > /etc/apt/sources.list.d/pve-install-repo.list
wget http://download.proxmox.com/debian/proxmox-ve-release-5.x.gpg -O /etc/apt/trusted.gpg.d/proxmox-ve-release-5.x.gpg
apt update && apt dist-upgrade

apt install proxmox-ve postfix open-iscsi
apt remove os-prober

# Déployer les VMs avec Ansible

https://docs.ansible.com/ansible/latest/modules/scaleway_image_facts_module.html#scaleway-image-facts-module

## Prérequis

Avoir une clé SSH, avoir la clé publique à la racine du projet, et l'appeler admin.pub

## Générer les machines

Créer un playbook pour

* récupérer un ID d'image compatible debian Stretch
* créer autant de machines que nécessaire

```YAML
cat scaleway/create_proxmox.yaml
---
- hosts: localhost
  connection: local
  tasks:
  - name: "Add SSH key"
    scaleway_sshkey:
      ssh_pub_key: "{{lookup('file', 'admin.pub') }}"
      state: "present"
  - name: Gather Scaleway images facts
    scaleway_image_facts:
      region: par1
  - name: "Create my servers"
    scaleway_compute:
      name: proxmox{{item}}
      state: running
      image: "37832f54-c18f-4338-a552-113e4302a236"
      organization: 1bb690d9-1039-4316-abec-91903df637af
      region: par1
      commercial_type: START1-S
      wait: true
      tags: proxmoxve
    loop: "{{ range(0, 4 + 1, 2)|list }}"
```

# Installer proxmox avec Ansible

## Mise en place de l'inventaire

cat scaleway/inventory_proxmox.yml
plugin: scaleway
regions:
  - par1
tags:
  - proxmoxve

```JSON
ansible-inventory --list -i scaleway/inventory_proxmox.yml
{
    "_meta": {
        "hostvars": {
            "51.15.217.222": {
                "arch": "x86_64", 
                "commercial_type": "START1-S", 
                "hostname": "proxmox01", 
                "id": "95f9bc84-20d5-40ea-85bf-adba7159bf73", 
                "organization": "1bb690d9-1039-4316-abec-91903df637af", 
                "private_ipv4": "10.16.166.25", 
                "public_ipv4": "51.15.217.222", 
                "state": "running", 
                "tags": [
                    "proxmoxve"
                ]
            }
        }
    }, 
    "all": {
        "children": [
            "par1", 
            "proxmoxve", 
            "ungrouped"
        ]
    }, 
    "par1": {
        "hosts": [
            "51.15.217.222"
        ]
    }, 
    "proxmoxve": {
        "hosts": [
            "51.15.217.222"
        ]
    }, 
    "ungrouped": {}
}
```



http://www.oznetnerd.com/jinja2-selectattr-filter/