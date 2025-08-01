# DataStax Mission Control and Hyper-Converged Database (HCD) Workshop Guide

## Workshop Overview

**DataStax Mission Control** is a unified management platform for Apache Cassandra® and DataStax databases. It acts as a “single pane of glass” for deploying, monitoring, and managing Cassandra clusters across environments. Mission Control provides a centralized web console and automation via Kubernetes operators, enabling operations teams to manage many database clusters from one place without manual per-node work. Key capabilities include one-click cluster provisioning, integrated monitoring and logging, automated repairs, backups, and seamless upgrades – all following best practices drawn from years of running Cassandra at scale. This means faster deployments, easier day‑2 operations (like scaling and repairs), and **zero-downtime maintenance** through rolling upgrades.

**DataStax Hyper-Converged Database (HCD)** is a next-generation, self-managed database built on Apache Cassandra. HCD extends Cassandra with **cloud-native enhancements** and new capabilities (such as vector search for AI/ML applications) while unifying transactional and analytical workloads in one database. It is designed for modern data centers and AI-ready infrastructure, allowing enterprises to run high-performance Cassandra-based databases with built-in support for vector storage and search. HCD includes DataStax Mission Control to deploy and operate clusters on Kubernetes, bringing cloud-like automation to on-prem or hybrid environments. In short, HCD offers the power of Cassandra with easier operations and integration for emerging use cases like generative AI.

**Value Proposition & Use Cases:** Together, Mission Control and HCD provide significant value for operations teams managing Cassandra/DSE clusters:

* **Unified Management:** Mission Control serves as a single control plane for all your Cassandra/DSE/HCD clusters – across on-premises, cloud, or hybrid setups. This consolidation means no more juggling of separate tools (OpsCenter for metrics, homegrown scripts for deployments, etc.) – everything is managed in one UI or via one CLI.

* **Rapid Provisioning & Scaling:** Mission Control automates Cassandra cluster deployment on Kubernetes, cutting setup time from days to minutes. Teams can spin up new clusters or add capacity with a few clicks or a small YAML change, which accelerates dev/test environment creation and scaling for growth. This agility supports use cases like on-demand test clusters, auto-scaling for peak loads, and quicker project onboarding.

* **Reduced Operational Overhead:** Common maintenance tasks that typically require manual steps and careful scheduling – adding nodes, replacing failed nodes, running repairs, taking backups, etc. – are handled by Mission Control’s operators and built-in services. For example, backups use the Medusa tool under the hood and repairs use Cassandra Reaper, but ops teams don’t have to set those up separately. The automation ensures these tasks are done correctly and consistently, reducing risk of human error.

* **Zero-Downtime Updates:** Mission Control enables rolling upgrades of Cassandra/DSE/HCD versions and rolling configuration changes **without downtime**. It sequentially updates one node at a time, automating what DBAs used to do manually (drain node, upgrade, reboot, repeat). This means you can apply patches and updates in production during business hours, maintaining continuous service availability – a huge win for mission-critical use cases that can’t afford extended outages.

* **Integrated Observability:** Out-of-the-box, Mission Control provides robust monitoring and alerting tailored to Cassandra clusters. It deploys a metrics collection stack (Prometheus) and aggregates logs from all nodes, presenting them in a unified dashboard. Ops teams get immediate visibility into cluster health (e.g. read/write throughput, latency, node status) and can troubleshoot from the Mission Control UI without SSHing into servers. Alerts can be set for key conditions (node down, high usage, etc.), enabling proactive issue detection.

* **Use Cases:** This combined solution is ideal for enterprises modernizing their Cassandra deployments. Example use cases include: deploying **Cassandra-as-a-Service** on Kubernetes for internal teams, managing geo-distributed clusters with a centralized tool, ensuring high uptime for customer-facing databases through automated failover and repairs, and integrating real-time **AI/ML workloads** with vector search on a scalable NoSQL backend (HCD). HCD’s support for both *traditional schema data and AI vector data* makes it versatile for applications from IoT and ecommerce to GenAI chatbots – and Mission Control makes these deployments operationally feasible at scale.

## Workshop Setup and Environment

**Kubernetes Environment:** In this workshop, DataStax Mission Control is deployed on a Kubernetes cluster which serves as the **Control Plane** for managing Cassandra/HCD. For simplicity, the control plane and data plane are the same cluster in our setup (e.g. an Amazon EKS or local Kind cluster), so the Cassandra nodes will be launched as pods in this K8s cluster.


> 🛠️ Attendees are expected to install Mission Control as part of the workshop exercises.



* **Local Mission Control Setup (Kind)** – *GitHub repo with step-by-step instructions:* [Mission Control Local Setup README](https://github.com/difli/mission-control-local-setup/blob/main/README.md) and the [Kind/EKS setup guide](https://github.com/difli/mission-control-local-setup/blob/main/ec_2_mission_control_setup.md). This demonstrates installing Mission Control using KOTS (Kubernetes Off-The-Shelf) in a local or EC2 environment.

* **Helm/EKS Terraform Setup** – *Terraform scripts for AWS EKS:* For a production-like install, see the Terraform example [tf-aws-mission-control](https://github.com/Scharlotten/tf-aws-mission-control)  which uses Helm charts to deploy Mission Control onto an EKS cluster.

## Key Concepts: Kubernetes and Mission Control Architecture

Before diving into the hands-on labs, let’s recap a few key concepts that underpin how Mission Control manages Cassandra/HCD:

* **Kubernetes Constructs for Cassandra:** Mission Control leverages Kubernetes to orchestrate database clusters. Each Cassandra node runs in a Kubernetes **Pod** (with its own container). Cassandra nodes in a datacenter are managed as a **StatefulSet**, ensuring stable network identities and persistent storage for each pod. A Kubernetes **Service** abstracts the Cassandra seed nodes for discovery and client access. Mission Control defines Cassandra clusters via Kubernetes **Custom Resources** (CRDs) – specifically a `MissionControlCluster` CR – which describes the desired state (number of datacenters, nodes, version, etc.). **Operators** (custom controllers running in K8s) watch these CRs and perform actions (create pods, attach volumes, configure Cassandra YAML, etc.) to reconcile the actual state to the desired state. In essence, Kubernetes provides the automation backbone: if a Cassandra pod fails, the StatefulSet will restart it; if we update the cluster definition, the operator will apply those changes cluster-wide.

* **Mission Control Architecture:** Mission Control is deployed in a **Control Plane / Data Plane** architecture. The Control Plane is the Mission Control application (running on K8s) that includes the web UI, REST API, and control components (operators, controllers). It can manage multiple **projects** and clusters. A **Project** in Mission Control is a logical grouping of clusters (e.g. by environment or team) to organize resources. In our workshop, a project named “Workshop” is used to contain the demo cluster(s). The Data Plane refers to the actual database cluster infrastructure. In many cases (like this workshop), the data plane is simply the same Kubernetes cluster where Mission Control is running, and Cassandra pods are launched there. Mission Control can also manage data planes outside of its own K8s cluster – for example, deploying Cassandra on VMs by using remote operators – but our focus is on Kubernetes-based deployment (which is the typical scenario for HCD).

* **Integrated Services:** Mission Control bundles several Cassandra operational tools into its platform. It has a **metric collector** (built on Prometheus) that scrapes metrics from each node and a **log aggregator** (using Grafana Loki or similar) to collect logs, all wired into the UI. For repair operations, Mission Control integrates **Reaper**, a Cassandra repair automation tool, so that repairs can be run or scheduled from the Mission Control UI without a separate Reaper installation. For backups, Mission Control uses **Medusa** behind the scenes – a backup service that coordinates Cassandra snapshots and uploads to cloud storage. These components (metrics, logging, backup, repair) are deployed as part of Mission Control’s control plane, often as separate pods or microservices in the `mission-control` namespace. The benefit is that operators get “batteries-included” – you don’t need to set up Grafana, Prometheus, Reaper, etc. manually; Mission Control provides a turnkey solution for full cluster observability and maintenance.

* **Declarative Management:** A core principle of Mission Control (inherited from Kubernetes) is **declarative configuration**. You declare the desired state of the cluster (for example, “3 nodes of version X in datacenter dc1”) and Mission Control’s operators continuously **reconcile** the actual state to match it. If you want to change something – e.g., add a node or enable a new feature – you update the declaration (via UI or by applying a YAML) and the system handles the rolling changes. This approach applies to everything: scaling, config changes, upgrades, etc. It ensures consistency and repeatability. For instance, a change to a config parameter in the cluster CR will trigger a rolling restart of nodes to apply it safely. As operators, this means we spend less time on manual procedures and more on supervising (or automating) higher-level policies.

With these concepts in mind, let’s proceed to the hands-on labs. Each lab will walk through a different Mission Control capability using HCD (or Cassandra/DSE) in a Kubernetes environment.

---

## Lab 1: Mission Control Installation and Setup (Pre-lab)

*Objective:* Verify that DataStax Mission Control is up and running, and familiarize yourself with the environment before creating a cluster.

**Step 1 – Access the Mission Control UI:** Open your web browser to the Mission Control interface. If you’re running Mission Control locally, you may need to port-forward the service (e.g., `kubectl port-forward svc/mission-control-ui -n mission-control 8080:80` and browse to `http://localhost:8080`). In a cloud deployment, use the provided URL. Log in with the admin credentials set during installation (for community edition, the default user might be `admin@example.com` unless changed). You should see the Mission Control dashboard – a web console with a sidebar for Projects and a main area for cluster information.

**Step 2 – Create or Select a Project:** In the sidebar, locate the **Projects** section. Projects help organize clusters by team or environment. If a “Workshop” project (or a project for this lab) is already created, click on it to enter that context. Otherwise, create a new project (using the “New Project” button) and give it a name (e.g., “Workshop”). Within a project, you will later create clusters. Ensuring you have the correct project selected will keep our work isolated and easy to manage.

> **About Projects:** A project is just a logical grouping in Mission Control – it doesn’t affect how clusters are deployed, but it’s useful for multi-tenant scenarios or to separate prod vs dev clusters. In our case, using one project for the workshop makes it simple to find our demo cluster(s) in one place.

**Step 3 – Confirm Control Plane Health:** In the Mission Control UI, check that the control plane components are healthy. There may be a section listing controllers/operators status. All components (Cluster operator, Medusa backup, Reaper, etc.) should show green/active status. If any component is not ready, troubleshoot before proceeding (for example, ensure the Mission Control pods are running with `kubectl get pods -n mission-control`). A healthy control plane is indicated by an “All systems go” type status in the admin console. Now we are ready to start creating and managing a Cassandra/HCD cluster.

---

## Lab 2: Creating a Cassandra/HCD Cluster with Mission Control

In this lab, we will create a new Cassandra cluster (using DataStax HCD) through the Mission Control UI. This demonstrates how easy it is to spin up a fully configured cluster with just a few clicks, compared to manual provisioning.

**Step 1 – Start Cluster Creation:** In the Mission Control UI, ensure you have your project selected (e.g., “Workshop”). Click the **“Create Cluster”** button. This opens a cluster provisioning wizard or form. For a beginner-friendly experience, select the **“Simple” (GUI) mode** if prompted (as opposed to uploading a YAML definition).

**Step 2 – Enter Cluster Details:** Fill out the cluster creation form with the desired settings:

* **Cluster Name:** Enter a name for your cluster, for example `demo-cluster` (names must be DNS-friendly).
* **Cluster Type:** Select **DataStax Hyper-Converged Database (HCD)** if available. (In some versions of Mission Control, you might choose “Apache Cassandra” and then pick an HCD distribution version, since HCD is Cassandra-based. The key is to choose a Cassandra 4.0-based option. In earlier workshop materials, DSE 6.8 was used due to audience familiarity, but here we will use HCD to explore the latest capabilities.)
* **Server Version:** Choose the version corresponding to HCD, e.g. **HCD 1.1 (Cassandra 4.0.x)**. If HCD is not explicitly listed, you could choose Apache Cassandra 4.0 or DataStax Enterprise 6.8 (whichever the environment is licensed for) – but assume HCD for this exercise.
* **Topology:** Define the datacenters and nodes. For this lab, configure a single datacenter, name it “dc1”, with **3 nodes** in that datacenter. This will create a minimal resilient cluster (RF=3). If the form has a field for racks or zones, you can leave defaults or set 1 rack. (We choose 3 nodes because it’s a common small cluster size that still provides replication and fault tolerance.)
* **Storage and Configuration:** Select a storage class for persistent volumes if required (use the default provided by your K8s cluster, e.g. gp2 on EKS or standard on Kind). For configuration, Mission Control might allow selecting a preset config profile; use the “default” profile for now, which has standard Cassandra settings. No need to adjust JVM options or GC – defaults will suffice for the workshop.

Review the form to ensure all required fields are filled. Your screen should look similar to this with the cluster name, type (HCD/Cassandra), version, and topology filled in:

![Mission Control cluster creation form (filled out)](./images/cluster-setup.png)

**Step 3 – Launch the Cluster:** Click **“Create”** (or the final **Submit**) to create the cluster. Mission Control will begin deploying the Cassandra cluster in the background. The UI will navigate to a cluster status or overview page for the new cluster (often showing a progress indicator). Initially, you will see the cluster state as **Provisioning** or similar. The UI may list the datacenter and the nodes (e.g., `dc1-node-0`, `dc1-node-1`, `dc1-node-2`), each with a status. At first, nodes might show “Initializing” or “Pending” as Kubernetes schedules the pods and they start up.

Behind the scenes, Mission Control has created the Kubernetes resources needed: a `MissionControlCluster` custom resource that describes this cluster, which in turn caused the Cassandra operator to create a **StatefulSet** with 3 pods for “dc1”. Each pod is attaching a persistent volume, pulling the HCD/Cassandra container image for the specified version, and configuring the node (setting the cluster name, seeds, `cassandra.yaml` properties, etc.) automatically. Mission Control’s control plane continuously monitors this process and will report progress until the desired state (3 running Cassandra nodes) is achieved.

> **Note:** In a traditional setup, at this point an admin would be installing Cassandra on three servers, configuring them, and starting them one by one. Mission Control handles all those steps for us via Kubernetes automation. We’ve declared our desired cluster, and the system is doing the heavy lifting (container orchestration, config management, service wiring). This drastically reduces setup time and ensures consistency.

**Step 4 – Monitor Cluster Startup:** While the cluster is provisioning, you can observe the status. The UI might show a **ring view** or simply a list of nodes with statuses updating from initializing to running. If there’s a logs tab, you could click a node to see its console output (showing Cassandra starting up, joining the cluster, etc.). Typically, one node will be designated as the seed node; others will contact it and join the ring. Within a few minutes, all 3 nodes should reach **Running/Active** state.

![Mission Control cluster status view – 3-node cluster running](./images/cluster-dashboard.png)

Once the UI indicates all nodes are **Up** (green status), the cluster is fully deployed. You should see details such as the cluster name, datacenter name (dc1), the server version (e.g., HCD 1.1 or DSE 6.8.x), and that there are 3 nodes active. Celebrate – you just deployed a multi-node Cassandra cluster with a single action!

**Step 5 – Verify Cluster Health:** In the cluster’s overview page, confirm that each node is reporting healthy. Mission Control will typically surface any issues (if a node failed to start, you’d see a warning). In our case, all nodes are running. The cluster is now ready for use – one could connect via CQLSH or an application, but our focus will be on management operations via Mission Control.

**What Happened Under the Hood?** Mission Control’s operator set ensures the cluster came up correctly: seed nodes were configured, tokens allocated, and Cassandra’s internal gossip has formed a ring with all 3 nodes. If you were to run `nodetool status` (which you can do by exec’ing into a pod or via MC’s UI if it provides that), you would see all nodes UN (Up/Normal) and the token ranges distributed. The process is entirely repeatable – you could create another cluster just as easily, with perhaps a different name or more nodes, and Mission Control would handle that in parallel. This showcases the **ease of provisioning** with Mission Control.

> **Discussion:** For those familiar with manual Cassandra ops, think about the steps that were skipped: provisioning VMs/instances, installing Java and Cassandra, configuring each node’s cassandra.yaml, setting up seed lists, starting services, and checking logs. Mission Control automated all of that in a standardized way. This not only saves time but also reduces errors (every node gets the correct config automatically). In an enterprise, this kind of automation means teams can spin up dev/test clusters on-demand and ensure production clusters are built exactly to spec every time.

**Optional – Enabling Data APIs:** HCD comes with a RESTful **Data API** (powered by Stargate) that allows you to interact with the database over HTTP. Let’s optionally deploy a Data API gateway for our new cluster to demonstrate this capability.

* In Mission Control’s UI, check if there is an option to enable “Data API” or “Stargate” for the cluster. This might be a toggle or an add-on you can deploy to the cluster. If the UI allows adding a Data API node, do so (it might ask for a port or simply add a new deployment).
* If such UI option is not readily available, you can still deploy a Data API pod manually (if you have the YAML for it). For this lab, assume Mission Control has added a data API service.

Once the Data API is running (a new pod will be running, perhaps named `*-data-api-*`), you can access it via a NodePort or port-forward. For example, if the Data API service is exposed on port 30005 (NodePort):

**Option 1:** If on a remote cluster, set up an SSH tunnel to your Kubernetes node or use `kubectl port-forward`:

```bash
# Example: Port-forward the Data API pod to localhost
kubectl port-forward -n <cluster-namespace> pod/<your-data-api-pod-name> 30005:8181
```

Here, replace `<cluster-namespace>` with the K8s namespace of your demo cluster (Mission Control often puts each cluster in its own namespace, e.g., `demo-xxxx`). Also replace `<your-data-api-pod-name>` with the actual pod name (you can find it via `kubectl get pods`). The Data API typically listens on port 8181 internally, which we map to localhost:30005.

Now, open a browser to [http://localhost:30005/swagger-ui/](http://localhost:30005/swagger-ui/) – you should see the Swagger UI for Stargate/Data APIs. This interface allows you to explore and test the REST endpoints.

**Step 6 – Test the Data API:** Let’s create a keyspace via the Data API. The API requires an authentication token. For simplicity, we’ll use the Cassandra superuser credentials (which in HCD default to user `hcd-superuser` and password `password`). These need to be base64-encoded for the API token header. You can encode any string to Base64 using a command like:

```bash
echo -n "hcd-superuser" | base64   # yields aGNkLXN1cGVydXNlcg==
echo -n "password" | base64       # yields cGFzc3dvcmQ=
```

Mission Control may have provided these encoded tokens as part of cluster creation. Using the default values above, construct the token header as `Token: Cassandra:<base64(username)>:<base64(password)>`. For our default, that becomes `Token: Cassandra:aGNkLXN1cGVydXNlcg==:cGFzc3dvcmQ=`.

Now run a sample `curl` to create a new keyspace using the Data API:

```bash
curl -sS --location -X POST "http://localhost:30005/v1/" \
  --header "Content-Type: application/json" \
  --header "Token: Cassandra:aGNkLXN1cGVydXNlcg==:cGFzc3dvcmQ=" \
  --data '{"createKeyspace": {"name": "demo"}}'
```

This sends a request to create a keyspace named “demo” (using the GraphQL API via Stargate). If successful, the response will indicate the keyspace was created. You can verify by using the Swagger UI (try the `getAllKeyspaces` query) or by connecting with CQLSH.

This optional exercise shows that with HCD, you not only have CQL access but also RESTful APIs for data – useful for modern app development. Mission Control itself doesn’t manage your data schemas, but it makes it easy to deploy the Data API alongside your cluster.

---

## Lab 3: Observability and Monitoring

Now that our Cassandra/HCD cluster is running, we’ll explore Mission Control’s built-in monitoring and logging features. Observability is crucial for day-2 operations, and Mission Control offers an integrated view of metrics and logs across the cluster.

**Step 7 – Open the Monitoring Dashboard:** Navigate to the cluster’s **Monitoring** or **Observability** section in the Mission Control UI (the exact name may vary – it could be a “Metrics” tab on the cluster page, or a separate Observability menu). Here you should see dashboards of key metrics aggregated for the cluster.

By default, Mission Control’s dashboard shows graphs for metrics like **CPU usage**, **Memory usage**, **Read/Write throughput**, **Read/Write latency**, and **Disk space** per node. Since our cluster was just created and likely idle, the metrics may be near zero. For a more interesting view, you can generate a bit of load: for instance, open CQLSH (port-forward the service and run a few inserts/selects) or use a stress tool to add some traffic. This will cause the graphs to show activity (e.g., some non-zero read/write ops). Mission Control collects these metrics using an internal Prometheus that scrapes each node’s metrics endpoints. The beauty for ops teams is that **you did not need to set up Grafana/Prometheus yourself** – Mission Control did it for you, and the UI presents the data out-of-the-box.

You can adjust the time window of the charts (e.g., last 5 minutes vs last 1 hour) if the UI allows. Take note of any spikes corresponding to the test load you generated.

![Mission Control cluster monitoring dashboard (graphs of throughput, latency, etc.)](./images/cluster-monitoring.png)

**Step 8 – Drill Down into Node Metrics:** If the interface supports it, select an individual node or datacenter to see more granular metrics. For example, clicking on a node might show its heap usage, garbage collection pause times, compaction activity, etc. This is useful for diagnosing hotspots (e.g., one node using more CPU than others). Also observe any **health status indicators** – typically, Mission Control will flag nodes that are down or have issues (perhaps a red icon next to the node name). At the moment, all our nodes should be green/healthy.

Mission Control can also raise **alerts** for certain conditions. There might be a predefined set of alerts (for instance, alert if a node goes down, or if disk usage > 90%). While we might not configure external alerting in this workshop, it’s good to know that you can integrate Mission Control with systems like PagerDuty, email, or Slack for notifications. The UI might allow you to see active or past alerts. If so, take a look – if a node was down even briefly during startup, an alert might be logged.

**Step 9 – View Cluster Logs:** Next, let’s check the logging aggregation. Find the **Logs** tab or section for the cluster (or per node). Mission Control streams the Cassandra logs from each node to a central view. Select one of the nodes (say node 0) and you should see recent log lines – these correspond to Cassandra’s `system.log` on that node. You’ll likely see entries about the gossip service, hints of the cluster formation, and so on. If nothing much is happening now, that’s okay. The key point is you can search and filter logs here; for example, try typing “WARN” or “ERROR” into the filter to see if any warnings were logged during startup. This is extremely useful for troubleshooting because you don’t have to ssh into individual machines or set up an ELK stack – **Mission Control centralizes all node logs** for you.

If you did the optional Data API step or ran stress, you might see some client connections or CQL statements logged. If not, you can generate a test error by, say, attempting a bad CQL query; it should appear in the logs.

> **Aside:** The integrated monitoring & logging illustrate how Mission Control simplifies ops. Traditionally, you might rely on DataStax OpsCenter for metrics (now deprecated for HCD) and something like `kubectl logs` or external logging for pods. Mission Control combines these into one interface. It deploys Prometheus and a Grafana-style dashboard for metrics, and a log collection (likely via FluentD/Vector and Loki) for logs – but all **under the hood**. As an operator, you just see the outcomes: graphs and logs in the UI. This saves time (no separate monitoring stack to maintain) and provides a consistent view across all clusters.

**Step 10 – (Optional) External Grafana Integration:** While Mission Control’s UI is comprehensive, some users may want to use **Grafana** for custom dashboards or integrate with existing observability systems. Mission Control allows exposing its Prometheus metrics to an internal Grafana instance. We will enable Grafana that comes with Mission Control (disabled by default in some installs) to demonstrate this:

1. **Enable Grafana via KOTS:** Run the following command to enable Grafana in the Mission Control configuration:

   ```bash
   kubectl kots set config mission-control -n mission-control --key grafana_enabled --value 1
   ```

   This sets the config value `grafana_enabled` to true. Next, access the KOTS Admin Console for Mission Control (if you installed via KOTS, this is a web UI at the KOTS URL). There, you would **redeploy** or **apply** the updated config (the command output may instruct to go to the admin console to deploy). In our workshop, assume this has been applied and Mission Control will now deploy a Grafana pod.

2. **Retrieve Grafana Credentials:** Once Grafana is enabled, Mission Control creates a secret with login info. Get the admin user and password by running:

   ```bash
   kubectl get secret mission-control-grafana -n mission-control -o jsonpath="{.data.admin-user}" | base64 --decode && echo
   kubectl get secret mission-control-grafana -n mission-control -o jsonpath="{.data.admin-password}" | base64 --decode && echo
   ```

   The first command will output the Grafana username (likely `admin`), and the second outputs the password (auto-generated). Copy these for later.

3. **Port-Forward Grafana:** Grafana in this setup is exposed internally. To access it, do:

   ```bash
   kubectl port-forward -n mission-control svc/mission-control-grafana 3000:80
   ```

   Now browse to [http://localhost:3000](http://localhost:3000) and you should see the Grafana login. Use the credentials retrieved above to log in.

4. **Explore Grafana Dashboards:** Mission Control likely comes with some pre-built Grafana dashboards for Cassandra metrics. Upon login, you might find a dashboard for your cluster under Grafana’s dashboard list. Open it to see similar metrics as the Mission Control UI, but now in Grafana’s interface. You can use Grafana’s features to set up custom views or alerts if needed. (In a real deployment, one might integrate this Grafana with other data sources too.)

![Grafana dashboard for Cassandra metrics (via Mission Control integration)](./images/grafana-dashboard.png)

This Grafana integration is optional – it demonstrates that Mission Control’s data can be consumed in standard tools. Many ops teams love Grafana, so being able to toggle it on means you’re not locked into only the MC UI. However, **out-of-the-box the Mission Control UI is usually sufficient** for most monitoring needs.

After you finish, you can stop the port-forward (Ctrl+C). We will continue using the Mission Control UI for the next tasks.

**Step 11 – Verify Cluster Stability:** Before moving on, ensure the cluster is still healthy. All nodes should remain **Up** throughout the monitoring exercise. If you left a load running, you can stop it now. Note how Mission Control would highlight any node outages clearly if they occurred (for example, a red status or an alert for down node). It might even attempt to automatically recover a failed node depending on configuration (for instance, by restarting a pod if it crashed). This is part of Mission Control’s **self-healing** design: if a Cassandra node process crashes, Kubernetes will restart the pod; Mission Control will register that and update status, possibly even trigger a replace if the persistent volume is corrupt (with operator intervention). In summary, at a glance you can trust the UI to tell you cluster health.

**(Optional) Exercise – Simulate a Node Failure:** To see Mission Control’s fault tolerance in action, we can manually kill a Cassandra pod and watch the system respond. **This step is optional and should only be done if you have time and the cluster can afford a brief disturbance.**

* Open a terminal and run `kubectl get pods -n <cluster-namespace> -w` (watch mode) to see the cluster’s pods. Identify one of the Cassandra pods (e.g., `demo-cluster-dc1-default-sts-0` or similar).
* Delete the pod forcibly:

  ```bash
  kubectl delete pod <pod-name> -n <cluster-namespace> --grace-period=0 --force
  ```

  For example,:

  ```bash
  kubectl delete pod dse-dc1-r1-sts-0 -n demo-876878ub --grace-period=0 --force
  ```

  (Your names will differ; use your actual namespace and pod name.)

Watch the behavior: Kubernetes will terminate that pod and immediately start a new pod (with a different restart count) because the StatefulSet expects 3 replicas. The new pod will start up and join the cluster. In the Mission Control UI, you’ll likely see node 0 go red/down, then return to green as the new pod comes up. Mission Control might log an event about node restart. Within a couple of minutes, the cluster should be back to full health with data automatically replicated to the new instance (since the persistent volume was preserved, the node may just restart using the same disk; if not, bootstrap will occur). This demonstrates **automatic failure recovery** – the combination of Kubernetes and Mission Control means even if a node crashes, it gets restored without operator intervention.

After the pod is running again, you can stop watching the pods. Our cluster endured a simulated failure and recovered gracefully.

*(End of Lab 3. We’ve seen how Mission Control offers integrated monitoring, logging, and even how it can integrate with Grafana. We also observed the cluster’s resilience to node failure.)*

---

## Lab 4: Upgrading a Cluster (Zero-Downtime Patching)

One powerful feature of Mission Control is performing **rolling upgrades** of your database version with minimal effort. In this lab, we will simulate upgrading our cluster to a newer version of HCD (or Cassandra/DSE) without taking the database offline.

**Scenario:** Assume our cluster is running an older patch version (for example, HCD 1.1.0) and a new patch (HCD 1.1.1) is available with important bug fixes. We’ll upgrade all nodes to the new version through Mission Control.

**Step 12 – Prepare for Upgrade:** In the Mission Control UI, navigate to the cluster’s **Settings/Configuration** page. There may be an “Edit Cluster” button or a YAML editor showing the cluster manifest. Here you can change the cluster’s configuration, including the server version. Verify the current version is listed (e.g., “6.8.25” in DSE terms, or “1.1.0” for HCD, depending on what you chose). Announce what you are about to do: *“We will perform a rolling upgrade of the cluster to the next version, with zero downtime.”* In a manual setting, an upgrade requires careful planning and node-by-node execution. Mission Control will automate that for us.

> **Note:** Zero-downtime is possible because Cassandra is distributed. As long as we upgrade one node at a time and have a suitable replication factor (RF >= 3) and consistency level, the cluster can serve requests with the remaining nodes. Mission Control’s operator encodes this procedure.

**Step 13 – Update the Version:** Using the UI form or YAML editor, change the cluster’s **server version** to the target version (e.g., from 1.1.0 to **1.1.1**). If there is a dropdown, select the new version and confirm the change (the UI might have an “Upgrade” or “Save changes” prompt). If using YAML, find the `serverVersion` field and modify it, then save. Once you submit this change, Mission Control recognizes that the desired state is a cluster running the new version.

**Step 14 – Observe the Rolling Upgrade:** Mission Control will initiate a **rolling restart and upgrade** of each node, one at a time. You can follow progress in a few ways:

* Watch the pods with `kubectl get pods -n <ns> -w`. You’ll see one pod (node) go down (terminating) as it’s stopped, then come back up (as a new container with the new version image). Only after the first node is back up will the next node begin upgrading.
* In the Mission Control UI, the cluster status might indicate “Upgrading” and show which node is being updated. Some UIs display a small progress per node or list nodes with version numbers (e.g., node1 now shows 1.1.1 while others still show 1.1.0).
* You can also check the **Logs** tab for the node currently upgrading to see it shut down and start up again on the new version.

Mission Control’s operator performs the equivalent of: `nodetool drain` on node1, shut down Cassandra, update the container image to new version, start Cassandra, wait for it to join ring; then repeat for node2, etc.. This process ensures the cluster stays available (at RF=3, losing one node still allows quorum reads/writes). No client-side configuration changes are needed – drivers will see nodes go down/up one by one, which they handle gracefully.

> **Live observation:** It typically takes a couple of minutes per node to restart on a new version (time to stop, pull new image, start and join). So for 3 nodes, budget \~5-10 minutes. Use this time to discuss how risky manual upgrades can be versus this automated approach. Also note that Mission Control can schedule upgrades during a maintenance window or do them cluster-by-cluster in a rolling fashion across datacenters if needed – but doing one cluster’s nodes one by one is the basic unit.

**Step 15 – Verify Upgrade Completion:** When Mission Control finishes, all nodes will be running the new version. The UI should now list the cluster’s version as the new one globally, and show the cluster in a normal healthy state (no “upgrading” indicators). Confirm this in the UI: for instance, check that each node’s version field (if shown) is updated, or that the cluster overview says “Server Version: X.Y.Z (new version)”. You have successfully upgraded the cluster **with zero downtime** for clients.

**Step 16 – Post-Upgrade Checks:** It’s always good practice to verify the cluster after an upgrade:

* Run some read/write queries (if you have an app or cqlsh) to ensure functionality is normal.
* Check the logs for any warnings or errors on startup of the new version.
* If Mission Control provides a health report, ensure no issues (some systems verify schema versions match, etc.).

Everything should be smooth. This lab highlights a major benefit of Mission Control: performing patch upgrades or minor version upgrades becomes a routine, low-risk task rather than a big project requiring an all-hands weekend effort. Operations teams can keep databases up-to-date more easily, which improves security and performance in the long run.

**(Optional) Configuration Change:** If time permits, you can also demonstrate a configuration change to the cluster in a similar fashion. For example, toggle a Cassandra setting like `dynamic_snitch` or change `concurrent_writes`. This would also be done via the cluster Settings/YAML, and upon saving, Mission Control would do a rolling restart of nodes to apply it. The mechanism is the same as the upgrade – one node at a time. This emphasizes that any cluster-wide change (be it version or config) is handled safely and consistently. We won’t do this now if time is short, but it’s good to know.

*(End of Lab 4. We upgraded the database cluster through Mission Control without downtime, illustrating declarative automation for maintenance.)*

---

## Lab 5: Backup and Restore Operations

Regular backups are essential for database operations. Mission Control simplifies Cassandra backups by coordinating them across the cluster and storing them in an external location (cloud storage or on-prem object store). In this lab, we will configure a backup target and take a full backup of our cluster. We will also discuss the restore process.

**Step 17 – Configure Backup Storage:** In the Mission Control UI, navigate to the **Backup** section for the cluster (it might be under a menu like “Maintenance” or a tab on the cluster page). Before running a backup, ensure Mission Control knows where to store the backup. Typically, this means:

* A **Storage Location** is defined (for example, AWS S3 bucket, GCP bucket, Azure blob container, or even a local MinIO service).
* The credentials for that storage are provided (likely via a Kubernetes Secret or through Mission Control config).

For our workshop, assume the instructor has pre-configured an S3 bucket or MinIO and loaded the credentials into Mission Control. (If you were doing this yourself, you’d create a Secret with your AWS keys and Mission Control’s backup config would reference it). Verify in the UI that a backup location is shown (e.g., “S3 – my-backups-bucket”). If not, you would set one up now, but in our case we proceed, trusting it’s configured.

**Step 18 – Initiate a Backup:** Click the **“Backup Now”** or **“Take Snapshot”** button to start a backup of the cluster. You may get a prompt to choose full or incremental; choose **Full Backup** (the first time you run, it’s full by default). Also, select which keyspaces to include. Typically, you’d back up all keyspaces except system keyspaces. The UI might allow selecting specific keyspaces – for simplicity, back up all keyspaces (the default).

Confirm and start the backup. Mission Control will now orchestrate a backup across all 3 nodes. Under the hood, the backup service (Medusa) triggers a **snapshot** on each node (equivalent to `nodetool snapshot`, which is an instantaneous hard link of SSTable files). Then, it begins uploading the snapshot files from each node to the configured S3/MinIO bucket. This effectively backs up the data files while the cluster is live.

In the UI, you should see the backup job progress – perhaps a status per node or an overall progress bar. It may say “Running” or list stages like “Snapshotting… Uploading…”. This process can take a few minutes, but since our cluster likely has minimal data, it should be quick.

**Step 19 – Monitor Backup Progress:** Wait for the backup to complete on all nodes. The UI will eventually mark the backup as **Completed** (or Success). You might see a timestamp and maybe the size of the backup. Congratulations, you now have a consistent backup of the cluster at this point in time.

The UI likely keeps a history of backups. You might see an entry like “Backup #1 – \[timestamp] – Success”. If you have access to the storage bucket (for instance, via an S3 browser or MinIO console), you could even verify that files have appeared there (e.g., in a folder named after the cluster and date).

> **Point to Highlight:** We just backed up a multi-node cluster with one click and without pausing the cluster. Mission Control/Medusa handled taking snapshots on each node near-simultaneously and uploading them. This means the backup represents a single moment in time across the cluster (technically each node’s snapshot is taken at roughly the same time, which is usually fine under eventual consistency). There was no need for custom scripts or running backup on each node individually – Mission Control coordinated it. This encourages more frequent backups since it’s so easy (no excuse to skip them!).

**Step 20 – (Optional) Schedule Backups:** In a production scenario, you’d schedule regular backups (nightly, weekly, etc.). Mission Control provides a way to schedule recurring backups (possibly via a cron expression in the UI). If time permits, find the scheduling option and set up a dummy schedule (e.g., daily at 2am). You don’t need to wait for it, but know that once set, Mission Control’s control plane will automatically initiate future backups per that schedule. This is far simpler than maintaining cron jobs on each node or a separate scheduler for `nodetool snapshot` jobs.

**Step 21 – Restore (Discussion):** We won’t perform a live restore during the workshop (as it would require wiping or creating a new cluster to import data), but let’s discuss how it works. In the event of data loss or disaster:

* You would go to the **Restore** option in Mission Control, likely on the same page as backups.
* You’d select the backup you want to restore from (by date or backup ID).
* Mission Control can restore **in-place** (to the same cluster) or to a **new cluster**. For a full cluster recovery, typically you’d restore to a new cluster or after wiping the existing one.
* When initiated, the restore process will **create replacement nodes** if needed and use Medusa to download the SSTable files from the backup storage to the nodes, then import them into Cassandra. Essentially, it reverses the backup: new nodes are seeded with the backup data and then brought up.

Mission Control would handle the heavy lifting: ensuring each node gets the right data, coordinating token ranges, etc. All you do is provide target cluster info (and possibly confirm you really want to overwrite data if in-place).

This integrated restore means you have a viable DR strategy with much less toil. No need to manually fetch files, run `sstableloader`, or worry about restoring node by node – the operator does it.

**Step 22 – Backup/Restore Discussion:** Ask yourself or participants: *“How are we doing backups today for Cassandra?”* Some might say they use scripts with `nodetool snapshot` and upload to S3, or perhaps DataStax OpsCenter’s backup (for DSE folks). Mission Control’s approach uses the open-source Medusa, which is a proven tool, but the key difference is **it’s built-in**. There’s a nice UI, scheduling, and central management. This likely increases the reliability of backups in an organization – it’s easy to set up and monitor, so teams will actually do it regularly. Similarly, restores can be tested periodically (you could spin up a new cluster from a backup to practice DR).

*(End of Lab 5. We successfully took a backup of our cluster via Mission Control and reviewed how we’d restore data if needed.)*

---

## Lab 6: Day-2 Operations – Scaling, Repairs, and More

In this final hands-on section, we’ll look at additional day-2 operations that Mission Control simplifies: scaling out a cluster and performing repairs. We’ll also mention other operations like node replacement and cleanups that Mission Control can handle. These tasks typically are the bread-and-butter of ongoing cluster maintenance.

**Step 23 – Scaling Out (Add a Node):** Suppose our data volume or traffic is increasing and we need to add capacity. Traditionally, adding a node to a Cassandra cluster involves setting up a new machine and running `nodetool rebuild` after joining. With Mission Control, it’s a declarative change.

In the Mission Control UI, find the option to **Scale** the cluster. This could be an “Edit” action where you change the number of nodes in the datacenter, or a dedicated “Add Node” button. For example, if we currently have 3 nodes in dc1, change that to **4** nodes. In a YAML view, this means editing the `datacenters[0].size` from 3 to 4. Apply the change.

Mission Control’s operator will detect the desired state now requires 4 nodes and will create a new Cassandra pod to meet that state. Watch the cluster status: a fourth node will appear and start with status “Provisioning/Joining”. Kubernetes will schedule a new pod, attach storage, and launch Cassandra on it, just like the initial ones. Cassandra will auto-discover the new node (since seeds are configured) and the new node will join the ring.

As the new node comes up, the operator will handle **streaming data to it**. When a new node joins a Cassandra cluster, it needs to take on its share of the token ranges. Data from existing nodes is streamed to the new node (this is similar to running `nodetool rebuild` under the covers). Mission Control will manage or monitor this streaming (it might invoke the rebuild process automatically). The cluster remains online throughout; clients may start using the new node once it’s up (drivers usually update their topology and can send traffic to the new node as well).

Eventually, the new node’s status becomes **Up/Normal** in the ring. The UI should now show 4 nodes all healthy. You have scaled out your cluster with essentially one action.

> **Note:** Mission Control ensures the new node has the same config as the others, and it automates what used to be multi-step (adding new seeds lists, updating config, running rebuild). This showcases **elasticity** in a containerized Cassandra – you can respond to increased load by adding nodes more easily than in bare-metal environments. In cloud or Kubernetes contexts, this is very powerful for dynamic workloads.

**Step 24 – Repair (Anti-Entropy Repair):** Cassandra repairs are necessary to fix inconsistencies due to hinted handoff, tombstones, etc., especially if you have replication across nodes. Normally, you’d run `nodetool repair` or use Cassandra Reaper externally. Mission Control integrates repair management via Reaper.

In the UI, navigate to the **Repair** or **Maintenance** section. There should be an option to **Run Repair**. You might be able to choose the keyspace and even the intensity of repair. For demo purposes, pick a small keyspace (like `system_auth` or one of your application keyspaces if you loaded data). Initiate the repair on that keyspace.

Once started, the UI will show the repair job progress – possibly listing percentage done or segments repaired. Reaper (under the hood) breaks the repair into segments per node and coordinates them so as not to overwhelm the cluster. You’ll likely see some status like “Repair running…”. Given our cluster is small and mostly empty, the repair should finish quickly, but if it doesn’t, we won’t wait for 100%. The point is to see that with a click, we triggered an anti-entropy repair across all nodes of that keyspace.

If the UI shows metrics, you might see validation compactions happening on nodes (repair uses those). If an ongoing progress is visible per node, that’s great. If not, trust that the process is underway.

Mission Control also allows **scheduling repairs** (for example, run a cluster-wide repair weekly). This would be similar to scheduling backups – an automated job that runs in the background. You might find a schedule option to set up recurring repairs.

After observing for a bit, you can cancel the repair if it’s supported, or simply note that it’s running and move on (in real large clusters, repairs can take hours, so often they are scheduled for off-hours or done incrementally). The key takeaway is Mission Control made this a one-click or one-command operation, integrated with the UI, whereas previously an operator might have to maintain a separate Reaper service or cron jobs.

**Step 25 – Other Day-2 Operations (Summary):** We’ve actively tried scaling and repairs. Mission Control supports many other day-2 tasks through its UI:

* **Node Replacement:** If a node/pod dies and cannot recover (e.g., disk loss), Mission Control can replace it with a fresh node automatically. You would typically click “Replace node” on the failed node entry, and the operator will create a new pod to take over its token ranges. This incorporates what would be a `nodetool removenode` / `bootstrap new node` sequence, done seamlessly.

* **Rolling Restarts:** For scenarios like OS kernel updates or certain configuration changes that require a restart, Mission Control can perform a rolling restart of all nodes. The UI likely has a “Restart Cluster” action, which, similar to upgrades, will cycle through each node one by one and restart it. This ensures continuous availability while rebooting the whole cluster.

* **Cleanup:** After reducing replication factor or removing a node, you should run `nodetool cleanup` on remaining nodes to remove orphaned data. Mission Control can coordinate cleanup operations across nodes with a single command in the UI. This saves you from running cleanup manually on each node.

* **Rebuild/New DC:** If adding a new datacenter (for multi-region clusters) or if a node has been down for a long time, you might need `nodetool rebuild`. Mission Control would handle data center additions similarly to how we added a node – by declaratively adding a new DC with X nodes in the cluster definition. The operator will deploy those and stream data (rebuild) automatically.

* **Configuration Management:** Any cluster-wide config changes (like enabling encryption, changing compaction strategy, etc.) can be applied via the Mission Control cluster config. The operator ensures all nodes get the change, using rolling restarts if needed (as we discussed). This central config pushes avoids mistakes of updating some nodes and not others.

In summary, virtually every routine operation you’d perform with `nodetool` or through custom scripts is available in Mission Control’s interface. By this point, we have covered provisioning, monitoring, scaling, repairs, backups, and upgrades – a comprehensive set of operations. All were done through one unified tool. This dramatically simplifies the operational workflow for Cassandra/HCD.

*(End of Lab 6. Our cluster has been through various day-2 ops tasks under Mission Control’s guidance, illustrating the platform’s breadth of functionality.)*

---

## Conclusion: Mission Control Key Benefits for Ops Teams

To wrap up, let’s summarize why DataStax Mission Control (together with HCD) is a game-changer for Cassandra/DSE operations:

* **End-to-End Management:** Mission Control provides a **unified operations console** for all phases of the database lifecycle. From deploying new clusters to monitoring, from scaling to backups and repairs, it’s handled in one place. This eliminates context-switching between multiple tools or manual processes, greatly simplifying an operator’s workflow.

* **Speed and Agility:** Tasks that used to take significant time and planning are now faster. Provisioning a cluster or adding nodes is done in minutes via a declarative spec. This means teams can respond quickly to new requirements (e.g., spin up a test cluster for a developer, expand capacity before Black Friday, etc.). The agility also extends to applying changes – rolling out a config update or patch is just a matter of editing a setting and letting Mission Control handle it, rather than touching each node individually.

* **Zero-Downtime Upgrades and Patches:** Mission Control enables upgrading Cassandra/DSE versions with **no downtime** by automating rolling upgrades. This is crucial for maintaining security and stability; ops teams can stay current with patches without scheduling lengthy maintenance windows. Similarly, config changes are applied with minimal impact using the same one-node-at-a-time strategy. The result is higher uptime and more confidence in making improvements to the cluster continuously.

* **Built-In Automation & Expertise:** Mission Control encodes Cassandra operational best practices. It integrates tools like Reaper and Medusa and uses automation logic proven in DataStax Astra (the cloud service) for on-prem clusters. This “expert in the box” means even smaller teams or less-experienced operators can perform complex tasks (like repairs or multi-DC deploys) correctly. It reduces the risk of human error (e.g., forgetting to run cleanup, or misconfiguring a new node). Regular maintenance tasks can even be scheduled (e.g., weekly repairs, backups) and will run reliably.

* **Improved Observability:** With metrics and logs consolidated, Mission Control makes it easier to **detect and troubleshoot issues**. There’s no need to bolt on an external monitoring stack – though you can integrate one if desired – the essentials are available on day one. Alerts help catch problems early (and in future, one could imagine Mission Control even triggering auto-remediation). This leads to faster resolution times and a more proactive ops posture.

* **Consistency Across Environments:** Mission Control works for clusters on Kubernetes and can even manage traditional environments. If an organization is migrating from legacy to cloud-native, they can use Mission Control for both during the transition. It provides a consistent interface and process for managing Cassandra whether it’s on VMs in a data center or containers in the cloud. This consistency means less retraining and fewer siloed skill sets – a unified approach to Cassandra operations.

In short, DataStax Mission Control + HCD bring cloud-like automation and simplicity to managing Cassandra databases on any infrastructure. Seasoned DBAs benefit by offloading routine work to automation (freeing them to focus on higher-level architecture and optimization), while newer operators benefit by having a safe, guided way to perform complex ops tasks. It’s the culmination of lessons learned from running large-scale Cassandra clusters, now delivered as a tool that any team can use.

**Mission Control + HCD Use Cases Recap:** This solution is particularly valuable for teams that:

* Manage multiple Cassandra/DSE clusters and need a single source of truth and control.
* Are expected to provide Database-as-a-Service internally (self-service provisioning with guardrails).
* Require high availability and cannot afford long maintenance downtime (e.g., financial systems, ecommerce).
* Wish to incorporate real-time analytics or AI (vector search) on their operational data – HCD makes that possible in the same database, and Mission Control makes it operationally feasible.
* Are transitioning to Kubernetes for stateful workloads and want to bring Cassandra into that fold with confidence.

## Next Steps and Further Exploration

Your hands-on journey doesn’t have to stop here. Here are some recommendations to continue learning and to apply what you’ve learned:

* **Experiment with Different Topologies:** Try creating additional clusters with Mission Control. For example, deploy a multi-datacenter cluster (two DCs with 2 nodes each) and see how Mission Control handles it. Practice decommissioning a node or resizing a cluster down (removing nodes) to observe how data is rebalanced and how Mission Control manages the process.

* **Explore DataStax HCD Features:** Use the Data APIs (REST/GraphQL) more deeply – for instance, create tables and data via the Stargate APIs, or try the vector search feature if you have use for it. You could also connect an app using a DataStax driver to the HCD cluster and observe metrics in Mission Control as you run queries.

* **Integrate with CI/CD and GitOps:** Since everything in Mission Control is backed by Kubernetes manifests, consider a GitOps approach. You can export a cluster definition YAML and put it under version control. Try modifying it and applying via `kubectl` instead of the UI, and watch Mission Control reconcile the changes. This way, you can integrate cluster changes into your CI/CD pipeline for infrastructure (especially if you use Helm or Terraform for your K8s environments).

* **Test Failure Scenarios:** In a non-production environment, simulate failures to build confidence in the automation. For example, kill a Cassandra pod (as we did), but also try scenarios like abruptly deleting a persistent volume to simulate disk loss, and see how Mission Control/Operators handle node replacement. Check how backups and restores work end-to-end by restoring a backup to a new cluster and verifying data integrity.

* **Performance Tuning:** Use Mission Control’s metrics to identify any performance bottlenecks on a larger dataset. You can load more data (perhaps with the DataStax Bulk Loader `dsbulk`) and then use the monitoring dashboards to observe behavior under load. This will give you practice in using Mission Control for capacity planning and performance troubleshooting.

* **Plan for Production:** If you’re considering Mission Control and HCD for production, review the official documentation for sizing guidelines and best practices. Pay attention to network requirements (all nodes must have full connectivity) and security configurations (Mission Control can manage TLS, authentication, etc.). Perhaps set up a pilot cluster in a staging environment managed by Mission Control and run your application’s integration tests on it.

* **Stay Updated:** DataStax is actively developing Mission Control and HCD. New features (like improved UI, more automation, etc.) come regularly. Keep an eye on the release notes and upgrade your Mission Control installation as needed. Also, join the DataStax community forums or Slack – many practitioners share tips and you can ask questions about specific scenarios.

Finally, refer to these resources for more information and guidance:

* **DataStax Mission Control Documentation** – Official docs for installation, configuration, and operations: [docs.datastax.com/en/mission-control/](https://docs.datastax.com/en/mission-control/)
* **DataStax Hyper-Converged Database (HCD)** – Product page and overview of HCD’s capabilities: [datastax.com/astra/databases/hyperconverged](https://www.datastax.com/astra/databases/hyperconverged)
* **GitHub Repos & Examples** – The [mission-control-local-setup](https://github.com/difli/mission-control-local-setup) repo we used and DataStax examples on GitHub provide scripts and configuration samples that can accelerate your learning.
* **DataStax Academy & Tutorials** – Keep an eye out for webinars or videos (DataStax has a YouTube playlist for Mission Control and HCD) to see live demos and deep dives into specific features.

By mastering Mission Control and HCD, you are essentially equipping yourself to run Cassandra with the same operational finesse as a cloud service. We encourage you to apply these skills by gradually introducing Mission Control into your workflows – maybe start managing a dev cluster with it, then a staging, and eventually production. You’ll likely find that the consistency and automation free up time to focus on optimizing data models and performance rather than firefighting routine ops issues.

Thank you for participating in the workshop! We hope this reference guide will serve you well as you continue your Cassandra and HCD journey. Happy clustering!
