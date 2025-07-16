# ⚡ PowerGuard

PowerGuard is a robust and intelligent energy monitoring and protection system designed to provide real-time analytics, automated protection mechanisms, and seamless integration with cloud platforms.

![GitHub Workflow Status](https://img.shields.io/github/actions/workflow/status/chandini1727/PowerGuard/main.yml?branch=main)

---

## 🚀 Features

- 🔌 Real-time energy usage tracking
- ⚙️ Automated overload and surge protection
- ☁️ Cloud integration for remote monitoring
- 📊 Dashboard for usage analytics
- 🔒 Secure data logging and alerts

---

## 🛠️ Tech Stack

- **Frontend:** React / Next.js
- **Backend:** Node.js / Express
- **Database:** MongoDB / Firebase
- **DevOps:** GitHub Actions, Docker, AWS (optional)

---

## 🔁 GitHub Actions Workflow

The CI/CD pipeline is powered by **GitHub Actions** to automate tasks such as:

- ✅ Linting and testing
- 🐳 Docker build
- 🚀 Deployment to production/staging
- 🛑 Error reporting on failed builds

### 📄 `.github/workflows/main.yml`

```yml
name: CI/CD Pipeline

on:
  push:
    branches:
      - main
  pull_request:

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: ⬇️ Checkout Code
        uses: actions/checkout@v3

      - name: ⚙️ Set up Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'

      - name: 📦 Install Dependencies
        run: npm ci

      - name: 🧪 Run Tests
        run: npm test

      - name: 🐳 Build Docker Image
        run: docker build -t powerguard-app .

      # Add deployment step here if needed
