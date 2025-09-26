oc cmd
 
 
 Rolling updates --> while running the multiple instances of the service it ensures that one pod is updating at a time and keep at least one pod running to serve the user 

S2I


To set up a CI/CD pipeline on OpenShift for a microservices-based application, I'd start by integrating a tool like Jenkins or Tekton for automation. I'd configure the pipeline to automatically build Docker images for each microservice whenever code is pushed to the main branch. Then I'd use OpenShift's native capabilities—like deploying these images into the cluster, managing environment-specific configurations through ConfigMaps and Secrets, and performing automated rollouts with health checks and rollback strategies if something goes wrong.

The idea is to have a seamless flow: code changes trigger builds, builds create container images, and OpenShift handles the deployment. That way, every microservice is consistently deployed and updated without manual intervention.

---

**"Performing automated rollouts with health checks and rollback strategies if something goes wrong"** means:

✅ **Automated Rollouts:**  
When a new version of your app is deployed (e.g., via Jenkins pipeline), OpenShift or Kubernetes gradually replaces old pods with new ones — typically using a **rolling update** strategy. This ensures traffic continues flowing during the update.

✅ **Health Checks:**  
OpenShift uses two probes to check your app's health:

- **Readiness probe:** checks if your app is ready to receive traffic.
    
- **Liveness probe:** checks if your app is still alive or stuck.
    

If a new pod fails these checks, the rollout **pauses**.

✅ **Rollback Strategy:**  
If the new version has problems (e.g., fails health checks or increases error rate), OpenShift can automatically or manually **rollback** to the last known good version — ensuring minimal downtime.

---

### Interview-Ready Example:

> "In our CI/CD pipeline, we trigger automated rollouts via OpenShift whenever a new image is built and pushed. OpenShift performs rolling updates while monitoring app health through readiness and liveness probes. If the new version fails any probe or the error rate spikes, the rollout can be paused and automatically rolled back to the previous stable version — all without affecting end users."

Let me know if you want a diagram or want to rehearse this answer.