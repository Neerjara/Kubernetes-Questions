# Kubernetes-Questions

1. Why do we use the Jenkins Kubernetes plugin instead of static Jenkins agents?
Earlier, we used static Jenkins agents, which were always running. The drawback was high cost, since those VMs consumed resources even when no jobs were running, and limited scalability, because one agent could only handle a fixed number of jobs.
With the Jenkins Kubernetes plugin, we get ephemeral agents — pods are created on demand when a job starts and destroyed once it completes. This approach gives us:
Cost efficiency → resources are only used while jobs run.
Scalability → Kubernetes can spin up multiple pods in parallel to handle peak workloads.
Consistency & security → each job runs in a clean, isolated environment with the exact dependencies defined in the pod template.
So overall, the plugin enables us to run Jenkins in a more efficient, scalable, and secure way compared to static agents.

2. Can you explain how you configured Jenkins to connect to Kubernetes?
   To connect Jenkins with Kubernetes, I used the Jenkins Kubernetes plugin.
First, I installed the plugin from Manage Jenkins → Plugins.
Then, under Manage Jenkins → Clouds, I added a new cloud and selected Kubernetes as the type.
In the configuration, I provided:
The Kubernetes API server URL (cluster endpoint).
Credentials → usually a service account from Kubernetes or a kubeconfig file.
The namespace where Jenkins should create pods.
After that, I defined Pod Templates with labels like k8s-agent, where I specified containers (Maven, Docker, Python, etc.) and resource requests/limits.
Finally, in the Jenkinsfile, I referenced that pod label. When a job ran, Jenkins connected to Kubernetes, launched a pod in that namespace, executed the job, and then terminated the pod automatically.
To make the integration secure, I created a dedicated service account with limited RBAC permissions in Kubernetes, instead of giving cluster-admin rights. This ensures Jenkins can only create and delete pods in its namespace.


4. How do you define a Pod Template in Jenkins?
   In Jenkins, a Pod Template defines what kind of Kubernetes pod will be spun up as an agent for running jobs.
To define one:
Go to Manage Jenkins → Clouds → Kubernetes → Add Pod Template.
Provide a label (e.g., k8s-agent) → this label is used inside the Jenkinsfile.
Add one or more containers inside the template:
For example, maven container with the Maven image,
docker container with Docker CLI,
python container for Python jobs.
For each container, I can specify:
Name (e.g., maven)
Docker image (e.g., maven:3.8.6-jdk-11)
Command/Args if I want to override defaults
Resource requests/limits (CPU, memory)
Volume mounts (if needed, e.g., Docker socket, cache)
I can also configure pod-level settings like namespace, idle timeout, and whether the pod should always be deleted after use.
Finally, in my Jenkinsfile, I simply reference the label, and Jenkins uses that Pod Template to spin up the pod dynamically.


6. How do you ensure resource limits are respected in Jenkins pod templates?
   In Jenkins pod templates, we ensure resource limits are respected by configuring CPU and Memory requests/limits for each container.
Requests → minimum guaranteed resources for the container.
Limits → maximum resources the container can consume.
When I define a Pod Template in Jenkins (through the UI or YAML), I explicitly set these values. For example:
Maven container → request 500m CPU, 1Gi memory, limit 1 CPU, 2Gi memory.
Docker container → request 200m CPU, 512Mi memory, limit 500m CPU, 1Gi memory.
This way:
Kubernetes scheduler ensures the pod only runs on nodes with enough resources.
Containers can’t consume more than their defined limits, preventing resource starvation of other pods.
Builds remain stable and predictable under load.


8. Can you describe how a Jenkinsfile looks when using a Kubernetes pod as the agent?
   pipeline {
    agent {
        kubernetes {
            label 'k8s-agent'
            defaultContainer 'maven'
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: maven
    image: maven:3.8.6-jdk-11
    command:
    - cat
    tty: true
    resources:
      requests:
        cpu: "500m"
        memory: "1Gi"
      limits:
        cpu: "1"
        memory: "2Gi"
  - name: docker
    image: docker:20.10
    command:
    - cat
    tty: true
"""
        }
    }
  
10. What happens to the pod after the Jenkins job finishes?
    When Jenkins uses the Kubernetes plugin, it spins up an ephemeral pod to act as the build agent.
Once the Jenkins job finishes (success or failure), that pod is automatically terminated and cleaned up by Kubernetes.
This ensures:
No leftover build environment → clean, isolated builds every time.
Efficient resource usage → cluster resources are freed immediately.
Security → sensitive data or credentials inside the pod don’t persist


12. How do you handle credentials (e.g., Docker registry, Git) inside Kubernetes agents?
    In our setup, we use CyberArk Conjur as the central secrets manager.
Jenkins doesn’t store long-lived credentials — instead, ephemeral Kubernetes agent pods fetch secrets on demand from Conjur at runtime.
This gives us:
Centralized secret management (one source of truth).
Rotation support (apps pick up new secrets automatically).
Ephemeral use (secrets injected only while the job runs)."*
🔹 How It Works (Flow)
Jenkins integrates with Conjur using the CyberArk Conjur Jenkins Plugin
The Jenkins master or agent pod authenticates with Conjur via:
Kubernetes Authenticator (recommended) → pod uses its ServiceAccount to authenticate.
Or via a host identity (less common now).
Secrets are retrieved just-in-time during the pipeline.
After the pod completes, secrets are gone (ephemeral pod lifecycle).
🔹 Example Jenkinsfile (Conjur Integration)
pipeline {
  agent {
    kubernetes {
      label 'jenkins-agent'
      defaultContainer 'jnlp'
    }
  }
  stages {
    stage('Login to Docker') {
      steps {
        withCredentials([conjurSecretCredential(credentialsId: 'docker-registry-creds')]) {
          sh '''
            docker login -u $CONJUR_USERNAME -p $CONJUR_PASSWORD registry.example.com
          '''
        }
      }
    }
    stage('Clone Git Repo') {
      steps {
        withCredentials([conjurSecretCredential(credentialsId: 'git-ssh-key')]) {
          sh '''
            git clone git@github.com:my-org/my-repo.git
          '''
        }
      }

      
14. What are the benefits of using Kubernetes with Jenkins in terms of scalability and maintenance?
    Using Kubernetes with Jenkins brings scalability and reduced maintenance overhead compared to traditional static Jenkins agents."
🔹 Scalability Benefits
On-demand agents – Jenkins dynamically provisions pods for each build and tears them down after completion. No need to pre-provision fixed agents.
Elastic scaling – Kubernetes can autoscale Jenkins agents based on workload, handling spikes in builds without manual intervention.
Parallel execution – Multiple jobs can run simultaneously without resource conflicts, since each job gets its own isolated pod.
Multi-environment support – Pod templates let us spin agents with different toolchains (Maven, Docker, Python, Node.js) in parallel, avoiding “dependency conflicts.”
🔹 Maintenance Benefits
Reduced VM maintenance – No need to patch or maintain static Jenkins worker nodes; Kubernetes schedules pods automatically.
Ephemeral agents – Build environments are clean every time (no leftover artifacts or config drift).
Declarative Pod Templates – Define build environments as code (in YAML or Jenkinsfile), making upgrades and versioning simple.
Resource optimization – Pods free up resources once the job finishes, unlike idle static VMs.
Built-in fault tolerance – If a node crashes, Kubernetes reschedules Jenkins agent pods automatically.
🔹 Interview Bonus 🚀
"This approach not only scales Jenkins elastically but also ensures maintainability. We no longer manage dozens of long-lived VMs — instead, we define pod templates and let Kubernetes orchestrate clean, ephemeral build agents. This aligns Jenkins with modern DevOps practices


16. What challenges have you faced while using Jenkins with Kubernetes?
    While integrating Jenkins with Kubernetes, I faced several challenges, mainly around stability, configuration, and security."
🔹 Key Challenges & Solutions
Pod Startup Delays
Sometimes builds waited while Kubernetes pulled large images.
Solution: Optimized pod images (multi-stage builds, caching), and used a local registry mirror to speed up pulls.

Debugging Ephemeral Pods
Pods get deleted after job completion → harder to debug.
Solution: Sidecar + S3 Sync (Automation)
Add a sidecar container that continuously syncs logs to S3 during build execution:
- name: log-uploader
  image: amazon/aws-cli:2.0.30
  volumeMounts:
  - name: build-logs
    mountPath: /logs
  command:
  - sh
  - -c
  - |
    while true; do
      aws s3 sync /logs s3://jenkins-debug-logs/$(hostname)/ --delete
      sleep 30
        done
Main jnlp container writes logs to /logs.
Sidecar uploads them to S3 every 30s.
Even if the pod dies unexpectedly, you still have partial logs in S3.

Credentials Management
Needed to inject Docker/Git credentials securely.
Solution: Integrated Jenkins with Conjur / Kubernetes Secrets and injected them at runtime via pod templates.

Resource Management
Builds sometimes consumed too much CPU/memory.
Solution: Set proper requests/limits in pod templates, and monitored usage via Prometheus + Grafana.
Networking & DNS Issues
Jenkins agents inside pods couldn’t always reach external repos or registries.
Solution: Configured proper cluster DNS + network policies, and in some cases used sidecar containers for proxies.
Plugin Compatibility
Some Jenkins plugins assumed static agents, not ephemeral pods.
Solution: Adjusted pipeline logic, or switched to Kubernetes-native plugins where possible.
🔹 Interview Bonus 🚀
"These challenges actually helped me strengthen automation — for example, optimizing pod images improved overall build times, and integrating with Conjur improved security. So Kubernetes with Jenkins became not just scalable, but also more secure and maintainable."


