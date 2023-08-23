- [1. Intro](#1-intro)
  - [Pre-requisites](#pre-requisites)
  - [On grafana/prometheus](#on-grafanaprometheus)
- [2. Getting up and running](#2-getting-up-and-running)
  - [Steps to start the monitoring apps](#steps-to-start-the-monitoring-apps)
  - [Steps to start the demo app](#steps-to-start-the-demo-app)
- [3. DevOps Challenge - Questions](#3-devops-challenge---questions)
- [4. Extra Questions](#4-extra-questions)
  - [How do i put applications behind ingress controller, and what are ingress controllers used for?](#how-do-i-put-applications-behind-ingress-controller-and-what-are-ingress-controllers-used-for)
    - [what are ingress controller used for:](#what-are-ingress-controller-used-for)
    - [How do you put applications behind ingress controller.](#how-do-you-put-applications-behind-ingress-controller)
  - [Options for storing sensitive data such as secrets](#options-for-storing-sensitive-data-such-as-secrets)
  - [Completely remove data worth protecting that was mistakenly stored in plain text](#completely-remove-data-worth-protecting-that-was-mistakenly-stored-in-plain-text)

## 1. Intro
### Pre-requisites

- minikube or an alternate local kubernetes cluster
- have kubectl configured to connect to this cluster and the default namespace
- have helm installed (i.e `brew install helm`)

### On grafana/prometheus

  I will be using  the chart kube-prometheus-stack from prometheus's helm charts. Its a collection of Kubernetes manifests,  dashboards, and  combined with documentation and scripts to provide easy to operate end-to-end Kubernetes cluster monitoring with Prometheus using the Prometheus Operator.

## 2. Getting up and running

### Steps to start the monitoring apps

1. Install kube-prometheus-stack

    ```
    helm install prometheus grafana-charts/charts/kube-prometheus-stack 
    ```
    

2. Confirm pods are running by running `kubectl get pods`

   You should be seeing something similar to this:

  ![grafana-prometheus-running](docs/grafana-prometheus-running.png)


3. To Login to grafana we must first expose the port 3000 using the following command 

    ```
    kubectl port-forward prometheus-grafana-8b6684dc8-pdkmp 3000:3000
    ```

    Now u can visit [localhost:3000](localhost:3000) and use the following credentials to login:

    - username: admin
    - password: prom-operator

4. Confirm we have prometheus and alertmanager configured by visiting `Connections > Data sources`

![](/docs/connections-data-sources.png)

### Steps to start the demo app

1. Install kuard-chart

```
helm install kuard-release kuard-chart    
```

2. Confirm kuard pod is running by executing `kubectl get pods`

![kuard-running](docs/kuard-running.png)

3. Expose the pod running kuad with the following command:

```
kubectl port-forward kuard-74675c74d6-fdm48 8080:8080
```

visiting [localhost:8080](localhost:8080) shoudl show you the following screen:

![](docs/kuard-app-running.png)

4. showing prometheus data for job kuad in grafana

![](docs/job-kuad-grafana.png)

## 3. DevOps Challenge - Questions

1. **Logging solution in Kubernetes** - Loki is grafana's solution for centralisation of logs. 

    To install loki, the best approach would be to install the loki-stack from grafana's helm charts

    ```
    helm install loki grafana/loki-stack --set "grafana.enabled=false,prometheus.enabled=false"    
    ```

    Notice how we don't need to enable grafana nor prometheus as its already installed via the kube-prometheus-stack (from the monitoring stack).

    Loki allows you to fine tune your selection of logs. From a specific node, namespace, container or even a specific pod
    
    ![](docs/loki-categories.png)

    An example of the Kuard logs would look something like the following:

    ![](docs/loki-kuard-logs.png)

2. **Kubernetes board resources ensure application can cope with a high number of requests.**

    Autoscaling should be enabled in the values.yaml, if the CPU threshold is reached (80%) then it will fire up a new pod allowing the app to cope with a higher number of requests

3. **Detect templating error before a deployment**

    In the CI/CD pipeline, before executing the deployment we would have a step to run the `helm lint` command in each of the charts    

4. **Resetting failed deployment**
    
    To rollback to the previous release

    ```
    helm rollback
    ```

    To rollback to the a specific release

    ```
    helm rollback <RELEASE>
    ```

5. **What metrics does the demo app offer**

   demo app uses the go_lang client prometheus, so we have typical go related stats:
   - go allocatable memory
   - go used memory
   - full list of metrics available [here](/docs/metrics.txt)

6. **Which CI/CD tool would be my tool of choice**

    Either Github Actions or Gitlab CI/CD (depends where the repo is hosted)

## 4. Extra Questions 

### How do i put applications behind ingress controller, and what are ingress controllers used for?

#### what are ingress controller used for:

Ingress controllers are used to manage external access to services within a Kubernetes cluster. They provide a way to expose services to the outside world without exposing the underlying pod IPs. 

#### How do you put applications behind ingress controller.

To put applications behind an Ingress controller, you follow these steps:

1. Install ingress controller
An Ingress controller is a Kubernetes resource that manages external access to services within your cluster. You need to install an Ingress controller, such as Nginx Ingress Controller, Traefik, or HAProxy Ingress.
2. Create an Ingress Resource:
Define an Ingress resource that specifies the rules for routing external traffic to your services. This includes hostnames, paths, and the associated service to forward traffic to.
3. Deploy Your Applications:
Deploy your applications as Kubernetes services and ensure they are correctly labeled. These labels will be used to match the services with the Ingress rules.
4. Configure Ingress Rules:
In your Ingress resource, set up rules that define how traffic should be directed to your services. You can configure path-based routing, SSL termination, and more.
5. Configure DNS:
Update your DNS records to point to the IP or hostname associated with your Ingress controller's load balancer or service.
6. Access Applications:
After DNS propagation, your applications should be accessible through the configured routes defined in the Ingress resource.

### Options for storing sensitive data such as secrets

1. Storing secrets as repository variables in Github (Or Gitlab), having a find and replace step that switches the value placeholder for the value from github just before running the helm update / kubectl apply. 
2. Using the Secret Store CSI Driver (with which ever flavour is preffered: AWS Secret store, Hashicop Vault). AWS Secret Store allows you to easily rotate, manage, and retrieve database credentials, API keys, certificates, and other secrets throughout their lifecycle. The  Secret Store CSI Driver allows secrets stored in manager to appear as files in Kubernetes pods, these can be mounted as enviroment variables.

I would go with option 2 as it allows all secrets to be centralised in the AWS Secret Store Console, and provides a cleaner approach, and this way you dont have secrets lingering in the repository's config section.

### Completely remove data worth protecting that was mistakenly stored in plain text

1. BFG Repo Cleaner - [link](https://rtyley.github.io/bfg-repo-cleaner/)

```
bfg --delete-files YOUR-FILE-WITH-SENSITIVE-DATA
git push origin --force 
```

2. Using git filter-repo

```sh
#install
brew install git-filter-repo

git filter-repo --invert-paths --path PATH-TO-YOUR-FILE-WITH-SENSITIVE-DATA
echo "YOUR-FILE-WITH-SENSITIVE-DATA" >> .gitignore
git add .gitignore
git push origin --force --all
```