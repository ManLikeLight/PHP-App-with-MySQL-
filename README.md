# PHP App with MySQL, Multi-Stage Jenkins Pipeline & Manual Promotion

## 🎯 Project Goal

Deploying a **PHP CRUD Application** connected to a **MySQL Database** using a manual deployment **CI/CD Jenkins Pipeline**. This includes:

- Deployment to **Staging**.
- **Manual promotion** to **Production**.
- Deployment artifacts stored in **AWS S3**.
- App is publicly accessible over the internet via **port 80**.

---

## 🧰 Requirements

### Infrastructure

| Component       | OS       | Description                          |
|----------------|----------|--------------------------------------|
| Jenkins Master | CentOS   | Controls pipeline execution          |
| Jenkins Agent  | Ubuntu   | Performs builds, deployments, etc.   |
| App Servers    | Ubuntu   | Hosts PHP app + MySQL DB    
| Sonar          | AWS

---

## 📦 Ansible Roles

The Jenkins agent will use **Ansible** to automate:

1. **Apache + PHP Installation**
2. **MySQL Setup**
   - Create users
   - Create schema
   - Insert test data
3. **Deploy PHP App**
   - Clone from Git
   - Deploy to web root
   - Restart Apache

---

## 🎯 Jenkins Pipeline

A Jenkins `Declarative Pipeline` with **multi-stage steps**:

### 🔨 Build Stage
- Clone the PHP app repo from GitHub
- Run PHP sonarqube test
- Archive the app as a `.zip` or `.tar.gz`
- Upload artifact to S3

### 🚀 Staging Deployment
- Deploy the archived artifact to the **staging** server

### ✋ Manual Approval
- Pause pipeline
- Wait for manual input before promoting to production

### 🔁 Production Deployment
- Select version (using Jenkins parameter)
- Deploy artifact to production server
- Make app accessible on port 80

---

