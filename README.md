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

### [E-Commerce: Monolith-to-Microservices Migration](./ecommerce_monolith_to_microservices.md)
**Migration Timeline:** 6 months  
**Tech Stack:** Node.js/Express, PostgreSQL, Redis, Kafka, Docker, Kubernetes  
**Highlights:**
- Complete migration guide using Strangler Pattern
- Before/after architecture with C4 diagrams (Levels 1-3)
- Database decomposition from monolith to database-per-service
- Kafka event-driven architecture with event schemas
- Feature flags for gradual rollout (0% → 100%)
- Comprehensive rollback plans and risk mitigation
- Docker Compose for local development
- Production Kubernetes manifests
- Contract testing, integration testing, and E2E testing strategies

### [Native Stock Trading App Ecosystem](./native_stock_trading_app.md)
**Platform:** iOS (SwiftUI + TCA) & Android (Jetpack Compose + MVI)  
**Scale:** 5M DAU  
**Deep Dive:** [Technical Implementation Details](./native_trading_deep_dive.md)
**Highlights:**
- Platform-specific architectures (TCA for iOS, MVI for Android)
- SwiftUI and Jetpack Compose code skeletons
- iOS features: Live Activities, WidgetKit, App Intents, Vision framework (KYC)
- Android features: Dynamic Feature Modules, WorkManager, Wear OS, BiometricPrompt
- 120fps chart rendering with Metal (iOS) and Canvas API (Android)
- Offline-first with SwiftData and Room database
- Native WebSocket clients for real-time quotes
- Shared gRPC / Protobuf infrastructure
- Fastlane (iOS) and Gradle (Android) CI/CD pipelines

### [React Native Stock Trading App](./rn_stock_trading_app.md)
**Platform:** Cross-platform (iOS, Android, Web via Expo)  
**Scale:** 5M DAU  
**Highlights:**
- MVVM Architecture with Zustand and Repository pattern
- Offline-first data layer using WatermelonDB / expo-sqlite
- Real-time quote management with Socket.io and Reanimated
- Optimistic UI updates and background synchronization
- Performance tuning with Hermes, FlashList, and bundle analysis
- Native integration (Biometrics, Push Notifications, Deep Linking)
- Deployment strategy via EAS (Expo Application Services)
- Complete infrastructure monitoring with Sentry and Firebase


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
