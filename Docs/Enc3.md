Okay, I understand. The document `puppet_enc_deployment_strategies` already includes alternatives for users not using Puppet Enterprise, and we can certainly focus on those.

For an **External Node Classifier (ENC)** with open-source Puppet:
* You would typically use a **custom ENC script** as described in section 2 of the document. This script is called by the Puppet Server and outputs YAML to classify the node.
* Tools like `Hiera` can also be used extensively for classification and parameter lookups, often in conjunction with a simple ENC or even just `site.pp` for smaller setups.

For **Deployment Orchestration** (serial and parallel deployments) without Puppet Enterprise:
* **Puppet Bolt** is the recommended open-source orchestration tool from Puppet. Section 4 mentions using `bolt task run puppet_agent::run` to trigger Puppet runs on target nodes, which can be used for both serial and parallel deployments by adjusting the targets and query parameters.
* **Custom scripting** (e.g., shell scripts using SSH to trigger `puppet agent -t` on nodes) is also a viable option, though it requires more manual effort to build and maintain.
* Tools like **MCollective (mco)** with the `mco puppet` plugin were traditionally used for this in open-source setups, and while less emphasized now with Bolt's rise, they are still functional in existing environments.

So, when considering the deployment strategies:
* **Asynchronous:** This is the default open-source Puppet agent behavior.
* **Serial (Canary):** Use `Puppet Bolt` to target individual nodes sequentially, or your custom scripts.
* **Parallel (Orchestrated Rollout):** Use `Puppet Bolt` with queries or node lists to target groups of nodes. You can manage concurrency through Bolt's capabilities or by how you batch commands in custom scripts.

The core concepts of roles, profiles, and the deployment strategies themselves remain the same regardless of whether you're using Puppet Enterprise or open-source Puppet with tools like Bolt.
