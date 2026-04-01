# Ansible Reference

Concise guide for Ansible ad-hoc commands, playbook execution, and vault management.

## Ad-Hoc Commands
| Task | Command |
| :--- | :--- |
| **Ping all hosts** | `ansible all -m ping` |
| **Check uptime** | `ansible all -a "uptime"` |
| **Install package** | `ansible web -m apt -a "name=vim state=present" --become` |
| **Restart service** | `ansible db -m service -a "name=nginx state=restarted" --become` |
| **Copy file** | `ansible all -m copy -a "src=/etc/hosts dest=/tmp/hosts"` |

## Playbooks
| Action | Command |
| :--- | :--- |
| **Run playbook** | `ansible-playbook site.yml` |
| **Limit to host** | `ansible-playbook site.yml -l node01` |
| **Dry run (Check)** | `ansible-playbook site.yml --check` |
| **List tasks** | `ansible-playbook site.yml --list-tasks` |
| **Start at task** | `ansible-playbook site.yml --start-at-task="Install Nginx"` |

## Ansible Vault
| Action | Command |
| :--- | :--- |
| **Create secret** | `ansible-vault create secret.yml` |
| **Edit secret** | `ansible-vault edit secret.yml` |
| **Encrypt existing** | `ansible-vault encrypt vars.yml` |
| **Run with vault** | `ansible-playbook site.yml --ask-vault-pass` |

## Ansible Galaxy
| Action | Command |
| :--- | :--- |
| **Install role** | `ansible-galaxy install geerlingguy.docker` |
| **Install reqs** | `ansible-galaxy install -r requirements.yml` |
| **Init new role** | `ansible-galaxy init my_new_role` |

## Debugging & Inventory
| Task | Command |
| :--- | :--- |
| **List inventory** | `ansible-inventory --list` |
| **Graph inventory** | `ansible-inventory --graph` |
| **Debug variables** | `ansible all -m debug -a "var=hostvars[inventory_hostname]"` |
