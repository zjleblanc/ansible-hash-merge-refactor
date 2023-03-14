# ansible-hash-merge-refactor

This repository contains an example project that provides methods for refactoring existing development relying on the DEFAULT_HASH_BEHAVIOR=merge setting (deprecated as of 2.13)

## playbook.yml

Contains three tasks which show how you can expect hashes to be combined by default and how to achieve the merge behavior with different priorities. The example shown is based on nfs mounts defined at the group level and in some cases defined per host.

---

## 1st task - default behavior (replace)

By default, any key that exists in host vars will replace a key that exists in corresponding group vars.

```yaml
...
- name: NFS mounts (using replace)
    ansible.builtin.set_fact:
    all_nfs_mounts: "{{ nfs_mounts | default([]) }}"
```

```yaml
# prod group vars
nfs_mounts:
  /yosemite:
    opts: rw,sync,hard
  /elcapitan: 
    opts: bind
```

```yaml
# hostvars for prod_vm_1 in prod group
nfs_mounts:
  /yosemite:
    opts: rw,sync
  /rainier: 
    opts: rw
```

```json
// all_nfs_mounts will be (because the key "nfs_mounts" is replaced)
{
    "/yosemite": {
        "opts": "rw,sync"
    },
    "/rainier": {
        "opts": "rw"
    }
}
```

---

## 2nd task - custom merge (host configs take priority)

In order to implement the merge behavior previously available with DEFAULT_HASH_BEHAVIOR=merge, you will have to use a standardized prefix for the for your group vars. I recommend prefixing vars with the group name to simplify accessing them in your playbook later.

```yaml
...
# all_nfs_mounts has been set to nfs_mounts (matching the hostvars definition)
# the loop will merge the existing nfs_mounts into the <group_name>_nfs_mounts key from each groupvars
- name: NFS mounts (merge groups) - Host mount configs take priority
  loop: "{{ group_names }}"
  ansible.builtin.set_fact:
  # Curious about the expression? See Note 1.1
  all_nfs_mounts: "{{ hostvars[inventory_hostname][item + '_nfs_mounts'] | combine(all_nfs_mounts) }}"
```

```yaml
# prod group vars
prod_nfs_mounts:
  /yosemite:
    opts: rw,sync,hard
  /elcapitan: 
    opts: bind
```

```yaml
# hostvars for prod_vm_1 in prod group
nfs_mounts:
  /yosemite:
    opts: rw,sync
  /rainier: 
    opts: rw
```

```json
// all_nfs_mounts will be (because the key "nfs_mounts" is merged into "prod_nfs_mounts")
{
    "/yosemite": {
        "opts": "rw,sync"
    },
    "elcapitan": {
        "opts": "bind",
    },
    "/rainier": {
        "opts": "rw"
    },
}
```

---

## 3rd task - custom merge (groups configs take priority)

Consider the 2nd task explanation, but you want group defined nfs_mounts to take priority. The only change required to the second task is which order you pass dicts to the combine filter. In this case, you want to merge the <group_name>_nfs_mounts into the existing nfs_mounts.

```yaml
...
# all_nfs_mounts has been set to nfs_mounts (matching the hostvars definition)
# the loop will merge the relevant <group_name>_nfs_mounts key from each groupvars into the existing nfs_mounts
- name: NFS mounts (merge groups) - Host mount configs take priority
  loop: "{{ group_names }}"
  ansible.builtin.set_fact:
  all_nfs_mounts: "{{ hostvars[inventory_hostname][item + '_nfs_mounts'] | combine(all_nfs_mounts) }}"
```

```json
// all_nfs_mounts will be (because the key "prod_nfs_mounts" is merged into "nfs_mounts")
// note the key difference: /yosemite.opts values comes from the group vars entry
{
    "/yosemite": {
        "opts": "rw,sync,hard"
    },
    "elcapitan": {
        "opts": "bind",
    },
    "/rainier": {
        "opts": "rw"
    },
}
```

## Notes

### 1.1 Accessing group vars during execution

During playbook execution, there is no concept of groupvars. Instead, hosts inherit vars from groups they are a member of and these vars are available in the same way that a hostvar can be accessed. To accomplish a merge using prefixed group vars, we need to reach into the hostvars dict using the relevant group key. A list of group names can be accessed via the `group_names` var. For example:
```yaml
...
- name: Example accessing group vars
  loop: "{{ group_names }}"  
  set_fact:
    all_nfs_mounts: "{{ hostvars[inventory_hostname][item + '_nfs_mounts'] | combine(all_nfs_mounts) }}"
```
Here we are accessing the hostvars for the current host and selecting the `<group_name>_nfs_mounts` key to combine with all_nfs_hosts. To make this work, your nfs_hosts key in the prod group would be `prod_nfs_hosts`. 