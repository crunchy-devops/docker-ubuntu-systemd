Of course. Here is an example of a Puppet infrastructure using an External Node Classifier (ENC) and an explanation of different deployment strategies.

An **External Node Classifier (ENC)** is a script or application that tells your Puppet Server which classes, parameters, and environment to apply to a specific node. This is more flexible and scalable than defining all nodes in the `site.pp` file. The Puppet Enterprise (PE) console is a powerful, GUI-based ENC.

---

### ## 🏢 Example Infrastructure Setup

First, let's define the roles for our 20 container hosts. The "role and profile" pattern is a best practice in Puppet. A **profile** class combines technical modules (like `nginx` or `mysql`), and a **role** class includes one or more profiles for a specific machine type.

| Role | Node Count | Puppet Role Class | Includes Profile Class | Manages |
| :--- | :--- | :--- | :--- | :--- |
| **Database** | 5 | `role::database` | `profile::database` | PostgreSQL/MySQL |
| **Web Server** | 10 | `role::web` | `profile::webserver`, `profile::app` | Nginx, Application |
| **Monitoring** | 5 | `role::monitoring` | `profile::monitoring_server`| Prometheus/Icinga |
| **All Nodes** | 20 | `role::base` | `profile::base` | Users, NTP, SSH |

---

### ## 📇 ENC Implementation Example

You would configure your ENC to assign the correct **role** class to each node. All other classes are included via that single role class. This keeps the ENC configuration clean and simple. Here are two ways to do this:

#### Method 1: Puppet Enterprise (PE) Console

The PE Console is the most common way to manage node classification. You would create node groups that match nodes based on rules.

1.  **Create a Base Group:**
    * **Group Name:** `All Nodes`
    * **Parent:** (none)
    * **Rule:** `trusted.certname` -> `matches regex` -> `.*` (This matches all nodes)
    * **Classes:** `role::base`

2.  **Create Role-Specific Groups:**
    * **Group Name:** `Database Servers`
    * **Parent:** `All Nodes` (This ensures they also get `role::base`)
    * **Rule:** `trusted.fact.pp_role` -> `=` -> `database`
    * **Classes:** `role::database`

    * **Group Name:** `Web Servers`
    * **Parent:** `All Nodes`
    * **Rule:** `trusted.fact.pp_role` -> `=` -> `web`
    * **Classes:** `role::web`

    * **Group Name:** `Monitoring Servers`
    * **Parent:** `All Nodes`
    * **Rule:** `trusted.fact.pp_role` -> `=` -> `monitoring`
    * **Classes:** `role::monitoring`

To make this work, you'd configure a "trusted fact" named `pp_role` on each container when it's built (e.g., in `/etc/puppetlabs/facter/facts.d/pp_role.yaml`). For example, a web container would have a file containing:
`pp_role: web`

#### Method 2: Custom ENC Script (Open Source Puppet)

If you're not using PE, you can write a simple script. The Puppet Server calls this script with the node's name. The script returns YAML.

**`/etc/puppetlabs/code/enc_script.sh`**
```bash
#!/bin/bash

CERTNAME=$1
ROLE=$(echo $CERTNAME | cut -d'-' -f1) # Assumes nodes are named like 'db-01', 'web-01'

# Default class for all nodes
BASE_CLASS="role::base"

case $ROLE in
  db)
    ROLE_CLASS="role::database"
    ;;
  web)
    ROLE_CLASS="role::web"
    ;;
  mon)
    ROLE_CLASS="role::monitoring"
    ;;
  *)
    ROLE_CLASS=""
    ;;
esac

# Output YAML for Puppet
echo "---"
echo "classes:"
echo "  - ${BASE_CLASS}"
if [ -n "$ROLE_CLASS" ]; then
  echo "  - ${ROLE_CLASS}"
fi
echo "environment: production"
```

---

### ## 🚀 Deployment Optimization Strategies

Once your code is written and your nodes are classified, you need to decide *how* to deploy the changes. Puppet offers several strategies.

#### A. Serial Deployment (Canary)

This is the **safest** method. You deploy changes to one node at a time to ensure nothing breaks before impacting the whole group. This is often called a "canary deployment."

* **Concept:** One-by-one rollout.
* **Use Case:** Testing high-risk changes (e.g., a major database version upgrade).
* **How to do it:** Use the `puppet job` command to target specific nodes.
    ```bash
    # Deploy to the first database container
    puppet job run --nodes db-01.example.com

    # After verifying it's okay, deploy to the next one
    puppet job run --nodes db-02.example.com
    ```

#### B. Parallel Deployment (Orchestrated Rollout)

This is the **fastest** method for controlled changes. You push the change to all nodes in a specific group at the same time or in concurrent batches.

* **Concept:** Deploy to a group of nodes simultaneously.
* **Use Case:** Rolling out a new feature to all web servers immediately.
* **How to do it:** Use a PQL (Puppet Query Language) query with `puppet job`.
    ```bash
    # Deploy to all 10 web servers at once
    puppet job run --query 'nodes[certname] { facts.pp_role = "web" }'

    # You can also control concurrency in PE to, for example,
    # deploy to only 25% of the web servers at a time.
    ```

#### C. Asynchronous Deployment (Standard Agent Run)

This is the **default** Puppet behavior. You don't "push" the change; you let each agent check in on its own schedule and pull the new configuration.

* **Concept:** Agents pull changes independently.
* **Use Case:** Low-risk changes or routine configuration enforcement where exact timing isn't critical (e.g., updating an SSH authorized keys file).
* **How to do it:**
    1.  Merge your code changes into your `production` environment in Git.
    2.  Deploy the code to your Puppet Server (`puppet-code deploy`).
    3.  **Do nothing else.** Each agent will automatically apply the changes the next time it runs (typically every 30 minutes). This happens asynchronously without any direct orchestration. You can preview the impact using `--noop` mode before letting it apply automatically.
