# System Design Documents

A collection of comprehensive system design documents for large-scale distributed systems.

## 📚 Available Designs

### [Real-Time Stock Trading Platform](./stock_trading_platform_design.md)
**Scale:** 1M+ concurrent users, 10K TPS  
**Tech Stack:** Node.js microservices, React Native, Kubernetes, PostgreSQL, Redis, Kafka  
**Highlights:**
- Complete microservices architecture with event-driven design
- WebSocket-based real-time quote streaming (<100ms latency)
- PostgreSQL sharding strategy for 100M+ users
- Kubernetes + Istio deployment with auto-scaling
- Comprehensive capacity planning and AWS cost estimates ($46K/month)
- GDPR compliance and security measures
- Production-ready deployment scripts (Terraform + K8s YAML)

## 🎯 Purpose

These documents serve as:
- **Technical specifications** for building production systems
- **Interview preparation** for system design discussions
- **Reference architectures** for scalable applications
- **Learning resources** for distributed systems concepts

## 📖 Document Structure

Each design document includes:
1. **Requirements** - Functional and non-functional specifications
2. **Architecture** - High-level and detailed component designs with diagrams
3. **Data Layer** - Database design, sharding, caching strategies
4. **Scalability** - Capacity planning, auto-scaling, performance optimization
5. **Reliability** - Failure scenarios, disaster recovery, monitoring
6. **Security & Compliance** - Authentication, encryption, regulatory compliance
7. **Deployment** - Infrastructure as code, CI/CD pipelines
8. **Cost Analysis** - Cloud infrastructure cost estimates

## 🛠️ Technologies Covered

- **Backend:** Node.js, Express, TypeScript
- **Frontend:** React Native
- **Databases:** PostgreSQL, Redis, Elasticsearch
- **Messaging:** Apache Kafka
- **Infrastructure:** Kubernetes, Istio, Terraform
- **Cloud:** AWS (EKS, RDS, ElastiCache, MSK)
- **Observability:** Prometheus, Grafana, Jaeger

## 📝 License

MIT License - Feel free to use these designs for learning and reference.

---

**Author:** Hitesh Ahuja  
**Last Updated:** January 2026
