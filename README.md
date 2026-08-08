# Kubernetes DevOps Learning Labs

[![Kubernetes](https://img.shields.io/badge/Kubernetes-1.28%2B-326ce5?style=flat-square&logo=kubernetes)](https://kubernetes.io/)
[![Docker](https://img.shields.io/badge/Docker-24.0%2B-2496ed?style=flat-square&logo=docker)](https://www.docker.com/)
[![Status](https://img.shields.io/badge/Status-Active%20Learning-green?style=flat-square)]()
[![License](https://img.shields.io/badge/License-MIT-blue?style=flat-square)]()
[![Phases](https://img.shields.io/badge/Phases-7%2F7-brightgreen?style=flat-square)]()
[![Timeline](https://img.shields.io/badge/Timeline-9%20Weeks-orange?style=flat-square)]()

---

## 🎯 Mission Statement

**Fast-track Kubernetes mastery**: From zero Docker/K8s knowledge to job-ready DevOps engineer in **9 weeks**.  
Self-paced, hands-on labs with production-grade microservices project. Designed for **QA engineers transitioning to DevOps** (but works for anyone).

> This repo demonstrates a **structured, interview-sharp learning methodology**—exactly what DevOps hiring teams want to see.

---

## 📊 At a Glance

| Metric | Details |
|--------|---------|
| **Duration** | 9 weeks (12-14 hours/week) |
| **Phases** | 7 progressive learning phases |
| **Hands-On Labs** | 25+ real-world scenarios |
| **Code Examples** | 50+ Kubernetes manifests |
| **Tech Stack** | Kubernetes, Docker, Jenkins, Terraform, AWS/Azure/GCP |
| **Deployment Targets** | Minikube, Kubeadm, EKS, AKS, GKE |
| **Portfolio Project** | 4-tier microservices application |
| **Interview Prep** | 100+ questions with deep answers |

---

## 🚀 Learning Path (7 Phases)

```
Phase 1: Fundamentals             → Minikube + Kubeadm setup
Phase 2: Core Components          → Pods, Services, Deployments
Phase 3: Networking & Storage     → Ingress, PVC, StatefulSets
Phase 4: Production Patterns      → RBAC, HPA, Health Checks
Phase 5: Microservices Project    → Real 4-tier application
Phase 6: CI/CD Integration        → Jenkins → K8s pipelines
Phase 7: Managed Services         → EKS, AKS, GKE deployment
```

**Timeline**: Week 1 → Fundamentals | Week 2-3 → Core | Week 4 → Networking | Week 5 → Production | Week 6-7 → Project | Week 8 → CI/CD | Week 9 → Managed

---

## 📁 Repository Structure

```
kubernetes-devops-learning-labs/
├── phase-1-fundamentals/               # K8s architecture, cluster setup
│   ├── 01-k8s-architecture.md
│   ├── 02-setup-minikube.md
│   ├── 03-setup-kubeadm.md
│   ├── scripts/                        # Automated setup scripts
│   ├── labs/                           # Lab 1-2: Cluster exploration
│   └── interview-prep/
│
├── phase-2-core-components/            # Pods, Services, Deployments
│   ├── 01-pods.md
│   ├── 02-services.md
│   ├── 03-deployments.md
│   ├── 04-configmaps-secrets.md
│   ├── manifests/                      # 15+ YAML examples
│   ├── docker-images/                  # Custom apps for labs
│   ├── labs/                           # Lab 1-4: Hands-on practice
│   └── interview-prep/
│
├── phase-3-networking-storage/         # Ingress, PVC, StatefulSets
│   ├── manifests/
│   ├── labs/
│   └── interview-prep/
│
├── phase-4-production-patterns/        # Resources, RBAC, HPA, Health checks
│   ├── manifests/
│   ├── labs/
│   └── interview-prep/
│
├── phase-5-real-world-project/         # ⭐ Microservices portfolio piece
│   ├── services/                       # API Gateway, User, Order, Payment
│   ├── manifests/                      # Complete deployment configs
│   ├── docker-compose.yaml             # Local reference
│   ├── monitoring/                     # Prometheus + Grafana
│   ├── deployment-guide.md
│   ├── troubleshooting.md
│   └── interview-prep/
│
├── phase-6-cicd-integration/           # Jenkins → K8s + GitOps
│   ├── jenkins/                        # Jenkinsfile for K8s deployment
│   ├── gitops/                         # ArgoCD setup
│   ├── manifests/                      # Blue-Green, Canary
│   ├── labs/
│   └── interview-prep/
│
├── phase-7-managed-services/           # EKS, AKS, GKE
│   ├── eks/                            # Terraform + deployment
│   ├── aks/
│   ├── gke/
│   ├── labs/
│   └── interview-prep/
│
├── reference/                          # Quick lookup resources
│   ├── kubectl-cheatsheet.md
│   ├── yaml-patterns.md
│   ├── troubleshooting-guide.md
│   ├── security-best-practices.md
│   └── performance-tuning.md
│
├── tools-setup/                        # Prerequisites installation
│   ├── prerequisites.sh
│   ├── linux-ubuntu-setup.sh
│   └── macos-setup.sh
│
├── PREREQUISITES.md                    # System requirements
├── LEARNING-NOTES.md                   # Your progress tracking
└── README.md                           # This file
```

---

## 🎓 What You'll Learn

### **Technical Skills**
- ✅ Kubernetes architecture & core concepts
- ✅ Container orchestration & scheduling
- ✅ Microservices deployment patterns
- ✅ Production-grade configurations (RBAC, namespaces, resource management)
- ✅ Networking & service discovery
- ✅ Persistent storage & StatefulSets
- ✅ Autoscaling & health checks
- ✅ Monitoring, logging, and debugging
- ✅ CI/CD integration (Jenkins → K8s)
- ✅ GitOps principles (ArgoCD)
- ✅ Managed Kubernetes (EKS, AKS, GKE)
- ✅ Infrastructure as Code (Terraform)

### **DevOps Mindset**
- Infrastructure automation
- Deployment reliability
- Production troubleshooting
- Scaling & performance optimization
- Security & RBAC
- Monitoring-driven operations

### **Interview Readiness**
- 100+ interview questions across all phases
- Deep-dive explanations (why, not just how)
- Real-world scenario discussions
- Common mistakes & best practices
- Comparison matrix (EKS vs AKS vs GKE)

---

## 🔥 Standout Features

### **1. QA-to-DevOps Optimized**
This repo is **built for QA engineers** transitioning to DevOps. It connects:
- Testing concepts → automation patterns
- Deployment validation → health checks
- Test environments → namespaces & multi-tier deployments

### **2. Production-Ready Examples**
Not "hello-world" tutorials. Every example is production-grade:
- Resource requests + limits
- Health checks (liveness + readiness)
- RBAC & security
- Proper error handling.

### **3. Real Microservices Project** (Phase 5)
A 4-service application:
- API Gateway (Node.js)
- User Service (Python FastAPI)
- Order Service (Go)
- Payment Service (Java Spring Boot)
- MySQL + Redis
- Prometheus + ELK monitoring

Deploy, scale, troubleshoot, monitor—all documented.

### **4. Interview-Sharp Organization**
Each phase has dedicated `interview-prep/` folder with:
- Expected questions
- Deep-dive answers
- Diagrams & explanations
- Common mistakes

### **5. Git-as-Learning-Journal**
Structured branching strategy mirrors professional workflows:
- `phase/*` branches = active learning
- `lab/*` branches = completed labs
- Clean commit history = demonstrates learning progression

### **6. Multi-Environment Learning**
- **Local**: Minikube (quick, safe)
- **Local Self-Managed**: Kubeadm (production-like)
- **Cloud**: EKS, AKS, GKE (managed Kubernetes)

---

## 🏃 Quick Start

### **Prerequisites** (5 minutes)
```bash
# Check requirements
- Linux/macOS or Windows with WSL2
- 8GB+ RAM, 20GB disk space
- Docker installed
- kubectl v1.28+

# One-command setup
bash tools-setup/linux-ubuntu-setup.sh
```

### **Day 1: Start Phase 1**
```bash
# Clone repo
git clone https://github.com/YOUR-USERNAME/kubernetes-devops-learning-labs.git
cd kubernetes-devops-learning-labs

# Create first branch
git checkout -b phase/1-fundamentals

# Read foundation
cat phase-1-fundamentals/01-k8s-architecture.md

# Setup minikube (15 minutes)
bash phase-1-fundamentals/scripts/install-minikube.sh

# First lab
cd phase-1-fundamentals/labs
bash lab-01-explore-cluster.sh
```

### **Week 1 Checkpoint**
- [ ] Minikube cluster running
- [ ] Kubeadm cluster running (3-node)
- [ ] kubectl commands working
- [ ] Phase 1 labs complete
- [ ] Notes committed to git

---

## 📈 Learning Progression

### **By End of Phase 1** (Week 1-2)
```
You understand:
- How K8s clusters work internally
- Control Plane vs Data Plane
- Nodes, Pods, Container runtime
- kubectl basics
```

### **By End of Phase 2** (Week 2-3)
```
You can deploy:
- Single-pod applications
- Multi-replica Deployments
- Service-to-service communication
- Config injection (ConfigMaps/Secrets)
```

### **By End of Phase 3** (Week 4)
```
You can deploy:
- External traffic routing (Ingress)
- Databases with persistent storage
- StatefulSet applications
```

### **By End of Phase 4** (Week 5)
```
You understand:
- Production considerations
- Auto-scaling
- Health checks
- Security (RBAC)
```

### **By End of Phase 5** (Week 6-7) ⭐ Portfolio Piece
```
You have deployed a complete microservices architecture:
- 4 independent services
- API Gateway routing
- Database + caching layer
- Monitoring stack
- Full troubleshooting guide
```

### **By End of Phase 6** (Week 8)
```
You can:
- Deploy from Jenkins to K8s
- Use GitOps (ArgoCD)
- Implement Blue-Green & Canary deployments
```

### **By End of Phase 7** (Week 9)
```
You have deployed to:
- AWS EKS (with Terraform)
- Azure AKS (with Terraform)
- Google GKE (with gcloud)
```

---

## 🎬 How to Use This Repo

### **Option A: Follow the Fast-Track (Recommended)**
1. Start with Phase 1 (Week 1)
2. Progress sequentially through Phase 7 (Week 9)
3. Complete all labs (hands-on is critical)
4. Build Phase 5 project (portfolio piece)

### **Option B: Targeted Learning**
Know Docker, want to jump to production patterns?
- Phase 1: Skim (1 hour)
- Phase 2-3: Thorough (2 weeks)
- Phase 4+: Detailed (continue)

### **Option C: Interview Prep Only**
Limited time before interview?
- Read each phase's `interview-prep/` folder
- Review `reference/` section
- Study Phase 5 project (shows hands-on)

---

## 🏆 What This Repo Shows Recruiters

### **Technical Competence**
- ✅ Structured learning (not scattered tutorials)
- ✅ Hands-on labs (not theory only)
- ✅ Production-ready examples (not toy projects)
- ✅ Deep understanding (interview prep shows thinking)

### **Professional Approach**
- ✅ Organized Git history (clean commits, proper branching)
- ✅ Documentation (every lab is documented)
- ✅ Troubleshooting knowledge (common issues covered)
- ✅ Real-world scenarios (microservices, monitoring)

### **DevOps Mindset**
- ✅ Infrastructure as Code (Terraform)
- ✅ Automation (CI/CD pipelines)
- ✅ Security (RBAC, secrets)
- ✅ Monitoring & observability (Prometheus, ELK)

> When a recruiter asks "Tell me about your Kubernetes experience," this repo is your proof.

---

## 💡 Key Learnings & Mindset Shifts

### **From QA to DevOps**

| QA Thinking | DevOps Thinking |
|-------------|-----------------|
| "Did the app work?" | "Does the infrastructure work reliably?" |
| "Test in staging" | "Deploy safely to production" |
| "Find bugs" | "Prevent issues at scale" |
| "Manual testing" | "Automated, self-healing systems" |
| "Isolated environments" | "Connected microservices" |

### **Common Mistakes (We'll Help You Avoid)**
1. ❌ **No resource limits** → App works locally, prod is chaos
2. ❌ **Direct pods in production** → Always use Deployments
3. ❌ **No health checks** → Service "up" but actually broken
4. ❌ **Hardcoded configs** → Use ConfigMaps/Secrets
5. ❌ **Ignoring namespaces** → Dev and prod collide
6. ❌ **No monitoring** → Can't troubleshoot production issues

---

## 📚 Phase Deep-Dive Guides

| Phase | Duration | Focus | Outcome |
|-------|----------|-------|---------|
| **[Phase 1: Fundamentals](./phase-1-fundamentals/README.md)** | Weeks 1-2 | Architecture, Setup | Cluster ready |
| **[Phase 2: Core Components](./phase-2-core-components/README.md)** | Weeks 2-3 | Pods, Services, Deployments | Microservices ready |
| **[Phase 3: Networking & Storage](./phase-3-networking-storage/README.md)** | Week 4 | Ingress, PVC, StatefulSets | Production data handling |
| **[Phase 4: Production Patterns](./phase-4-production-patterns/README.md)** | Week 5 | RBAC, HPA, Health checks | Production-ready |
| **[Phase 5: Real-World Project](./phase-5-real-world-project/README.md)** | Weeks 6-7 | Microservices app | **Portfolio piece** |
| **[Phase 6: CI/CD Integration](./phase-6-cicd-integration/README.md)** | Week 8 | Jenkins, GitOps | Automated deployments |
| **[Phase 7: Managed Services](./phase-7-managed-services/README.md)** | Week 9 | EKS, AKS, GKE | Cloud-native ready |

---

## 🛠️ Tools & Technologies

```
Container Orchestration:
  Kubernetes 1.28+, Docker, Podman

Local Development:
  Minikube, Kubeadm, kind

Cloud Platforms:
  AWS EKS, Azure AKS, Google GKE

Configuration Management:
  Helm, Kustomize, ArgoCD

CI/CD:
  Jenkins, GitHub Actions, GitLab CI

Infrastructure as Code:
  Terraform, CloudFormation

Monitoring & Logging:
  Prometheus, Grafana, ELK Stack, CloudWatch

Security:
  RBAC, Network Policies, Secrets

Package Registry:
  Docker Hub, ECR, ACR, GCR
```

---

## 📖 Reference Materials

### **Quick Lookups**
- [kubectl Cheatsheet](./reference/kubectl-cheatsheet.md) — Commands you'll use daily
- [YAML Patterns](./reference/yaml-patterns.md) — Copy-paste manifest templates
- [Troubleshooting Guide](./reference/troubleshooting-guide.md) — Debug common issues
- [Security Best Practices](./reference/security-best-practices.md) — Production hardening
- [Performance Tuning](./reference/performance-tuning.md) — Optimization techniques

### **External Resources**
- [Kubernetes Official Docs](https://kubernetes.io/docs/)
- [CNCF Kubernetes Curriculum](https://www.cncf.io/)
- [Kubernetes the Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way)

---

## 🎯 Interview Preparation

Each phase includes dedicated interview content:

### **What You'll Be Ready For**
- ✅ "Explain Kubernetes architecture"
- ✅ "How does service discovery work?"
- ✅ "Describe a production deployment"
- ✅ "What are resource requests and limits?"
- ✅ "Compare EKS vs AKS vs GKE"
- ✅ "How would you troubleshoot a failing pod?"
- ✅ "Explain rolling updates and rollbacks"
- ✅ "Design a microservices deployment"

### **Preparation Strategy**
1. Complete each phase thoroughly
2. Review `interview-prep/` after phase completion
3. Practice explaining concepts out loud
4. Reference Phase 5 project during interviews ("I deployed a 4-service microservices app with...")

---

## 📊 Progress Tracking

Track your learning in `LEARNING-NOTES.md`:

```markdown
## Learning Progress

### Phase 1: Fundamentals
- [x] K8s architecture understood
- [x] Minikube setup complete
- [x] Kubeadm 3-node cluster running
- [x] Lab 1 & 2 complete
- [x] Interview prep reviewed

### Phase 2: Core Components
- [ ] Pods concept mastered
- [ ] Services deployed
- [ ] Deployments working
...
```

---

## 🤝 Contributing

This repo documents **your** learning journey. Feel free to:
- Add personal notes
- Modify examples for your use case
- Create additional labs
- Document discoveries

> This is your portfolio—make it reflect your understanding and style.

---

## 🔒 Security Note

This repo contains learning examples. Before deploying to production:
- ✅ Review [Security Best Practices](./reference/security-best-practices.md)
- ✅ Implement RBAC properly
- ✅ Use Secret management (AWS Secrets Manager, Azure KeyVault, etc.)
- ✅ Enable network policies
- ✅ Audit logs and monitoring

---

## 📞 Support & Questions

Stuck on a lab? Confused about a concept? Tips:

1. **Check the troubleshooting guide** → `reference/troubleshooting-guide.md`
2. **Review the phase README** → Each phase has a detailed guide
3. **Look at interview prep** → Explains concepts deeply
4. **Revisit previous labs** → Often the answer is in earlier work

---

## 📋 Quick Navigation

- **Getting Started?** → Start with [Phase 1](./phase-1-fundamentals/README.md)
- **Need kubectl commands?** → See [kubectl Cheatsheet](./reference/kubectl-cheatsheet.md)
- **Portfolio project?** → Go to [Phase 5](./phase-5-real-world-project/README.md)
- **Interview prep?** → Each phase has `interview-prep/` folder
- **Troubleshooting?** → Check [Troubleshooting Guide](./reference/troubleshooting-guide.md)

---

## 🎓 Learning Outcomes

After completing this repo, you will be able to:

- ✅ Design and deploy microservices on Kubernetes
- ✅ Set up production-grade clusters (self-managed & managed)
- ✅ Implement CI/CD pipelines to Kubernetes
- ✅ Debug and troubleshoot Kubernetes issues
- ✅ Apply security best practices (RBAC, network policies)
- ✅ Monitor and scale applications in production
- ✅ Use Infrastructure as Code (Terraform)
- ✅ Compare and choose between cloud Kubernetes options

---

## ⭐ Star This Repo

If this learning path helps you, please give it a star! It helps others find this resource.

---

## 📝 License

MIT License — Feel free to fork, modify, and share.

---

## 🚀 Ready to Start?

**Next Step**: Open [Phase 1: Fundamentals](./phase-1-fundamentals/README.md) and begin your Kubernetes journey!

```bash
# Clone and get started
git clone https://github.com/YOUR-USERNAME/kubernetes-devops-learning-labs.git
cd kubernetes-devops-learning-labs
git checkout -b phase/1-fundamentals
```

---

<div align="center">

**Made with ❤️ for DevOps engineers in transition**

From QA to DevOps | Kubernetes Mastery | Production-Ready Learning

[⬆ Back to Top](#kubernetes-devops-learning-labs)

</div>
