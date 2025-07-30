# ERPNext CI/CD Setup

This repository contains a lightweight CI/CD setup for validating pull requests to a custom ERPNext fork.

It uses Docker Compose to spin up a minimal Frappe/ERPNext environment, mount local application code, and run basic checks.

---

## 🔧 What’s Inside

- **`docker-compose.yml`**  
  A minimal ERPNext Docker setup, optimized for fast local runs inside CI.  
  Includes only essential services like:
  - `backend`
  - `create-site`
  - `db` (MariaDB)
  - `redis-cache` / `redis-queue`

- **`Jenkinsfile`**  
  Jenkins pipeline script to:
  1. Clone your ERPNext fork and checkout a pull request branch.
  2. Mount it into the Docker container via bind mounts.
  3. Spin up the environment using `docker-compose`.
  4. Create a test site using `create-site`.
  5. Run automated checks.
  6. Clean up the environment after the job.

---

## 📁 Project Structure

```bash
erpnext-ci/
├── docker-compose.yml        # lightweight infrastructure
├── Jenkinsfile               # pipeline logic
└── README.md                 # this file
