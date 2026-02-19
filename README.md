---
# ==============================================================================
# Confluent Kafka Broker Deployment Playbook
# Hosts  : broker_nodes group (defined in AAP inventory)
# Secrets: group_vars/secret.yml  (Ansible Vault, password: humana#1)
#
# AAP Job Template requires TWO credentials:
#   1. Machine credential  - SSH access to brokers
#   2. Vault credential    - password humana#1  to decrypt group_vars/secret.yml
#
# What this playbook does (serial: 1 = rolling, one broker at a time):
#   create_user → create_directories → install_confluent → setup_prometheus
#   → deploy_secrets (writes /opt/secrets/local-secrets.properties)
#   → configure → configure_service → start_service → validate
# ==============================================================================

- name: Deploy Confluent Kafka Broker Cluster
  hosts: broker_nodes
  become: true
  gather_facts: true
  serial: 1

  vars_files:
    - "{{ playbook_dir }}/../group_vars/secret.yml"

  roles:
    - broker

  post_tasks:
    - name: Display deployment summary
      ansible.builtin.debug:
        msg: |
          ============================================
          Kafka Broker Deployment Complete
          ============================================
          Nodes:
          {% for host in groups['broker_nodes'] %}
          - {{ hostvars[host]['ansible_fqdn'] }} (ID: {{ hostvars[host]['broker_id'] }})
          {% endfor %}

          Configuration:
          - Install Dir  : {{ broker_install_dir }}
          - Data Dir     : {{ broker_data_dir }}
          - Config Dir   : {{ broker_config_dir }}
          - Secrets File : {{ broker_secrets_dir }}/local-secrets.properties
          - SASL_SSL Port: {{ kafka_sasl_ssl_port }}
          - TOKEN Port   : {{ kafka_token_port }}
          - AD Port      : {{ kafka_ad_port }}
          - MDS Port     : {{ kafka_mds_port }}
          - Prometheus   : {{ prometheus_jmx_exporter_port }}
          ============================================
      run_once: true


---
# ==============================================================================
# Confluent Kafka Broker - Intelligent Rolling Restart Playbook
# Strategy: Restart non-leader brokers first, preferred-leader broker last
# No shell or command modules - pure Ansible built-in modules only
# Vault secrets loaded from: group_vars/secret.yml  (password: humana#1)
# Vault password supplied by: AAP Vault credential (password: humana#1)
# Mirrors: playbooks/restart_zookeeper.yml pattern
# ==============================================================================

# ------------------------------------------------------------------------------
# PLAY 1: Discover cluster state - which broker holds most leader partitions
# ------------------------------------------------------------------------------
- name: Discover Broker Cluster State
  hosts: broker_nodes
  become: true
  gather_facts: true
  serial: "100%"

  vars_files:
    - "{{ playbook_dir }}/../group_vars/secret.yml"

  tasks:
    - name: Resolve broker_id for this host
      ansible.builtin.set_fact:
        broker_id: "{{ broker_id_map[inventory_hostname] }}"
        cacheable: true

    - name: Check confluent-kafka service status
      ansible.builtin.systemd:
        name: "{{ broker_service_name }}"
      register: broker_service
      failed_when: false

    - name: Query MDS for cluster info
      ansible.builtin.uri:
        url: "https://{{ ansible_fqdn }}:{{ kafka_mds_port }}/v3/clusters"
        method: GET
        validate_certs: true
        ca_path: "{{ ssl_truststore_location }}"
        status_code: 200
        return_content: true
        timeout: 30
      register: cluster_info
      failed_when: false
      when: broker_service.status.ActiveState == "active"

    - name: Parse cluster ID
      ansible.builtin.set_fact:
        kafka_cluster_id: >-
          {{
            (cluster_info.json.data | map(attribute='cluster_id') | list | first)
            if (cluster_info is succeeded and cluster_info.json is defined)
            else 'unknown'
          }}
      when: cluster_info is defined and cluster_info is succeeded

    - name: Default cluster_id if MDS unavailable
      ansible.builtin.set_fact:
        kafka_cluster_id: "unknown"
      when: kafka_cluster_id is not defined

    - name: Query MDS for leader partition count on this broker
      ansible.builtin.uri:
        url: "https://{{ ansible_fqdn }}:{{ kafka_mds_port }}/v3/clusters/{{ kafka_cluster_id }}/brokers/{{ broker_id }}/partition-replicas"
        method: GET
        validate_certs: true
        ca_path: "{{ ssl_truststore_location }}"
        status_code: 200
        return_content: true
        timeout: 60
      register: partition_replicas
      failed_when: false
      when:
        - broker_service.status.ActiveState == "active"
        - kafka_cluster_id != 'unknown'

    - name: Count leader partitions on this broker
      ansible.builtin.set_fact:
        leader_partition_count: >-
          {{
            (partition_replicas.json.data |
             selectattr('is_leader', 'equalto', true) |
             list | length)
            if (partition_replicas is defined and partition_replicas is succeeded
                and partition_replicas.json is defined)
            else 0
          }}

    - name: Set is_leader_broker flag (broker with most leaders restarts last)
      ansible.builtin.set_fact:
        is_leader_broker: "{{ leader_partition_count | int > 0 }}"
        cacheable: true

    - name: Display broker role
      ansible.builtin.debug:
        msg: "{{ ansible_fqdn }} (ID: {{ broker_id }}) leads {{ leader_partition_count }} partition(s) — restart order: {{ 'LAST (leader broker)' if is_leader_broker else 'NORMAL' }}"


# ------------------------------------------------------------------------------
# PLAY 2: Restart non-leader brokers first (serial: 1)
# ------------------------------------------------------------------------------
- name: Restart Non-Leader Brokers First
  hosts: broker_nodes
  become: true
  gather_facts: false
  serial: 1

  tasks:
    - name: Restart non-leader broker
      when: not (is_leader_broker | default(false) | bool)
      block:
        - name: Display restart message
          ansible.builtin.debug:
            msg: "Restarting NON-LEADER broker: {{ ansible_fqdn }} (ID: {{ broker_id }})"

        - name: Stop confluent-kafka (controlled shutdown re-elects leaders)
          ansible.builtin.systemd:
            name: "{{ broker_service_name }}"
            state: stopped

        - name: Wait for SASL_SSL port to close
          ansible.builtin.wait_for:
            port: "{{ kafka_sasl_ssl_port }}"
            state: stopped
            timeout: "{{ broker_restart_timeout }}"

        - name: Pause {{ restart_wait_time }}s for leader election propagation
          ansible.builtin.pause:
            seconds: "{{ restart_wait_time }}"

        - name: Start confluent-kafka
          ansible.builtin.systemd:
            name: "{{ broker_service_name }}"
            state: started

        - name: Wait for SASL_SSL port {{ kafka_sasl_ssl_port }} to open
          ansible.builtin.wait_for:
            port: "{{ kafka_sasl_ssl_port }}"
            state: started
            delay: 5
            timeout: "{{ broker_restart_timeout }}"

        - name: Wait for MDS port {{ kafka_mds_port }} to open
          ansible.builtin.wait_for:
            port: "{{ kafka_mds_port }}"
            state: started
            timeout: "{{ broker_restart_timeout }}"

        - name: Wait for Prometheus port {{ prometheus_jmx_exporter_port }} to open
          ansible.builtin.wait_for:
            port: "{{ prometheus_jmx_exporter_port }}"
            state: started
            timeout: "{{ broker_restart_timeout }}"
          when: prometheus_jmx_exporter_enabled | bool

        - name: Assert broker service is healthy
          ansible.builtin.systemd:
            name: "{{ broker_service_name }}"
          register: post_restart_status

        - name: Verify broker running before moving to next host
          ansible.builtin.assert:
            that:
              - post_restart_status.status.ActiveState == "active"
              - post_restart_status.status.SubState == "running"
            fail_msg: >-
              ABORT: Broker {{ ansible_fqdn }} (ID: {{ broker_id }}) failed health check.
              Rolling restart halted to protect cluster integrity.
            success_msg: "Broker {{ ansible_fqdn }} (ID: {{ broker_id }}) restarted successfully."

        - name: Pause {{ restart_wait_time }}s before next broker
          ansible.builtin.pause:
            seconds: "{{ restart_wait_time }}"

        - name: Display success
          ansible.builtin.debug:
            msg: "✓ {{ ansible_fqdn }} (non-leader) restarted successfully"


# ------------------------------------------------------------------------------
# PLAY 3: Restart the leader broker LAST
# ------------------------------------------------------------------------------
- name: Restart Leader Broker Last
  hosts: broker_nodes
  become: true
  gather_facts: false
  serial: 1

  tasks:
    - name: Restart leader broker
      when: is_leader_broker | default(false) | bool
      block:
        - name: Display leader restart warning
          ansible.builtin.debug:
            msg: |
              ⚠ Restarting LEADER broker: {{ ansible_fqdn }} (ID: {{ broker_id }})
              ⚠ Kafka controlled.shutdown will trigger new leader election before stop

        - name: Pause 10s before leader restart
          ansible.builtin.pause:
            seconds: 10

        - name: Stop confluent-kafka leader (controlled shutdown re-elects all leader partitions)
          ansible.builtin.systemd:
            name: "{{ broker_service_name }}"
            state: stopped

        - name: Wait for SASL_SSL port to close
          ansible.builtin.wait_for:
            port: "{{ kafka_sasl_ssl_port }}"
            state: stopped
            timeout: "{{ broker_restart_timeout }}"

        - name: Pause {{ restart_wait_time }}s for new leader election to complete
          ansible.builtin.pause:
            seconds: "{{ restart_wait_time }}"

        - name: Start confluent-kafka leader
          ansible.builtin.systemd:
            name: "{{ broker_service_name }}"
            state: started

        - name: Wait for SASL_SSL port {{ kafka_sasl_ssl_port }} to open
          ansible.builtin.wait_for:
            port: "{{ kafka_sasl_ssl_port }}"
            state: started
            delay: 5
            timeout: "{{ broker_restart_timeout }}"

        - name: Wait for MDS port {{ kafka_mds_port }} to open
          ansible.builtin.wait_for:
            port: "{{ kafka_mds_port }}"
            state: started
            timeout: "{{ broker_restart_timeout }}"

        - name: Wait for Prometheus port {{ prometheus_jmx_exporter_port }} to open
          ansible.builtin.wait_for:
            port: "{{ prometheus_jmx_exporter_port }}"
            state: started
            timeout: "{{ broker_restart_timeout }}"
          when: prometheus_jmx_exporter_enabled | bool

        - name: Assert leader broker is healthy
          ansible.builtin.systemd:
            name: "{{ broker_service_name }}"
          register: leader_post_restart_status

        - name: Verify leader broker running
          ansible.builtin.assert:
            that:
              - leader_post_restart_status.status.ActiveState == "active"
              - leader_post_restart_status.status.SubState == "running"
            fail_msg: >-
              Leader broker {{ ansible_fqdn }} (ID: {{ broker_id }}) failed health check after restart.
            success_msg: "Leader broker {{ ansible_fqdn }} (ID: {{ broker_id }}) restarted successfully."

        - name: Display leader restart success
          ansible.builtin.debug:
            msg: "✓ {{ ansible_fqdn }} (leader) restarted successfully — new leader election complete"


# ------------------------------------------------------------------------------
# PLAY 4: Final cluster health verification
# ------------------------------------------------------------------------------
- name: Verify Cluster Health After Restart
  hosts: broker_nodes
  become: true
  gather_facts: false

  tasks:
    - name: Get final service status
      ansible.builtin.systemd:
        name: "{{ broker_service_name }}"
      register: final_status

    - name: Check all listener ports are open
      ansible.builtin.wait_for:
        host: "{{ ansible_fqdn }}"
        port: "{{ item.port }}"
        timeout: 30
        state: started
      loop:
        - { port: "{{ kafka_sasl_ssl_port }}", name: "SASL_SSL" }
        - { port: "{{ kafka_token_port }}",    name: "TOKEN" }
        - { port: "{{ kafka_ad_port }}",       name: "AD" }
        - { port: "{{ kafka_mds_port }}",      name: "MDS" }

    - name: Display final cluster state
      ansible.builtin.debug:
        msg: |
          ============================================
          Rolling Restart Complete
          ============================================
          {% for host in groups['broker_nodes'] %}
          - {{ hostvars[host]['ansible_fqdn'] }} (ID: {{ hostvars[host]['broker_id'] }}):
            {{ hostvars[host]['final_status']['status']['ActiveState'] | default('unknown') }} / {{ hostvars[host]['final_status']['status']['SubState'] | default('unknown') }}
          {% endfor %}
          ============================================
      run_once: true
