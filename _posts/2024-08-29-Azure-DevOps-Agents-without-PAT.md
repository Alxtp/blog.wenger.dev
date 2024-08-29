---
title: "Simple and Secure: Deploy self-hosted Azure DevOps Agents without PAT"
date: 2024-08-29 21:00:00 +0200
categories: [Azure]
tags: [azure, automation, terraform, iac, azure devops, container, docker]
img_path: /azure/
---

In the world of Azure DevOps, self-hosted agents provide greater control and customization for your CI/CD pipelines. However, traditional deployment methods often rely on personal access tokens (PATs), which can introduce security risks and are bound to a user account. This post explores a modern, secure approach to deploying self-hosted Azure DevOps agents using Azure Container Apps jobs and managed identity.