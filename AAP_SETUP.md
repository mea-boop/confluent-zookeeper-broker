# Ansible Automation Platform Setup Guide

## Complete Setup for Confluent Kafka Broker Cluster
## Environment: prd-eastus2-az3

---

### Step 1: Create Inventory in AAP

1. Log into AAP: `https://your-aap-server`
2. Navigate to **Resources → Inventories**
3. Click **Add** → **Add inventory**

```
Name        : Kafka_Broker_prd_eastus2_az3
Description : Confluent Kafka Broker cluster - prd-eastus2-az3
Organization: Your Organization
```

---

### Step 2: Create Group

1. In the inventory, click **Groups** tab
2. Click **Add**

```
Name       : broker_nodes
Description: Confluent Kafka Broker cluster nodes
```

---

### Step 3: Add Hosts

Add each broker host. The hostname MUST match the key in `broker_id_map` in `group_vars/broker_nodes.yml`.

**Host 1:**
```
Name       : broker1-east-az3.prd.eeh.humana.com
Description: Kafka Broker Node 1 (ID: 35)

Variables:
---
ansible_host: <IP_ADDRESS>
ansible_user: <SSH_USER>
ansible_become: yes
broker_rack: eastus2-2
```

**Host 2:**
```
Name       : broker2-east-az3.prd.eeh.humana.com
Description: Kafka Broker Node 2 (ID: 37)

Variables:
---
ansible_host: <IP_ADDRESS>
ansible_user: <SSH_USER>
ansible_become: yes
broker_rack: eastus2-2
```

**Host 3:**
```
Name       : broker3-east-az3.prd.eeh.humana.com
Description: Kafka Broker Node 3 (ID: 39)

Variables:
---
ansible_host: <IP_ADDRESS>
ansible_user: <SSH_USER>
ansible_become: yes
broker_rack: eastus2-2
```

---

### Step 4: Associate Hosts with Group

1. In the `broker_nodes` group, click **Hosts** tab
2. Click **Add existing host**
3. Select all three broker hosts
4. Click **Save**

---

### Step 5: Create AAP Custom Credential Type

Navigate to **Administration → Credential Types → Add**

**Name:** `Confluent Broker Secrets`

**Input Configuration (YAML):**
```yaml
fields:
  - id: confluent_license
    type: string
    label: Confluent License Key
    secret: true
  - id: kafka_admin_password
    type: string
    label: Kafka Admin SCRAM Password
    secret: true
  - id: ldap_bind_password
    type: string
    label: LDAP Bind Password
    secret: true
  - id: ssl_truststore_password
    type: string
    label: SSL Truststore Password
    secret: true
  - id: ssl_keystore_password
    type: string
    label: SSL Keystore Password
    secret: true
  - id: ssl_key_password
    type: string
    label: SSL Key Password
    secret: true
required:
  - confluent_license
  - kafka_admin_password
  - ldap_bind_password
  - ssl_truststore_password
  - ssl_keystore_password
  - ssl_key_password
```

**Injector Configuration (YAML):**
```yaml
extra_vars:
  confluent_license: '{{ confluent_license }}'
  kafka_admin_password: '{{ kafka_admin_password }}'
  ldap_bind_password: '{{ ldap_bind_password }}'
  ssl_truststore_password: '{{ ssl_truststore_password }}'
  ssl_keystore_password: '{{ ssl_keystore_password }}'
  ssl_key_password: '{{ ssl_key_password }}'
```

---

### Step 6: Create Credential Instance

Navigate to **Resources → Credentials → Add**

```
Name            : Confluent_Broker_Secrets_prd
Credential Type : Confluent Broker Secrets
Organization    : Your Organization
```

Fill in all secret field values. They are encrypted and stored in AAP's vault. **Never stored in Git.**

---

### Step 7: Create SSH Machine Credential

Navigate to **Resources → Credentials → Add**

```
Name                         : Kafka_Broker_SSH_prd
Credential Type              : Machine
Username                     : <SSH_USER>
SSH Private Key              : (paste private key)
Privilege Escalation Method  : sudo
Privilege Escalation Username: root
```

---

### Step 8: Upload to Azure DevOps

```bash
cd broker-confluent-ansible
git init
git add .
git commit -m "Initial: Confluent Kafka Broker Ansible"
git remote add origin https://dev.azure.com/<org>/<project>/_git/broker-ansible
git push -u origin main
```

---

### Step 9: Create Project in AAP

Navigate to **Resources → Projects → Add**

```
Name        : Confluent_Kafka_Broker_Deployment
Description : Kafka Broker deployment automation
Organization: Your Organization

SCM Type    : Git
SCM URL     : https://dev.azure.com/<org>/<project>/_git/broker-ansible
SCM Branch  : main

SCM Update Options:
☑ Clean
☑ Update Revision on Launch
```

Click **Save** then **Sync**.

---

### Step 10: Create Job Templates

**Template 1 - Deploy Brokers:**

```
Name       : Deploy_Kafka_Broker_prd
Job Type   : Run
Inventory  : Kafka_Broker_prd_eastus2_az3
Project    : Confluent_Kafka_Broker_Deployment
Playbook   : playbooks/deploy_broker.yml
Credentials: Kafka_Broker_SSH_prd
             Confluent_Broker_Secrets_prd
Verbosity  : 1 (Verbose)

Options:
☑ Enable Privilege Escalation
☑ Enable Fact Storage
```

**Template 2 - Rolling Restart (Leader Last):**

```
Name       : Restart_Kafka_Broker_prd
Job Type   : Run
Inventory  : Kafka_Broker_prd_eastus2_az3
Project    : Confluent_Kafka_Broker_Deployment
Playbook   : playbooks/restart_broker.yml
Credentials: Kafka_Broker_SSH_prd
             Confluent_Broker_Secrets_prd
Verbosity  : 1 (Verbose)

Options:
☑ Enable Privilege Escalation
```

---

### Step 11: Run Deployment

1. Go to **Resources → Templates**
2. Find `Deploy_Kafka_Broker_prd`
3. Click **Launch**
4. Monitor progress (15-20 minutes for full cluster)

---

### Step 12: Verify Deployment

After completion, check each broker:

**Service status:**
```bash
ssh broker1-east-az3.prd.eeh.humana.com
sudo systemctl status confluent-kafka
```

**Listener ports:**
```bash
ss -tlnp | grep -E '9093|9092|9094|8090|7071'
```

**Prometheus metrics:**
```bash
curl http://broker1-east-az3.prd.eeh.humana.com:7071/metrics | grep kafka_
```

**MDS API:**
```bash
curl -k https://broker1-east-az3.prd.eeh.humana.com:8090/v1/metadata/id
```

---

## Secret Flow Reference

```
AAP Custom Credential (encrypted in AAP vault)
    ↓ Credential Injector → extra_vars at job runtime
Ansible plays in memory (never logged - no_log: true on secrets task)
    ↓ ansible.builtin.template renders local-secrets.properties.j2
/opt/secrets/local-secrets.properties (mode 0600, kafka:kafka)
    ↓ Confluent SecurePass reads at JVM startup
server.properties ${securepass:...} placeholders → JVM memory only
```

## Security Checklist

| Item                                        | Location                | Status |
|---------------------------------------------|-------------------------|--------|
| Confluent License Key                       | AAP Custom Credential   | ✅     |
| Kafka Admin SCRAM Password                  | AAP Custom Credential   | ✅     |
| LDAP Bind Password                          | AAP Custom Credential   | ✅     |
| SSL Truststore / Keystore / Key Passwords   | AAP Custom Credential   | ✅     |
| SSH Private Key                             | AAP Machine Credential  | ✅     |
| local-secrets.properties (plaintext values) | Broker disk / mode 0600 | ✅     |
| server.properties (SecurePass refs only)    | Broker disk / mode 0640 | ✅     |
| **Nothing sensitive in Azure DevOps Git**   | Azure DevOps            | ✅     |
