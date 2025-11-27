# Deploying-a-Sample-Index.html-with-Azure-Pipelines

# **Guided Lab 01: Deploying a Sample Index.html with Azure Pipelines**

---

## **Why This Lab?**

So far, we've explored how infrastructure is created with Terraform and configured with Ansible. We’ve seen how powerful automation can be when managing systems. But automation doesn't stop with provisioning — we also want to automate how applications and files are **delivered to those systems**.

Imagine this: you have a simple static HTML page. You want it deployed on a real web server — not by logging into the VM and manually copying it — but by pushing a change to your repository and letting a pipeline do the rest.

That’s exactly what you’ll build in this lab.

---

## **Lab Goal**

You’ll create a simple `index.html` page, store it in Azure Repos, and use Azure Pipelines to deploy it to an existing Azure VM running NGINX — using **password-based SSH authentication**.

This lab simulates how CI/CD can automate even the simplest deployments.

---

## **Pre-Requisites**

Before starting, make sure you have:

* An **Azure VM running Ubuntu or Debian**
* **NGINX installed** and running (`sudo systemctl status nginx`)
* Port **80 is open** in the VM’s NSG/firewall
* Azure DevOps project created
* You know the VM’s **public IP**, **username**, and **password**

---

## **Step 1: Set Up the Azure DevOps Repo**

Let’s start by creating a new repository and adding a static HTML file.

1. In Azure DevOps, go to **Repos → Files → New Repository**
2. Name it something like `html-deploy-lab`
3. Create a new file called `index.html` with the following content:

```html
<!DOCTYPE html>
<html>
<head><title>Azure DevOps Deployed Page</title></head>
<body>
  <h1>This page was deployed via Azure Pipeline to a real VM!</h1>
</body>
</html>
```

4. Commit the file to the `main` branch.

This is the file we’ll deliver to the web server.

---

## **Step 2: Create an SSH Service Connection (Password-Based)**

To allow Azure DevOps to copy files to your VM, you need a service connection.

1. Go to **Project Settings → Service Connections**

2. Click **New service connection → SSH**

3. Fill in the following:

   * **Host name**: Your VM’s public IP
   * **Port**: 22
   * **Username**: e.g., `azureuser`
   * **Password**: Your VM's SSH password
   * **Service connection name**: `ssh-to-nginx-vm`

4. Save the service connection.

You now have a secure way for Azure DevOps to access your VM.

---

## **Step 3: Define the Azure Pipeline**

Now let’s automate the deployment using YAML.

1. Go to **Pipelines → Create Pipeline**
2. Select your repo
3. Choose **YAML**, then select **“Starter Pipleline”**
4. Name your pipeline `azure-pipelines.yml` and paste the following:

```yaml
trigger:
  - main

pool:
  name: "self-hosted-agent-pool"

variables:
  sshService: 'ssh-to-nginx-vm'
  webRoot: '/var/www/html'

stages:
  - stage: DeployHTML
    displayName: 'Deploy HTML Page to Azure VM'
    jobs:
      - job: Deploy
        displayName: 'Copy index.html to NGINX web root'
        steps:
          - checkout: self

          - task: SSH@0
            inputs:
              sshEndpoint: '$(sshService)'
              runOptions: 'inline'
              inline: |
                sudo usermod -aG www-data $(whoami)
                sudo chown -R www-data:www-data /var/www/html
                sudo chmod -R 775 /var/www/html
            displayName: 'Set secure permissions for web root'

          - task: CopyFilesOverSSH@0
            inputs:
              sshEndpoint: '$(sshService)'
              sourceFolder: '$(Build.SourcesDirectory)'
              contents: 'index.html'
              targetFolder: '$(webRoot)'
            displayName: 'Copy index.html to VM'

          - task: SSH@0
            inputs:
              sshEndpoint: '$(sshService)'
              runOptions: 'inline'
              inline: |
                echo "Restarting NGINX..."
                sudo systemctl restart nginx
            displayName: 'Restart NGINX on VM'
```

This pipeline does exactly what we want:

* Watches the `main` branch
* Copies `index.html` to the VM’s NGINX root directory
* Restarts NGINX so the new page is served

---

## **Step 4: Run the Pipeline**

Let’s test it.

1. Commit the YAML file and push to `main`
2. Azure DevOps will trigger the pipeline
3. Watch the output under **Pipelines → Runs**

When it finishes, your `index.html` is live.

---

## **Step 5: Verify the Deployment**

In your browser, visit:

```
http://<your-vm-public-ip>
```

You should see the page you created in `index.html`. That’s CI/CD in action — without touching the VM manually.

---
