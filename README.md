# gcp-cicd-gke

Here's a step-by-step tutorial for **Automating CI/CD pipelines using Google Cloud Build and GKE (Google Kubernetes Engine)**:

---

### **1. Prerequisites**
- **Google Cloud Platform (GCP) Account**.
- Install the **gcloud CLI**.
- Enable necessary APIs:
  ```bash
  gcloud services enable container.googleapis.com cloudbuild.googleapis.com
  ```
- A Kubernetes cluster on GKE.
- A GitHub repository with your application code.

---

### **2. Setting Up the Environment**
1. Authenticate the Google Cloud CLI:
   ```bash
   gcloud auth login
   ```
2. Set the active project:
   ```bash
   gcloud config set project [PROJECT_ID]
   ```
3. Set up your GKE cluster:
   ```bash
   gcloud container clusters create [CLUSTER_NAME] --zone [ZONE]
   gcloud container clusters get-credentials [CLUSTER_NAME] --zone [ZONE]
   ```

---

### **3. Application and Docker Setup**
1. **Create Application Code**:
   Create a simple Node.js app as an example:

   - `app.js`:
     ```javascript
     const express = require('express');
     const app = express();
     const PORT = process.env.PORT || 8080;

     app.get('/', (req, res) => {
       res.send('Hello, CI/CD with GKE!');
     });

     app.listen(PORT, () => console.log(`App running on port ${PORT}`));
     ```

   - `package.json`:
     ```json
     {
       "name": "gke-cicd-app",
       "version": "1.0.0",
       "main": "app.js",
       "dependencies": {
         "express": "^4.18.2"
       }
     }
     ```

2. **Create Dockerfile**:
   ```dockerfile
   FROM node:18-slim

   # Set working directory
   WORKDIR /usr/src/app

   # Install dependencies
   COPY package*.json ./
   RUN npm install

   # Copy application files
   COPY . .

   # Expose port
   EXPOSE 8080

   # Start the app
   CMD ["node", "app.js"]
   ```

3. Build the Docker image locally to verify:
   ```bash
   docker build -t gke-cicd-app .
   docker run -p 8080:8080 gke-cicd-app
   ```

---

### **4. Push to Google Container Registry (GCR)**
1. Tag the Docker image:
   ```bash
   docker tag gke-cicd-app gcr.io/[PROJECT_ID]/gke-cicd-app
   ```
2. Push the image to GCR:
   ```bash
   docker push gcr.io/[PROJECT_ID]/gke-cicd-app
   ```

---

### **5. Deploy to GKE**
1. Create a Kubernetes deployment and service YAML file:

   - `deployment.yaml`:
     ```yaml
     apiVersion: apps/v1
     kind: Deployment
     metadata:
       name: gke-cicd-app
     spec:
       replicas: 3
       selector:
         matchLabels:
           app: gke-cicd-app
       template:
         metadata:
           labels:
             app: gke-cicd-app
         spec:
           containers:
           - name: gke-cicd-app
             image: gcr.io/[PROJECT_ID]/gke-cicd-app
             ports:
             - containerPort: 8080
     ---
     apiVersion: v1
     kind: Service
     metadata:
       name: gke-cicd-service
     spec:
       type: LoadBalancer
       selector:
         app: gke-cicd-app
       ports:
       - protocol: TCP
         port: 80
         targetPort: 8080
     ```

2. Deploy the app:
   ```bash
   kubectl apply -f deployment.yaml
   ```

3. Verify the deployment:
   ```bash
   kubectl get pods
   kubectl get service gke-cicd-service
   ```

   Access the application using the external IP of the service.

---

### **6. Automate CI/CD with Google Cloud Build**
1. **Create a Cloud Build Configuration File**:
   - `cloudbuild.yaml`:
     ```yaml
     steps:
     - name: 'gcr.io/cloud-builders/docker'
       args: ['build', '-t', 'gcr.io/$PROJECT_ID/gke-cicd-app', '.']
     - name: 'gcr.io/cloud-builders/docker'
       args: ['push', 'gcr.io/$PROJECT_ID/gke-cicd-app']
     - name: 'gcr.io/cloud-builders/kubectl'
       args:
       - 'apply'
       - '-f'
       - 'deployment.yaml'
     - name: 'gcr.io/cloud-builders/kubectl'
       args:
       - 'set'
       - 'image'
       - 'deployment/gke-cicd-app'
       - 'gke-cicd-app=gcr.io/$PROJECT_ID/gke-cicd-app:$BUILD_ID'
     ```

2. **Trigger Build Automatically on Git Push**:
   - Connect your GitHub repo to Cloud Build:
     ```bash
     gcloud builds triggers create github \
         --repo-name=[REPO_NAME] \
         --repo-owner=[OWNER] \
         --branch-pattern="^main$" \
         --build-config=cloudbuild.yaml
     ```

3. Push your code to the main branch:
   ```bash
   git add .
   git commit -m "Add CI/CD pipeline"
   git push origin main
   ```

4. Cloud Build will:
   - Build the Docker image.
   - Push it to GCR.
   - Deploy or update the Kubernetes deployment.

---

### **7. Verify the Pipeline**
- Check build logs:
  ```bash
  gcloud builds list
  gcloud builds log [BUILD_ID]
  ```
- Confirm the deployment is updated:
  ```bash
  kubectl get pods
  kubectl describe deployment gke-cicd-app
  ```

---

### **8. Clean Up**
- Delete the GKE cluster:
  ```bash
  gcloud container clusters delete [CLUSTER_NAME] --zone [ZONE]
  ```
- Delete the Docker image from GCR:
  ```bash
  gcloud container images delete gcr.io/[PROJECT_ID]/gke-cicd-app
  ```

---

This pipeline ensures automated builds and deployments to GKE upon every code change. Let me know if you need help enhancing it with additional steps like testing or notifications!
