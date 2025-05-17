### Ansible without/without Temporal

### Simple workflow

[![GitLab](https://e.radikal.host/2025/05/17/Screenshot-2025-05-17-at-07.41.15.png)](https://radikal.host/i/Ir0JQe)

[![Temporal](https://e.radikal.host/2025/05/17/Screenshot-2025-05-17-at-07.52.56.png)](https://radikal.host/i/Ir0Q5K)

```yaml
ansible_workflows:
  stage: deploy
  tags:
    - shell
  script:
    - python3 ansible.py --playbook apt_update.yml
```

#### 1. Long-Running Workflows with Checkpointing

Problem: Ansible playbooks fail on long-running tasks (e.g., cloud provisioning, multi-hour deployments).
Solution:
Use Temporal workflows to automatically resume from failures.
Checkpoint progress (e.g., "50% of VMs deployed") even if the process crashes.
```python
@workflow.defn
class AnsibleWorkflow:
    async def run(self, playbook: str):
        while not workflow.is_replaying():
            result = await workflow.execute_activity(
                run_ansible_playbook,
                args=[playbook],
                start_to_close_timeout=timedelta(hours=6),
                heartbeat_timeout=timedelta(minutes=5),
            )
            if result.failed:
                await asyncio.sleep(300)  # Retry after 5 minutes
```

#### 2. Dynamic Parallel Execution

Problem: Ansible’s async is limited—hard to manage 1000s of nodes dynamically.
Solution:
Fan-out/fan-in workflows (e.g., deploy 500 VMs in parallel, then aggregate results).
```python
async def deploy_vms():
    vm_list = await get_vm_list_from_api()
    results = await asyncio.gather(
        *[run_ansible_playbook(f"deploy_vm.yml --extra-vars 'vm_id={vm.id}'") for vm in vm_list]
    )
    return sum(results)
```

#### 3. Human-in-the-Loop Approvals

Problem: Ansible Tower requires manual static approvals.
Solution:
Dynamic pauses (e.g., "Wait for Slack approval before deleting prod DB").
```python
@workflow.defn
class ProdDeploymentWorkflow:
    async def run(self):
        await run_ansible_playbook("deploy_staging.yml")
        if workflow.is_replaying():
            await workflow.wait_for_signal("prod_approval")
        else:
            await send_slack_approval_request()
            await workflow.wait_for_signal("prod_approval")  # Blocks until approved
        await run_ansible_playbook("deploy_prod.yml")
```

#### 4. Cross-Cloud Orchestration

Problem: Ansible alone can’t coordinate AWS + Azure + GCP workflows.
Solution:
Temporal workflows orchestrate multi-cloud playbooks:
```python
async def migrate_to_aws():
    await run_ansible_playbook("azure_shutdown.yml")
    await run_ansible_playbook("aws_provision.yml")
    if await check_aws_health():
        await run_ansible_playbook("cutover_dns.yml")
```

#### 5. Event-Driven Ansible (EDA) on Steroids

Problem: Ansible EDA reacts to simple events (e.g., webhooks).
Solution:
Complex event chains (e.g., "If CPU > 90% for 5min, scale + notify PagerDuty + ticket").
```python
async def auto_scale_workflow():
    while True:
        metrics = await fetch_cloud_metrics()
        if metrics.cpu > 90:
            await run_ansible_playbook("scale_out.yml")
            await page_team()
            await workflow.sleep(timedelta(minutes=5))  # Cooldown
```

#### 6. Stateful Workflows with Recovery

Problem: Ansible has no memory of past runs.
Solution:
Temporal remembers state (e.g., "Retry only failed nodes after outage").
```python
async def patch_workflow():
    hosts = await get_hosts()
    for host in hosts:
        try:
            await run_ansible_playbook(f"patch.yml -l {host}")
        except ActivityError:
            await log_failed_host(host)
    if workflow.is_replaying():
        await retry_failed_hosts()  # Only retries what crashed
```

#### 7. Time Travel Debugging

Problem: Debugging Ansible failures is painful.
Solution:
Replay workflows exactly (e.g., "See why playbook failed 3 days ago").
```python
# Temporal UI shows full history + inputs/outputs for every step.
```

8. Cross-Tool Chaining

Problem: Ansible can’t seamlessly call Terraform + Kubernetes.
Solution:
Mix tools in one workflow:

```python
async def deploy_full_stack():
    await run_terraform("apply")
    await run_kubectl("apply -f k8s/")
    await run_ansible_playbook("configure_ingress.yml")
```

## Dynamic Ansible inventory

Here’s how to create a dynamic Ansible inventory using a Temporal workflow that fetches host data from a CMDB via REST API, ensuring real-time, fault-tolerant infrastructure management:
```
Temporal Workflow → Fetches CMDB Data (REST API) → Generates Dynamic Inventory → Runs Ansible
```

```python
#!/usr/bin/env python3
import requests
import json
import argparse

def fetch_cmdb_hosts(cmdb_api_url: str, token: str) -> dict:
    headers = {"Authorization": f"Bearer {token}"}
    response = requests.get(f"{cmdb_api_url}/hosts", headers=headers)
    response.raise_for_status()
    return response.json()

def generate_inventory(cmdb_data: dict) -> dict:
    inventory = {
        "_meta": {"hostvars": {}},
        "all": {"hosts": []},
        "web": {"hosts": []},
        "db": {"hosts": []}
    }
    
    for host in cmdb_data["hosts"]:
        inventory["all"]["hosts"].append(host["name"])
        inventory["_meta"]["hostvars"][host["name"]] = {
            "ansible_host": host["ip"],
            "ansible_user": host["user"],
            "ansible_become": True
        }
        if host["role"] == "web":
            inventory["web"]["hosts"].append(host["name"])
        elif host["role"] == "db":
            inventory["db"]["hosts"].append(host["name"])
    
    return inventory

if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("--list", action="store_true", help="List all hosts")
    args = parser.parse_args()
    
    if args.list:
        cmdb_data = fetch_cmdb_hosts(
            cmdb_api_url="https://cmdb.example.com/api",
            token="your-api-token"
        )
        print(json.dumps(generate_inventory(cmdb_data)))
```

---

- Complex multi-cloud deployments
- Mission-critical workflows (e.g., banking migrations)
- Self-healing infra (e.g., auto-remediation)
