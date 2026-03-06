# Code of Conduct

**Document ID**: POL-013
**Version**: 1.0
**Effective Date**: 2026-02-16
**Last Reviewed**: 2026-02-16
**Next Review**: 2027-02-16
**Owner**: CTO
**Approved By**: CEO, Questo
**Classification**: INTERNAL

---

## 1. Purpose

This Code of Conduct establishes the ethical standards and professional behavior expectations for all Questo Ltd personnel. It defines the values that guide decision-making, the ethical principles governing the development and operation of the corteX AI Agent SDK, and the specific responsibilities that accompany building AI systems deployed in enterprise environments.

Questo Ltd recognizes that as a creator of AI agent technology, it bears a heightened responsibility to ensure that its products are developed and operated ethically, safely, and with appropriate consideration for the societal impact of autonomous AI systems. This code addresses both general professional conduct and AI-specific ethical obligations.

**SOC 2 Mapping**: CC1.1 (Control Environment - Ethical Values), CC1.4 (Commitment to Integrity)

## 2. Scope

This policy applies to:

- **Personnel**: All employees, contractors, consultants, and temporary staff of Questo Ltd.
- **Activities**: All professional activities conducted on behalf of Questo Ltd, including development, sales, support, communication, and business operations.
- **Interactions**: All interactions with colleagues, customers, vendors, partners, competitors, regulators, and the public.
- **Products**: The design, development, testing, distribution, and support of the corteX AI Agent SDK and all associated products and services.
- **Representations**: All representations made about Questo, corteX, or AI capabilities, whether in technical documentation, marketing, sales, or casual conversation.

## 3. Definitions

| Term | Definition |
|------|-----------|
| **Ethical AI** | The practice of designing, developing, and deploying AI systems in a manner that is fair, transparent, accountable, and aligned with human values |
| **AI Safety** | The field concerned with ensuring AI systems behave as intended and do not cause unintended harm |
| **Conflict of Interest** | A situation where personal interests could compromise professional judgment or obligations to Questo |
| **Whistleblower** | A person who reports misconduct, illegal activity, or ethical violations in good faith |
| **Agent Autonomy** | The degree to which a corteX AI agent can act independently without human oversight |
| **SafetyPolicy** | The corteX module that enforces input/output safety checks and prompt injection protection |
| **GoalTracker** | The corteX module that verifies every agent step against the original goal |

## 4. Roles and Responsibilities

### 4.1 CEO

- Sets the tone for ethical conduct across the organization
- Serves as the ultimate authority for ethical decisions
- Ensures that business objectives do not compromise ethical standards
- Receives and acts on reports of serious ethical violations

### 4.2 CTO (Policy Owner)

- Owns this policy and ensures AI ethics principles are implemented in the corteX SDK
- Reviews ethical concerns related to product features and capabilities
- Ensures that engineering decisions consider ethical implications
- Reports ethical concerns and trends to the CEO

### 4.3 All Personnel

- Read, understand, and abide by this code
- Report ethical concerns or violations through the established channels
- Consider the ethical implications of their work, especially as it relates to AI capabilities
- Seek guidance when facing ethical dilemmas

## 5. Policy Statements

### 5.1 Core Values

Questo Ltd operates according to the following core values:

1. **Integrity**: We are honest, transparent, and consistent in our actions. We do not misrepresent our products, capabilities, or intentions.
2. **Responsibility**: We take ownership of the impact of our technology. We design safety controls as a core feature, not an afterthought.
3. **Respect**: We treat all individuals with dignity and respect, regardless of role, background, or status.
4. **Excellence**: We pursue the highest quality in our work. The corteX SDK maintains 8,082+ automated tests and rigorous code review because quality protects our customers.
5. **Trust**: We earn customer trust through consistent, reliable behavior. We never compromise customer data security for convenience.

### 5.2 Professional Conduct

#### 5.2.1 Workplace Behavior

1. Treat all colleagues with professionalism, respect, and courtesy.
2. Harassment, discrimination, bullying, and retaliation are prohibited in all forms.
3. Foster an inclusive environment where diverse perspectives are valued and all team members feel safe contributing.
4. Communicate honestly and constructively. Disagree respectfully and professionally.
5. Protect the confidentiality of personnel information, business discussions, and customer data.

#### 5.2.2 Conflicts of Interest

1. Disclose any actual or potential conflict of interest to the CTO or CEO promptly.
2. Do not engage in outside employment, consulting, or business activities that conflict with your obligations to Questo.
3. Do not use Questo resources, information, or relationships for personal gain.
4. Do not accept gifts or payments from vendors, customers, or competitors that could influence your professional judgment, except for nominal value items ($50 maximum).
5. If you are uncertain whether a situation constitutes a conflict, disclose it and seek guidance.

#### 5.2.3 Intellectual Property

1. Respect the intellectual property rights of others. Do not incorporate copyrighted code, patented algorithms, or trade secrets without proper authorization.
2. All work product created during your engagement with Questo is Questo's intellectual property per your employment or contractor agreement.
3. Do not disclose Questo trade secrets, proprietary algorithms (brain-inspired engine, weight system, prediction algorithms), or confidential technical details to unauthorized parties.
4. Open-source contribution guidelines are defined in POL-010.

#### 5.2.4 Compliance with Laws

1. Comply with all applicable laws and regulations in every jurisdiction where Questo operates.
2. If you believe Questo is engaged in illegal activity, report it immediately through the channels in Section 6.2.
3. Particular attention is required for: GDPR (data protection), export control regulations, anti-bribery laws, and competition laws.

### 5.3 AI Ethics Principles

The following principles govern the design, development, and operation of the corteX AI Agent SDK:

#### 5.3.1 Safety by Default

1. The corteX SDK ships with safety controls enabled by default. `SafetyPolicy` enforcement is on by default, and `ToolPolicy` uses a deny-all default posture.
2. Safety controls must not be made easy to disable. The `SafetyLevel.LOCKED` mode prevents tenants from overriding safety configurations.
3. We never sacrifice safety for performance, speed-to-market, or customer convenience.
4. Every feature must be evaluated for potential misuse before implementation.
5. Loop prevention (state hashing, drift detection in `corteX/engine/loop_detector.py` and `corteX/engine/drift_engine.py`) protects against resource exhaustion and runaway agent behavior.

#### 5.3.2 Human Oversight

1. The corteX SDK is designed to augment human capabilities, not replace human judgment for consequential decisions.
2. `GoalTracker` (`corteX/engine/goal_tracker.py`) ensures that every agent step is verified against the original human-specified goal.
3. Agent autonomy levels must be configurable by the tenant. Customers control what actions their agents can take through `ToolPolicy` and `TenantConfig`.
4. Critical actions (data deletion, financial transactions, external communications) should always provide for human-in-the-loop confirmation in enterprise deployments.
5. We do not build features that make it impossible for a human to understand what an agent did and why.

#### 5.3.3 Transparency

1. We are transparent about what corteX agents can and cannot do. We do not overstate AI capabilities in documentation, marketing, or sales.
2. Agent decision-making is auditable through the audit logging system (`corteX/engine/audit_logger.py` and `corteX/security/audit_logger.py`).
3. The decision log (`corteX/engine/decision_log.py`) provides explainability for agent actions.
4. We document known limitations, failure modes, and edge cases in the SDK documentation.
5. When an agent produces unreliable or uncertain output, the SDK indicates this through confidence scoring rather than presenting low-confidence outputs as definitive.

#### 5.3.4 Fairness and Non-Discrimination

1. We design the corteX SDK to avoid amplifying biases present in LLM outputs.
2. The `SafetyPolicy` module includes checks for harmful, discriminatory, and biased content.
3. We do not knowingly enable use cases that discriminate against individuals or groups based on protected characteristics.
4. We provide guidance to customers on responsible AI deployment, including bias testing and fairness evaluation.

#### 5.3.5 Privacy by Design

1. The corteX SDK collects and processes the minimum data necessary for its function (data minimization).
2. Multi-tenant isolation is an architectural guarantee, not a configuration option. Tenant data cannot cross boundaries.
3. The `DataClassifier` (`corteX/security/classification.py`) automatically detects PII and escalates data classification.
4. On-premises-first design means that customer data can remain entirely within the customer's infrastructure.
5. BYOK means Questo does not need to hold or have access to customer API keys in production.

#### 5.3.6 Responsible Disclosure

1. If we discover a vulnerability in corteX that affects customer security, we disclose it transparently and promptly.
2. We provide security advisories for vulnerabilities that affect deployed SDK versions.
3. We welcome responsible security research and do not retaliate against researchers who report vulnerabilities in good faith.

### 5.4 Customer Trust Commitments

1. **No Hidden Capabilities**: The corteX SDK does not contain hidden telemetry, backdoors, or undisclosed data collection. Agent behavior is determined by the customer's configuration.
2. **Security as DNA**: Security controls are fundamental to the architecture, not optional add-ons. Every architectural decision considers the security implications for enterprise customers.
3. **Honest Communication**: We communicate honestly about incidents, vulnerabilities, and limitations. We do not downplay security events or misrepresent the status of issues.
4. **Customer Data Sovereignty**: In on-premises deployments, customers retain full control over their data. Questo cannot access customer data unless the customer explicitly grants access for support purposes.

### 5.5 Reporting and Whistleblower Protection

1. All personnel are encouraged to report ethical concerns, policy violations, legal violations, and safety concerns without fear of retaliation.
2. Reports may be made through any channel described in Section 6.2.
3. **Retaliation against good-faith reporters is strictly prohibited and will result in disciplinary action against the retaliator, up to and including termination.**
4. Reports are investigated confidentially to the extent possible.
5. Anonymous reporting is supported through the designated reporting mechanism.

## 6. Procedures

### 6.1 Code of Conduct Acknowledgement

1. All personnel receive this code upon hire or engagement.
2. Personnel sign an acknowledgement within 14 days of receipt.
3. The code is re-acknowledged annually as part of security awareness training.
4. Acknowledgement records are maintained for the duration of employment plus 3 years.

### 6.2 Reporting Ethical Concerns

Ethical concerns, policy violations, or suspected misconduct may be reported through:

1. **Direct Manager**: For routine concerns or questions about expected behavior.
2. **CTO**: For concerns related to product ethics, AI safety, or technical decisions.
3. **CEO**: For concerns involving the CTO, senior leadership conduct, or organizational direction.
4. **Anonymous Channel**: A designated anonymous reporting mechanism for sensitive concerns.
5. **External Authorities**: Personnel may report to relevant regulatory authorities if internal channels fail to address the concern or if required by law.

### 6.3 Ethics Review for New Features

1. Before implementing a significant new feature in the corteX SDK, the engineering team conducts an ethics review considering:
   - Could this feature be used to cause harm?
   - Does this feature maintain appropriate human oversight?
   - Does this feature respect user privacy?
   - Does this feature maintain tenant data isolation?
   - Could this feature amplify biases or discriminate?
2. The ethics review is documented in the Pull Request description per POL-003.
3. Features with significant ethical implications require CTO review and approval.

### 6.4 Violation Investigation Procedure

1. Reports of code of conduct violations are received by the appropriate party per Section 6.2.
2. The investigating party (CTO, CEO, or designee) conducts a confidential investigation.
3. The investigation considers: the nature and severity of the violation, the intent, the impact, any mitigating circumstances, and the individual's history.
4. Findings and recommended actions are documented.
5. Enforcement actions are taken per Section 8.
6. The reporter is informed of the outcome to the extent possible while maintaining confidentiality.

## 7. Exceptions

1. This code represents minimum standards. Higher standards may be imposed for specific roles or situations.
2. No exceptions are permitted for: the prohibition on harassment and discrimination, the requirement for honest communication, the whistleblower protection provisions, or the AI safety-by-default principle.
3. Exceptions to other provisions require CEO approval with documented justification.
4. All exceptions are documented in the Exception Register per POL-001 Section 7.

## 8. Enforcement

- Violations of this code result in disciplinary action proportionate to the severity of the violation, up to and including termination.
- Minor violations (e.g., inadvertent conflict of interest, accidental disclosure) result in counseling and documented correction.
- Serious violations (e.g., harassment, intentional data mishandling, deliberate safety bypass, retaliation against reporters) result in formal disciplinary action, which may include immediate termination.
- Violations involving illegal activity are reported to law enforcement.
- The CEO makes final enforcement decisions for serious violations.
- Enforcement actions are documented and retained in personnel records.

## 9. Related Documents

| Document ID | Document Name |
|-------------|---------------|
| POL-001 | Information Security Policy |
| POL-004 | Incident Response Plan |
| POL-008 | Data Classification Policy |
| POL-010 | Acceptable Use Policy |

## 10. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-16 | CTO | Initial policy creation |
