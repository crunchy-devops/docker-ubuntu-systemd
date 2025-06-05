Puppet Infrastructure with ENC and Optimized DeploymentsThis document outlines an example Puppet infrastructure managing containerized hosts, leveraging an External Node Classifier (ENC), and details various deployment optimization strategies.1. Example Infrastructure OverviewWe have a Puppet Server managing 20 containerized remote hosts, grouped by function:Database Hosts (5 containers): Responsible for running database services (e.g., PostgreSQL, MySQL).Web Server Hosts (10 containers): Responsible for serving web applications (e.g., Nginx, Apache, application code).Monitoring Hosts (5 containers): Responsible for monitoring the infrastructure (e.g., Prometheus, Grafana, Nagios/Icinga).Puppet Code Philosophy: Roles and ProfilesWe will use the standard "roles and profiles" pattern:Profiles: These classes manage specific technologies or configurations (e.g., profile::base_linux, profile::nginx, profile::postgresql_server, profile::prometheus_node_exporter). They are the building blocks.Roles: These classes represent the business function of a server. A role class includes one or more profile classes. Each node should only ever be assigned one role class.Defined Roles for our Infrastructure:Node TypeContainer CountPuppet Role ClassKey Profile(s) IncludedBase (All Nodes)20role::baseprofile::base_linux (users, ssh, ntp, base packages)Database Server5role::database_serverprofile::base_linux, profile::postgresql_serverWeb Server10role::web_serverprofile::base_linux, profile::nginx, profile::webapp_XMonitoring Server5role::monitoring_serverprofile::base_linux, profile::prometheus_server(Note: The role::base is often implicitly included or handled by having common profiles in each specific role.)2. External Node Classifier (ENC)An ENC is a script or external system that the Puppet Server consults to determine:Which classes to apply to a node.What parameters (class parameters) to set for those classes on that node.Which environment the node should be in (e.g., production, development, test).For our containerized setup, the ENC would classify nodes based on some identifying characteristic, often a trusted fact (like pp_role or pp_environment) set during container build/provisioning or a naming convention.Example ENC Logic (Conceptual):Let's assume each container has a trusted fact pp_role set (e.g., via /etc/puppetlabs/facter/facts.d/custom_facts.yaml or an environment variable passed to Facter).If a container reports pp_role: database, the ENC assigns role::database_server.If a container reports pp_role: web, the ENC assigns role::web_server.If a container reports pp_role: monitoring, the ENC assigns role::monitoring_server.All nodes might also implicitly get role::base or have common profiles directly included by their specific roles.How it looks in a site.pp (if NOT using a true ENC, for illustration):# site.pp - This is what an ENC effectively generates for each node.
# Do NOT do this for many nodes in a real site.pp; use an ENC.

node 'db-container-01.example.com', 'db-container-02.example.com', ..., 'db-container-05.example.com' {
  class { 'role::database_server': }
}

node 'web-container-01.example.com', ..., 'web-container-10.example.com' {
  class { 'role::web_server': }
}

node 'mon-container-01.example.com', ..., 'mon-container-05.example.com' {
  class { 'role::monitoring_server': }
}

# A more common way to handle base configurations if not using an ENC
# is a 'default' node definition, or ensuring base profiles are in each role.
node default {
  # Minimal base configuration if a node doesn't match others.
  # Or, ensure role::base_linux is part of every other role.
}
Using a Custom ENC Script (Open Source Puppet):You'd configure node_terminus = exec and external_nodes = /path/to/your/enc_script.sh in puppet.conf. The script would receive the node's certname as an argument and output YAML.Example enc_script.sh:#!/bin/bash
CERTNAME=$1

# Logic to determine role based on CERTNAME or facts (more complex for facts)
# For simplicity, assuming a naming convention: db-*, web-*, mon-*
ROLE_TYPE=$(echo "$CERTNAME" | cut -d'-' -f1)

echo "---"
echo "environment: production" # Or determine dynamically
echo "classes:"

# Common base class for all
echo "  role::base:" # Assuming role::base exists and includes profile::base_linux

case "$ROLE_TYPE" in
  db)
    echo "  role::database_server:"
    ;;
  web)
    echo "  role::web_server:"
    ;;
  mon)
    echo "  role::monitoring_server:"
    ;;
  *)
    # Default or error
    ;;
esac
Note: Accessing facts in a simple shell ENC is harder; often ENCs are small web apps that can query PuppetDB for facts.Puppet Enterprise (PE) Console as ENC:In PE, you'd create node groups:All Nodes Group: Rule: agent_specified_environment = production. Class: role::base.Database Servers Group: Parent: All Nodes. Rule: trusted.facts.pp_role = database. Class: role::database_server.Web Servers Group: Parent: All Nodes. Rule: trusted.facts.pp_role = web. Class: role::web_server.Monitoring Servers Group: Parent: All Nodes. Rule: trusted.facts.pp_role = monitoring. Class: role::monitoring_server.3. Puppet Code Structure (Roles & Profiles)Example profile::base_linux (Simplified):# manifests/profile/base_linux.pp
class profile::base_linux {
  # Ensure essential packages are present
  package { ['ntp', 'curl', 'vim', 'sudo']: ensure => installed }

  # Manage NTP service
  service { 'ntpd': # or 'chronyd' depending on distro
    ensure => running,
    enable => true,
  }

  # Include other base configurations like user management, ssh hardening, etc.
  # include profile::users
  # include profile::ssh_hardening
}
Example role::web_server (Simplified):# manifests/role/web_server.pp
class role::web_server {
  include profile::base_linux  # Common configurations
  include profile::nginx       # Manages Nginx installation and config
  include profile::webapp_X    # Manages deployment of your specific web application
}
4. Deployment Optimization StrategiesOnce your code is ready and pushed to the Puppet Server's code directory (e.g., via Git and r10k or puppet-code deploy), agents will pick up changes on their next run (typically every 30 minutes). However, for controlled rollouts, you can use Puppet's orchestration capabilities (available in Puppet Enterprise, or with open-source tools like mco puppet or custom scripting).A. Asynchronous Deployment (Standard Agent Behavior)Concept: This is the default behavior. Each Puppet agent checks in with the Puppet Server at a regular interval (defined by runinterval in puppet.conf, usually 30 minutes). If there's new catalog information (due to code changes or ENC updates), the agent applies it.How it works:You commit your Puppet code changes to your version control system (e.g., Git).You deploy this code to your Puppet Server(s) (e.g., r10k deploy environment production or puppet-code deploy --all).Agents will pick up these changes individually and asynchronously as their runinterval triggers a new run.Pros:Simple, no direct orchestration needed.Resilient; if an agent is offline, it will get the changes when it comes back.Cons:No direct control over when specific nodes get updated.Rollout can be slow across a large fleet if runinterval is long.Harder to correlate deployment with issues if changes trickle out.Optimization/Usage:Suitable for low-risk, routine changes (e.g., updating a list of approved SSH CAs, minor package updates).You can trigger a run on a specific agent manually if needed: puppet agent -t on the agent itself.B. Serial Deployment (Canary / Phased Rollout)Concept: Intentionally deploy changes to a small subset of nodes (e.g., one or two per group) first. Monitor them for issues before rolling out to the rest. This is a "canary" deployment.How it works (with Puppet Orchestrator - part of PE, or via bolt or custom scripting):Deploy your code to the Puppet Server as usual.Use an orchestration tool to trigger Puppet runs on specific nodes or a small group.Puppet Enterprise (puppet job run):# Target a single web server container
puppet job run --nodes web-container-01.example.com

# After verifying, target another
puppet job run --nodes web-container-02.example.com
Puppet Bolt:bolt task run puppet_agent::run --targets web-container-01.example.com
Monitor these canary nodes closely for any adverse effects (application errors, performance degradation, Puppet run failures).If canaries are healthy, proceed to deploy to a larger group or the entire fleet (see Parallel Deployment). If not, halt the deployment and investigate.Pros:Greatly reduces the blast radius of a problematic change.Allows for early detection of issues.Cons:Slower overall deployment time than parallel.Requires active monitoring and decision-making at each step.Optimization/Usage:Essential for high-risk changes (e.g., major version upgrades of databases, core application updates, significant OS-level changes).Define clear success criteria for canary nodes before proceeding.C. Parallel Deployment (Orchestrated Rollout)Concept: Trigger Puppet runs on multiple nodes simultaneously or in controlled concurrent batches.How it works (with Puppet Orchestrator/Bolt):Deploy code to the Puppet Server.Use the orchestrator to target a group of nodes.Puppet Enterprise (puppet job run with PQL query):# Target all database containers
puppet job run --query 'nodes[certname] { facts.pp_role = "database" }'

# Target all web containers, but run on only 2 at a time (concurrency)
puppet job run --query 'nodes[certname] { facts.pp_role = "web" }' --concurrency 2
Puppet Bolt:bolt task run puppet_agent::run --query 'nodes[certname] { facts.pp_role = "database" }'
# Bolt's concurrency is controlled by its own configuration or command-line flags.
Monitor the job progress and the health of the targeted group.Pros:Much faster rollout to a large number of nodes compared to purely asynchronous or fully serial.Provides control over the scope and speed of the deployment.Cons:If a change is bad, it can impact multiple systems quickly (though less than a fully unthrottled asynchronous rollout on a short runinterval).Requires an orchestration tool.Optimization/Usage:Good for deploying tested changes to entire roles or environments.Often used after successful canary deployments.Adjust concurrency based on the risk of the change and the capacity of your systems/team to respond to issues. For example, for the 10 web servers, you might do:Canary: web-container-01Batch 1 (Parallel): web-container-02, web-container-03 (concurrency 2)Batch 2 (Parallel): web-container-04 to web-container-10 (concurrency 3 or more)Summary of Choosing a StrategyRoutine, Low-Risk: Asynchronous (default agent runs) is often fine.Moderate Risk, Tested Changes: Parallel deployment to specific roles/groups, possibly after a small canary.High-Risk, Untested in Prod: Serial (canary) deployment is crucial. Start with one node, then a small batch, then wider.By combining a robust ENC with thoughtful deployment strategies, you can manage your containerized infrastructure efficiently and safely with Puppet.
