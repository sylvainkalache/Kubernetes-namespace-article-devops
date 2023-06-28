# Overcoming Kubernetes Namespace Limitations
As companies are standardizing on Kubernetes and moving more of their workloads to the platform, the need emerges for resource isolation and, more generally, multi-tenancy features. Kubernetes Namespaces are the tool of choice to achieve that. The  [CNCF Survey Report 2020](https://www.cncf.io/wp-content/uploads/2020/11/CNCF_Survey_Report_2020.pdf)  found that 83% of companies used namespaces to separate Kubernetes applications.

This article will explore what  [Kubernetes](https://cloudnativenow.com/topics/cloudnativedevelopment/5-kubernetes-concepts-you-must-know-hpa-api-gateway-opentelemetry-and-more/)  namespaces are and go through classic multi-tenancy scenarios. It will explore some limitations and how they can be solved using hierarchical namespaces or Cloud Foundry Korifi.
### What Are Namespaces?

Namespaces provide a logical partitioning for managing Kubernetes resource allocation, access control, and configuration management, allowing developers to manage multiple applications and environments within the same Kubernetes cluster.

This is beneficial in many use cases, such as when running unrelated applications in different namespaces or when running different versions (such as dev, test, and production) of the same application in a single Kubernetes cluster.

For example, a company could have two applications:

App1: An e-commerce website maintained by Team1  
App2: A backend to manage orders and customers maintained by Team2

Both applications need to be deployed in the same Kubernetes cluster for cost optimization. Let’s look at how we can create a basic namespace setup. Before we start: for each step, a screenshot of the commands is provided as well as a link to a GitHub file for easy copy/pasting.  [Here is a link to the Github repo](https://github.com/sylvainkalache/Kubernetes-namespace-article-devops).

The first step is to create namespaces for each application by running these [kubectl commands](https://github.com/sylvainkalache/Kubernetes-namespace-article-devops/blob/main/create-namespaces.sh).
```[[create-namespaces.sh](create-namespaces.sh)](https://github.com/sylvainkalache/Kubernetes-namespace-article-devops/blob/main/create-namespaces.sh)```
Then, we can configure access control and resource quotas for each namespace. Role-Based Access Control (RBAC) policies can limit which users or service accounts can access or manage resources in each namespace. In the example below, we will define a resource quota to restrict the amount of CPU each app can use.

For App1, we will limit the CPU usage to 80% of the cluster. Therefore, we create a  [app1-resource-quota.yaml](https://github.com/sylvainkalache/Kubernetes-namespace-article-devops/blob/main/app1-resource-quota.yaml)  file containing the following.
[app1-resource-quota.yaml](app1-resource-quota.yaml)
For App2, we will limit the CPU usage to 10% of the cluster. We create a [app2-resource-quota.yaml](https://github.com/sylvainkalache/Kubernetes-namespace-article-devops/blob/main/app2-resource-quota.yaml) file containing the following.
[app2-resource-quota.yaml](app2-resource-quota.yaml)
To apply the resource quotas, we need to run the  [following commands](https://github.com/sylvainkalache/Kubernetes-namespace-article-devops/blob/main/apply-quotas.sh).
[apply-quotas.sh](apply-quotas.sh)
We confirm that the quotas have been applied with the  [following commands](https://github.com/sylvainkalache/Kubernetes-namespace-article-devops/blob/main/confirm-quotas.sh).
[confirm-quotas.sh](confirm-quotas.sh)
App1 and App2 are respectively managed by Team1 and Team2. Let’s give them the appropriate deployment rights.

First, we will create a Role for each namespace that allows managing deployments. Let’s start with App1 by creating the  [_app1-role.yaml_](https://github.com/sylvainkalache/Kubernetes-namespace-article-devops/blob/main/app1-resource-quota.yaml) file.
[app1-resource-quota.yaml](app1-resource-quota.yaml)
And proceed with the same for App2 with the file [_app2-role.yaml_](https://github.com/sylvainkalache/Kubernetes-namespace-article-devops/blob/main/app2-role.yaml).
[app2-role.yaml](app2-role.yaml)
The next step is to create RoleBinding resources to associate each role with the respective team.

We start with the binding between App1 and Team1 with the file  [_app1-rolebinding.yaml_](https://github.com/sylvainkalache/Kubernetes-namespace-article-devops/blob/main/app1-rolebinding.yaml).
[app1-rolebinding.yaml](app1-rolebinding.yaml)
Now, let’s create the binding between App2 and Team2 with the file  [_app2-rolebinding.yaml_](https://github.com/sylvainkalache/Kubernetes-namespace-article-devops/blob/main/app2-rolebinding.yaml).
[app2-rolebinding.yaml](app2-rolebinding.yaml)
We apply the RBAC configurations using these [kubectl commands](https://github.com/sylvainkalache/Kubernetes-namespace-article-devops/blob/main/apply-roles-configuration.sh).
[apply-roles-configuration.sh](apply-roles-configuration.sh)
Those are straightforward examples, and Kubernetes namespaces have countless more possibilities. However, namespaces have limitations that can impact their flexibility in certain use cases.

For instance, when a team owns multiple microservices with distinct secrets and quotas, placing them in separate namespaces for isolation can lead to issues. Kubernetes lacked a common ownership concept for these namespaces, making it difficult to apply namespace-scoped policies uniformly across them.

![](https://cloudnativenow.com/wp-content/uploads/2023/05/11.png)

Another problematic situation is that teams usually perform better when operating autonomously, but creating namespaces is a highly-privileged Kubernetes operation. As a result, developers must request a new namespace from the cluster administrator, which can create unnecessary administrative work, especially in larger organizations.

### Hierarchical Namespaces

Kubernetes Hierarchical Namespace Controller ([HNC](https://github.com/kubernetes-sigs/hierarchical-namespaces)) addresses these issues. A hierarchical namespace functions similarly to a standard Kubernetes namespace but includes a small custom resource that specifies an optional parent namespace. This introduces the notion of ownership across multiple namespaces, extending beyond the scope of individual namespaces.

This concept of ownership enables two additional types of behaviors:

-   **Policy inheritance:**  Policy objects such as RBAC RoleBindings are  [copied from the parent namespace to the child](https://github.com/kubernetes-sigs/multi-tenancy/blob/master/incubator/hnc/docs/user-guide/concepts.md%23basic-propagation)  one.
-   **Delegated creation:**  [subnamespaces](https://github.com/kubernetes-sigs/multi-tenancy/blob/master/incubator/hnc/docs/user-guide/concepts.md%23basic-subns), which can be manipulated using only limited permissions in the parent namespace without the need for cluster-level privileges.

This solves both of the problems for dev teams. The Kubernetes cluster administrators can create a single “root” namespace for the entire organization, along with all necessary policies, and then delegate permission to create subnamespaces to members, for example, one subnamespace for both App1 and App2. Team members can then create subnamespaces for their own use, without violating the policies that the cluster administrators imposed.

[![](https://cloudnativenow.com/wp-content/uploads/2023/05/12.png)](https://cloudnativenow.com/wp-content/uploads/2023/05/12.png)

Note that HNC is an optional extension not shipped with Kubernetes by default. To install HNC, you can follow the instructions provided in the  [HNC GitHub repository.](https://github.com/kubernetes-sigs/hierarchical-namespaces)

### The Cloud Foundry Alternative

The Cloud Foundry community recently launched a platform that solves these same concerns. Open source  [Korifi](https://www.cloudfoundry.org/technology/korifi/)  provides a modern cloud-native application delivery and management model for Kubernetes. And when it comes to handling multi-tenancy, the project goal is to mimic the same  [Cloud Foundry RBAC](https://docs.cloudfoundry.org/concepts/roles.html)  syntax that grants permissions to Cloud Foundry users, but for Kubernetes clusters.

[![](https://cloudnativenow.com/wp-content/uploads/2023/05/13.png)](https://cloudnativenow.com/wp-content/uploads/2023/05/13.png)

Therefore, companies familiar with the way Cloud Foundry Orgs, Spaces, Roles, and Permissions will not need to learn anything new. Similarly to Hierarchical Namespaces, Korifi also allows giving Kubernetes permissions to one user over an entire Org and all the Spaces it contains. The installation can be easily done by following the instructions in the  [Korifi GitHub repository](https://github.com/cloudfoundry/korifi).

[![](https://cloudnativenow.com/wp-content/uploads/2023/05/14.png)](https://cloudnativenow.com/wp-content/uploads/2023/05/14.png)

### What to Keep in Mind

There is a classic misconception about Kubernetes namespaces. While they provide logical isolation at the API level, they do not inherently provide network isolation between namespaces. Network isolation between namespaces can be achieved using Network Policies, which are a separate Kubernetes feature. So be sure to keep this in mind when setting them up.

This article was originally published on Cloud Native Now https://cloudnativenow.com/features/overcoming-kubernetes-namespace-limitations/.
