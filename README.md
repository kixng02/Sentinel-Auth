# LoginCompact 360 (.NETCore Ecosystem)
# Credential Stuffing Detection and Alerting System
A production-ready system built with .NET Core, designed to handle and scale a millions of users.
(This tool is best suited for intergration with login forms)
- The system acts as Second layer of protection on the your login form
- The security admin can determine the geolaction they intend their users to be coming from; If a user logs in from a not permitted location the security team of your company gets alerted 
- The .NET Core backend logs every failed login attempt to a dedicated Postgres auditing table. 
- A separate background worker analyzes this table for patterns (e.g., failed attempts across many accounts from one IP) and triggers alerts. k6 simulates a continuous, distributed credential stuffing attack.
- A service that records every critical security action (login, logout, password change, permission grant) as an immutable, time-stamped record in a dedicated Postgres log table.
- k6 simulates heavy bursts of user activity to stress the write performance of the logging service.
- A system to handle immediate revocation of access tokens (e.g., after a password change or forced logout). The .NET Core API checks this list (which can be memory-cached from Postgres) on every request. k6 tests the performance hit of checking the TRL against the benefit of immediate revocation.

# Technology Stack
- Frontend: Razor OR Blazer (Not yet decided)
- Analytics Dashboard: Streamlit (A list of the libraries and their uses is found in the project documentation)
- Backend: .NET Core Web API
- Geolocation Map: open-source, MapLibre GL JS https://github.com/maplibre/maplibre-gl-js
- Database: PostgreSQL
- Authentication: JWT, ASP.NET Core Identity
- Security: bcrypt, security headers, CORS policies
- Monitoring: Prometheus and Grafana stack.
- Development Tools: Git, Docker, Entity Framework Core

# 1. ⚙️ Real-Time Log Ingestion and Throttling
- This feature focuses on securing the login endpoint and efficiently capturing the data needed for detection.

  .NET Core Backend (Login API)
- Secure Hashing: Implement the login logic using a high-work-factor hashing algorithm like Argon2 (integrated via a custom PasswordHasher in ASP.NET Core Identity).
- Immediate Logging: After a failed login attempt, log the required metadata (IP address, timestamp, username/email, and device signal hash) to the LoginAttempts Postgres table asynchronously to avoid adding latency to the user's failed response.IP-Based
- Throttling- Implement a lightweight, highly-cached rate limiter (e.g., using a Concurrent Dictionary or Redis in the .NET Core middleware) to enforce a hard limit on login attempts per IP address (e.g., 10 attempts per minute) to slow down simple, non-distributed attacks.
- Postgres Schema- The primary table, LoginAttempts, must be indexed heavily on IPAddress and AttemptTime for fast queries by the detection worker.
- k6 Simulation: Design a low-volume, high-frequency test where a single k6 Virtual User (VU) sends login attempts from the same IP address to verify the IP throttling mechanism correctly returns HTTP 429 errors after the limit is hit.

# 2. 🕵️ Background Detection Worker (The Analyzer)
This is the core analysis engine that runs the detection logic.
- .NET Core Backend (Worker Service)- Implement a .NET Core Background Service (or a separate worker application using IHostedService) that runs on a schedule (e.g., every 5 minutes).
- Detection Logic (Postgres Queries)- The worker executes complex, optimized SQL queries against the LoginAttempts table to identify suspicious patterns.
Key queries include
- Query A (IP Fingerprint)- Find all IP addresses that have failed logins against N unique usernames (e.g., $N > 50$) within the last hour.
- Query B (Account Focus)- Find all usernames that have received failed login attempts from M unique IP addresses (e.g., $M > 10$) within the last 10 minutes.Postgres
- Optimization: These queries must use efficient GROUP BY and HAVING clauses, relying on the indexes to perform fast counts across millions of log records.
- k6 Simulation- The main credential stuffing test. k6 VUs use a list of 10,000 unique usernames and 5 common passwords. They perform a continuous, distributed attack, generating millions of entries in the LoginAttempts table to ensure the worker's queries remain performant under load.

# 3. 🚨 Alerting and Remediation PipelineOnce a suspicious pattern is identified, 
This feature handles the appropriate response.
- .NET Core Backend (Alerting Module): Risk Scoring The detection worker (from Section 2) assigns a Risk Score (e.g., 0 to 100) to each suspicious IP address or username based on the magnitude of the attack.
- Remediation Action: For highly targeted accounts, the worker should call a secured internal API endpoint to force a temporary user account lockout or force password reset for affected accounts.
- k6 Simulation - Include a test where the attack simulation is so severe that it exceeds the high-score threshold. The k6 VUs then check the status of the "remediated" accounts (the ones targeted in the alert) and confirm they are now locked out, verifying the full end-to-end security pipeline.

Action Trigger
- Low Score- Log an entry in a SecurityAlerts table.
- High Score (Mass Attack)- Trigger an immediate email/Slack/SMS notification (e.g., via a simple .NET Core HttpClient call to a third-party service).

# 4. 🗄️ Log Retention and Archiving 
Handling the massive volume of data generated by a large-scale system is critical for long-term operational 
- health.Postgres Strategy- Implement Table Partitioning in Postgres for the LoginAttempts table based on the AttemptTime column (e.g., monthly partitions). This dramatically improves the performance of the worker's queries (which typically only look at recent data) and simplifies archiving.Use a separate, less-indexed ArchiveLog table.
- .NET Core Backend (Maintenance Service):Implement another scheduled .NET Core Background Service responsible for maintenance.
- Archiving Logic- On a weekly basis, move all records older than a configured period (e.g., 90 days) from the highly-indexed LoginAttempts table into the ArchiveLog table or delete them entirely, ensuring the primary detection table remains fast and manageable.
- k6 Simulation: While k6 doesn't directly test archiving, a stress test generating a month's worth of log data (simulated at a very fast pace) should be performed to measure the impact of the partitioning strategy on the detection worker's query times.

# 5. 📉 Dasboard & Monitoring
(i) 📊 Streamlit (Businessend Analytics)
Streamlit is for Business/Security-Specific Data Analysis. Its primary purpose in your system is to
- Analyze Audit Logs- It connects to your PostgreSQL database to query the contents of the LoginAttempts and SecurityAlerts tables.
- Visualize Findings- It visualizes the results of the detection worker's job: "Top 20 Targeted Accounts," "Suspicious IPs by Geo," and "Total Unique Failed Logins."
- Decision Support- It's an application dashboard used by security analysts to make remediation decisions.

(ii) ⏱️ Prometheus & Grafana (Performance & System Health)
Prometheus and Grafana are for System and Infrastructure Monitoring. Their primary purpose is to monitor the health and performance of the code and infrastructure itself

- Prometheus (Collector)- Gathers metrics (numerical, time-series data) from the .NET Core backend. This includes
  
1. Latency: The 99th percentile ($p(99)$) duration of your login API and detection worker runtime.
2. Errors: HTTP 500 error counts, connection errors to Postgres.
3. Resource Usage: CPU/Memory usage of the .NET Core and Postgres processes.
4. Custom Code Metrics: Total times the detection worker ran, count of high-risk alerts triggered by the worker.

Grafana (Visualizer & Alerter)
-  Displays these real-time metrics and allows you to set up alerts.


6. Detection Methodology
- Behavioral Analysis: Machine learning models to identify unusual login patterns and anomalies
- IP Intelligence: Real-time integration with threat intelligence databases and reputation services
- Account Targeting: Detection of coordinated attacks against multiple user accounts
- Temporal Analysis: Identification of attack waves, campaign patterns, and time-based anomalies
- Credential Correlation: Analysis of credential pairs across multiple authentication attempts


# Key features
🔍 Attack Detection
- Pattern Recognition: Identify credential stuffing patterns using behavioral analytics and machine learning
- Velocity Monitoring: Detect rapid login attempts across multiple accounts with configurable thresholds
- IP Reputation Analysis: Cross-reference attacker IPs with real-time threat intelligence feeds
- Geolocation Analytics: Flag suspicious login attempts from unusual geographic locations
- User Agent Analysis: Detect suspicious browser fingerprints and automation tools

🚨 Real-time Alerting
- Instant Notifications: Immediate alerts for suspected credential stuffing attacks via multiple channels
- Multi-channel Alerts: Notifications via email, SMS, Slack, Microsoft Teams, and webhooks
- Custom Thresholds: Configurable sensitivity levels based on organizational risk tolerance
- Escalation Protocols: Automated escalation workflows for high-severity security incidents
- Alert Triage: Priority-based alert management with false positive reduction

📊 Security Analytics Dashboard
- Attack Visualization: Streamlit-powered dashboards showing real-time attack patterns and trends
- Threat Intelligence: Correlation with known attack campaigns and global IP blacklists
- Impact Assessment: Analysis of compromised accounts and potential data exposure risks
- Historical Analysis: Long-term tracking of attack patterns, success rates, and mitigation effectiveness
- Compliance Reporting: Ready-made reports for security audits and regulatory compliance

🛡️ Mitigation & Response
- Automated Blocking: Real-time IP, CIDR range, and user agent blocking capabilities
- Progressive Challenges: CAPTCHA and multi-factor authentication for suspicious activities
- Account Protection: Automatic locking and protection of targeted user accounts
- Incident Response: Guided workflows with integration to SOAR platforms
- Rate Limiting: Intelligent rate limiting based on threat level and user behavior

- Distributed caching with Redis for session management
- Database optimization for high-concurrency scenarios
- Horizontal scaling support across multiple instances
- Efficient token validation with minimal overhead

⚡ Running Performance Tests
We use k6 to simulate massive user loads and ensure system reliability
Test scenarios include
- Peak traffic simulation - 1M+ user spike testing
- Endurance testing - Sustained load over hours
- Stress testing - Beyond-capacity scenarios
- Smoke testing - Basic functionality under load


# 🤝 Contributing
We welcome contributions! Please see our Contributing Guide for details.
Fork the repository

- Create a feature branch (git checkout -b feature/amazing-feature)

- Commit your changes (git commit -m 'Add amazing feature')

- Push to the branch (git push origin feature/amazing-feature)

- Open a Pull Request


# Installation
1. Clone the repository
- git clone https://github.com/kixng02/CredCompact360.git
- cd CredCompact360

2. Frontend Setup (React + TypeScript)
- cd frontend
- npm install
- npm run dev

3. Security Dashboard Setup (Streamlit)
- cd dashboard
- pip install -r requirements.txt
- streamlit run security_dashboard.py

4. Backend Setup (.NET Core)
- cd backend/CredCompact360.API
- dotnet restore
- dotnet run


# Security Notice
This tool is designed for hobby and educational purposes to showcase credential stuffing detection capabilities. The implementation provided here is intended for learning, personal projects, and demonstration environments.

For business or production deployment: We strongly recommend reaching out to the repository owners for professional guidance and enterprise-grade implementation. Production environments require additional security considerations, comprehensive testing, and enterprise-level support that extends beyond this showcase implementation.

Always ensure proper authorization and compliance with local laws and regulations before deploying any security monitoring tool. Conduct thorough testing in isolated environments before any deployment.

# 📄 License
This project is licensed under the MIT License - see the LICENSE file for details.

*Built with ❤️ using .NET 8 - Ready for your next million-user application!*


