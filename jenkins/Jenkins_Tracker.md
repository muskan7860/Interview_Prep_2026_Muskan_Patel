# Jenkins Interview Preparation — Complete Topic Tracker
## Muskan Patel | Target: Mid-July 2026

---

## How to Use This File
- ✅ = Completed (files created + lab done)
- 🔄 = In Progress
- ⏳ = Remaining
- ⚡ = Extra topic from coach material

---

## Day 1 — Pipeline Types ✅

| Status | Topic | File Name |
|--------|-------|-----------|
| ✅ | Declarative vs Scripted Pipeline | `jenkins_day1_01_pipeline_types_theory_labs.md` |
| ✅ | Jenkinsfile in SCM (Pipeline as Code) | `jenkins_day1_01_pipeline_types_theory_labs.md` |
| ✅ | Interview Q&A — Pipeline Types | `jenkins_day1_02_pipeline_types_interview_qa.md` |
| ✅ | Lab Output Explanation (whoami, NODE_NAME) | `jenkins_day1_03_lab_output_explanation.md` |

---

## Day 2 — Multibranch Pipeline & Shared Libraries ✅

| Status | Topic | File Name |
|--------|-------|-----------|
| ✅ | Multibranch Pipeline — auto branch detection | `jenkins_day2_01_multibranch_shared_library_theory_labs.md` |
| ✅ | Shared Libraries — vars/, src/, resources/ | `jenkins_day2_01_multibranch_shared_library_theory_labs.md` |
| ✅ | @Library import, call() method, Elvis operator | `jenkins_day2_01_multibranch_shared_library_theory_labs.md` |
| ✅ | Interview Q&A — Multibranch + Shared Libraries | `jenkins_day2_02_multibranch_shared_library_interview_qa.md` |
| ✅ | Practical Lab (separate easy guide) | `jenkins_day2_03_practical_lab_only.md` |

---

## Day 3 — Stages & Steps ✅

| Status | Topic | File Name |
|--------|-------|-----------|
| ✅ | Checkout (checkout scm) | `jenkins_day3_01_stages_steps_theory_lab.md` |
| ✅ | Build (mvn clean package) | `jenkins_day3_01_stages_steps_theory_lab.md` |
| ✅ | Test (mvn test, junit reporting) | `jenkins_day3_01_stages_steps_theory_lab.md` |
| ✅ | Code Analysis — SonarQube + Quality Gate | `jenkins_day3_01_stages_steps_theory_lab.md` |
| ✅ | Docker Build + Docker Push | `jenkins_day3_01_stages_steps_theory_lab.md` |
| ✅ | Deploy stage + when condition | `jenkins_day3_01_stages_steps_theory_lab.md` |
| ✅ | Post Actions — always, success, failure | `jenkins_day3_01_stages_steps_theory_lab.md` |
| ✅ | cleanWs(), docker rmi cleanup | `jenkins_day3_01_stages_steps_theory_lab.md` |
| ✅ | Interview Q&A — Stages & Steps | `jenkins_day3_02_stages_steps_interview_qa.md` |

---

## Day 4 — Agents & Nodes ✅

| Status | Topic | File Name |
|--------|-------|-----------|
| ✅ | Master-Agent architecture | `jenkins_day4_01_agents_nodes_theory_lab.md` |
| ✅ | agent any, agent none, agent label | `jenkins_day4_01_agents_nodes_theory_lab.md` |
| ✅ | Docker agent (image, args, volume mount) | `jenkins_day4_01_agents_nodes_theory_lab.md` |
| ✅ | Kubernetes agent (pod templates, containers) | `jenkins_day4_01_agents_nodes_theory_lab.md` |
| ✅ | Labels (&&, \|\| logic) | `jenkins_day4_01_agents_nodes_theory_lab.md` |
| ✅ | NODE_NAME, WORKSPACE variables | `jenkins_day4_01_agents_nodes_theory_lab.md` |
| ✅ | Interview Q&A — Agents & Nodes | `jenkins_day4_02_agents_nodes_interview_qa.md` |
| ✅ | Real K8s Agent Setup + Troubleshooting | `jenkins_day4_03_k8s_agent_setup_reference.md` |
| ✅ | Real K8s Agent Hands-On Lab | `jenkins_day4_04_k8s_agent_real_lab.md` |

---

## Day 5 — Credentials ⏳

| Status | Topic | File Name |
|--------|-------|-----------|
| ⏳ | Credentials Binding Plugin | `jenkins_day5_01_credentials_theory_lab.md` |
| ⏳ | withCredentials { } block | `jenkins_day5_01_credentials_theory_lab.md` |
| ⏳ | Secret Text credential type | `jenkins_day5_01_credentials_theory_lab.md` |
| ⏳ | Username/Password credential type | `jenkins_day5_01_credentials_theory_lab.md` |
| ⏳ | SSH Key credential type | `jenkins_day5_01_credentials_theory_lab.md` |
| ⏳ | Never hardcode — why and how | `jenkins_day5_01_credentials_theory_lab.md` |
| ⏳ | environment { } with credentials() helper | `jenkins_day5_01_credentials_theory_lab.md` |
| ⏳ | Interview Q&A — Credentials | `jenkins_day5_02_credentials_interview_qa.md` |

---

## Day 6 — Triggers ⏳

| Status | Topic | File Name |
|--------|-------|-----------|
| ⏳ | Webhook — GitHub/GitLab trigger | `jenkins_day6_01_triggers_theory_lab.md` |
| ⏳ | pollSCM (cron syntax) | `jenkins_day6_01_triggers_theory_lab.md` |
| ⏳ | Cron trigger (scheduled builds) | `jenkins_day6_01_triggers_theory_lab.md` |
| ⏳ | Upstream job trigger | `jenkins_day6_01_triggers_theory_lab.md` |
| ⏳ | Parameterized builds (string, choice, boolean) | `jenkins_day6_01_triggers_theory_lab.md` |
| ⏳ | Interview Q&A — Triggers | `jenkins_day6_02_triggers_interview_qa.md` |

---

## Day 7 — Advanced Topics ⏳

| Status | Topic | File Name |
|--------|-------|-----------|
| ⏳ | Options block (timeout, disableConcurrentBuilds, buildDiscarder) | `jenkins_day7_01_advanced_theory_lab.md` |
| ⏳ | Parallel stages (failFast: true) | `jenkins_day7_01_advanced_theory_lab.md` |
| ⏳ | stash / unstash (share files between agents) | `jenkins_day7_01_advanced_theory_lab.md` |
| ⏳ | input step (manual approval gate) | `jenkins_day7_01_advanced_theory_lab.md` |
| ⏳ | Rollback strategy | `jenkins_day7_01_advanced_theory_lab.md` |
| ⏳ | Interview Q&A — Advanced Topics | `jenkins_day7_02_advanced_interview_qa.md` |

---

## Day 8 — Full End-to-End Pipeline + Mock Interview ⏳

| Status | Topic | File Name |
|--------|-------|-----------|
| ⏳ | Complete real-world Jenkinsfile (all topics combined) | `jenkins_day8_01_full_pipeline.md` |
| ⏳ | Mock Interview — 20 questions with answers | `jenkins_day8_02_mock_interview.md` |

---

## Coach Assignments — After Day 8 ⏳

| Status | Assignment | Description |
|--------|-----------|-------------|
| ⏳ | Assignment 1 | Jenkins installation + first pipeline (Freestyle → Pipeline) |
| ⏳ | Assignment 2 | Python app pipeline — pytest, junit results, archive artifacts |
| ⏳ | Assignment 3 | Multi-branch pipeline with branch-specific deploy behavior |
| ⏳ | Assignment 4 | Docker build + push + container health check pipeline |
| ⏳ | Assignment 5 | Shared library — deployToServer with SSH + health check |
| ⏳ | Assignment 6 | AWS ECS deployment pipeline with ECR push |
| ⏳ | Interview Assignment | Design CI/CD system for 10 microservices (architecture + runbook) |

---

## Extra Topics from Coach Material ⚡

| Status | Topic | Covered In |
|--------|-------|-----------|
| ⚡⏳ | Jenkins security — RBAC, CSRF, audit logs | Day 7 or Day 8 |
| ⚡⏳ | Essential plugins list | Day 8 reference |
| ⚡⏳ | Jenkins CLI commands | Day 8 reference |
| ⚡⏳ | Blue Ocean UI | Day 8 reference |
| ⚡✅ | buildDiscarder, timestamps, ansiColor | Day 7 |
| ⚡⏳ | recordCoverage (Jacoco) | Assignment 2 |
| ⚡⏳ | sshagent step | Assignment 5 |
| ⚡⏳ | withAWS step | Assignment 6 |

---

## Your Real Setup — Quick Reference

| Item | Value |
|------|-------|
| Jenkins URL | https://devopsbymuskan07.com/jenkins/ |
| Jenkins Version | 2.555.2 |
| Running on | Kubernetes (MicroK8s) — namespace: monitoring |
| Agent name | k8s-agent |
| Agent image | jenkins/inbound-agent |
| Agent connection | WebSocket via http://jenkins:8080/jenkins/ |
| Working YAML | ~/jenkins-agent-WORKING.yaml |
| Practice repo | https://github.com/muskan7860/Devops-Practice-lab |

---

## If Agent Disconnects — Run This

```bash
kubectl delete pod jenkins-agent -n monitoring --force
kubectl apply -f ~/jenkins-agent-WORKING.yaml
sleep 20 && kubectl logs jenkins-agent -n monitoring | tail -5
```
Look for: `INFO: Connected`

---

## Progress Summary

| Phase | Status |
|-------|--------|
| Days 1-4 Theory + Labs | ✅ Complete |
| Days 5-8 Theory + Labs | ⏳ Remaining |
| Coach Assignments | ⏳ After Day 8 |
| Mock Interview | ⏳ Day 8 |

**Target completion: Mid-July 2026** 🎯
