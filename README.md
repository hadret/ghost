# Simple Ansible helper playbook

This playbook is used as a helper for preparing Docker Swarm-based infra for the
rollout.

Assuming Ansible is already installed and you are inside this repo:

```bash
ansible-galaxy install -r roles/requirements.yml # to fetch external roles
ansible all -m ping # (optional) to check the connectivity
ansible all -m setup # (optional) to gather facts
ansible-playbook site.yml
```

Further instructions regarding Swarm setup were moved here:
[swarm/README.md](swarm/README.md).
