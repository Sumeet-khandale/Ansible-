# Ansible
Ansible step-by-step actions

# Ansible setup & playbook run — Steps I performed

This document records the step-by-step actions I ran on an Ubuntu EC2 instance and a target host while installing Ansible, configuring SSH keys, creating an inventory, running ad-hoc commands, fixing a playbook syntax error, and finally running the playbook that installed and started Nginx.

---

## 1. SSH into the controller (local Windows → EC2)

```bash
# From Windows (PowerShell / CMD)
ssh -i c:/ssk/devops.pem ubuntu@3.110.164.124
```

* Accepted the host fingerprint and permanently added `3.110.164.124` to `~/.ssh/known_hosts`.
* Logged in to an Ubuntu 24.04.3 LTS instance which became the Ansible controller.

## 2. Update package lists on the controller

```bash
sudo apt update
```

* `apt` fetched package lists. Output showed `14 packages can be upgraded`.

## 3. Install Ansible on the controller

```bash
# attempted wrong command first
sudo install ansible        # -> error: missing destination file operand

# correct command
sudo apt install ansible
```

* Confirmed installation by pressing `y` when prompted.
* Packages installed included `ansible` and several Python dependencies.

## 4. Verify Ansible installation

```bash
ansible --version
```

* Output:

  * `ansible [core 2.16.3]`
  * Python 3.12.3
  * `executable location = /usr/bin/ansible`

## 5. Attempt SSH to target host (private IP) and fix public-key auth

```bash
# from controller
ssh 172.31.3.71
# Result: Permission denied (publickey)

# generate a new keypair on controller
ssh-keygen
# saved to /home/ubuntu/.ssh/id_ed25519 and id_ed25519.pub

# view the public key
cat /home/ubuntu/.ssh/id_ed25519.pub
```

* After adding the new public key to the target host's `~/.ssh/authorized_keys` (implicit by subsequent success), SSH to `172.31.3.71` worked and logged into the target host.

## 6. Create a simple inventory file

```bash
# file: inventary
172.31.3.71
```

* Inventory file created in the controller home directory.

## 7. Test an ad-hoc Ansible command

```bash
# run an ad-hoc shell module to create a file on the target
ansible -i inventary all -m shell -a "touch devopsansible"
```

* Output: `172.31.3.71 | CHANGED | rc=0 >>` (file created successfully)

## 8. Create the first playbook `first-playbook.yml`

You created a playbook to install and start Nginx. Initial attempt failed due to YAML/syntax problems.

**First error:**

```
ERROR! We were unable to read either as JSON nor YAML
... Syntax Error while loading YAML. did not find expected '-' indicator
```

* You re-opened and fixed the file with proper YAML structure.

**Second error when running the playbook:**

```
fatal: [172.31.3.71]: FAILED! => {"msg": "the field 'become' has an invalid value (root), and could not be converted to an bool..."}
```

* This happened because `become` was set incorrectly to `root` instead of using `become: yes` and `become_user: root` (or leaving `become: yes` alone and the default become user `root` will be used).

## 9. Final working `first-playbook.yml` (what ended up in the file)

```yaml
---
- name: Install and start Nginx
  hosts: all
  become: yes

  tasks:
    - name: Install Nginx
      apt:
        name: nginx
        state: present

    - name: Start Nginx
      service:
        name: nginx
        state: started
```

* Running the playbook succeeded:

  * `Gathering Facts` — ok
  * `Install Nginx` — changed
  * `Start Nginx` — ok

Play recap: `ok=3 changed=1 unreachable=0 failed=0`

## 10. Files present in controller home directory

```
first-playbook.yml
inventary
```

## Notes / Troubleshooting summary (useful for README)

* If you see `Permission denied (publickey)` when SSHing from controller to target, ensure the controller public key is present in the target's `~/.ssh/authorized_keys`.
* `ssh-keygen` generates a keypair and saves the public key in `~/.ssh/id_ed25519.pub` — copy that into the target host's `authorized_keys`.
* When Ansible reports `ERROR! No argument passed to command module` or similar, remember ad-hoc usage requires `-a` (module args) and `-m` (module) when you want to perform a command.
* Playbook YAML must be properly indented and start with `---`. The `become` field expects a boolean (`yes`/`no`), not a username. Use `become_user: root` if you want to specify the escalation user.

---

*End of recorded steps.*

## 11. Second EC2 session — actions you ran (summary)

This section records the second EC2 session (controller connecting to a different public IP) and the specific commands you ran there.

* SSH into new instance from Windows:

  ```bash
  ssh -i c:/ssk/devops.pem ubuntu@13.233.144.247
  ```

  * Accepted fingerprint and added to `~/.ssh/known_hosts`.
  * Logged into Ubuntu 24.04.3 LTS (this instance shows private IP `172.31.3.71`).

* Generated an ED25519 keypair on this instance:

  ```bash
  ssh-keygen
  # created /home/ubuntu/.ssh/id_ed25519 and id_ed25519.pub
  ```

* Verified `~/.ssh` contents and `authorized_keys`:

  ```bash
  ls -a
  ls ~/.ssh/
  vim ~/.ssh/authorized_keys
  ```

* Confirmed presence of `devopsansible` file (created earlier via Ansible) and that Nginx is running:

  ```bash
  ls -ltr
  sudo systemctl status nginx
  ```

  * `nginx.service` is active (running).

## 12. Side-by-side comparison (first EC2 vs second EC2)

* **Public IPs used to SSH from Windows**

  * First session: `3.110.164.124` (controller where you installed Ansible).
  * Second session: `13.233.144.247` (another EC2 you logged into; its private IP is `172.31.3.71`).

* **Roles of the machines during your workflow**

  * First EC2 (`3.110.164.124`) acted as the *Ansible controller* where you installed Ansible, created an inventory file (`inventary`), generated SSH keys, and ran the playbook.
  * Second EC2 (`13.233.144.247` / `172.31.3.71`) appears to be the *target host* (the machine that received `devopsansible`, had Nginx installed and running). You also SSH'd into it directly and generated an SSH key there.

* **Key operations that happened on each**

  * *Controller (3.110.164.124)*

    * `sudo apt update`
    * `sudo apt install ansible`
    * `ansible --version`
    * `ssh-keygen` (created keys used to allow controller → target access)
    * created `inventary` listing `172.31.3.71`
    * ad-hoc command: `ansible -i inventary all -m shell -a "touch devopsansible"`
    * created `first-playbook.yml` and ran `ansible-playbook` to install/start Nginx
  * *Target (13.233.144.247 / 172.31.3.71)*

    * Received the controller public key in `~/.ssh/authorized_keys` (so controller SSH worked)
    * `devopsansible` file exists
    * `nginx` installed and running (`systemctl status nginx` shows active)
    * You also ran `ssh-keygen` here (created another keypair under this user's home), and inspected `~/.ssh`.

* **Differences & observations**

  * The controller generated a keypair and used its public key to connect to the target. On the target you also generated keys — generating keys on the target is fine but not required for controller authentication; the important piece is the controller's *public* key being in the target `authorized_keys`.
  * Both sessions show Ubuntu 24.04.3 LTS and a similar system state messages.
  * Nginx is installed and active only on the target host (`172.31.3.71`). The controller only ran the playbook to install it remotely.

## 13. Consolidated step-by-step (ready to paste into README.md)

1. **SSH from local Windows to controller EC2**

   ```bash
   ssh -i c:/ssk/devops.pem ubuntu@3.110.164.124
   ```

   * Accept host fingerprint when prompted.

2. **Update packages on controller**

   ```bash
   sudo apt update
   ```

3. **Install Ansible on controller**

   ```bash
   sudo apt install ansible
   ansible --version
   ```

4. **Generate SSH keypair on controller (if not already present)**

   ```bash
   ssh-keygen
   # default path: /home/ubuntu/.ssh/id_ed25519
   cat /home/ubuntu/.ssh/id_ed25519.pub
   ```

   * Copy the public key to target `~/.ssh/authorized_keys` (you can use ssh-copy-id or manually append).

5. **Create inventory file on controller**

   ```text
   # file: inventary
   172.31.3.71
   ```

6. **Test connectivity with an ad-hoc command**

   ```bash
   ansible -i inventary all -m shell -a "touch devopsansible"
   ```

7. **Write playbook to install and start Nginx**

   ```yaml
   ---
   - name: Install and start Nginx
     hosts: all
     become: yes

     tasks:
       - name: Install Nginx
         apt:
           name: nginx
           state: present

       - name: Start Nginx
         service:
           name: nginx
           state: started
   ```

8. **Run the playbook**

   ```bash
   ansible-playbook -i inventary first-playbook.yml
   ```

9. **Verify on target (optional from controller)**

   ```bash
   # from controller or by SSH-ing directly to target
   ssh ubuntu@172.31.3.71
   sudo systemctl status nginx
   ```

---
