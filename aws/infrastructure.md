# Module 1 - Architecting Fundamentals

## AWS Global Infrastructure

- 26 Regions
- Availability Zones
    - Each region consists of at least 3 availability zones.
- Data Centers
    - Each availability zone consists of one or more data centers.
    - Data centers within availability zones are separated by a few miles for redundancy.
- Local Zones
    - Bring AWS resources closer to where you are or to where your users are in order to reduce latency.
- Edge Locations
    - Content delivery network edge locations.

## Well Architected Framework

- Security
    - Apply at all layers
    - Enforce principal of least privilege
    - Use MFA
- Cost optimization
    - Analyze and attribute expenditures
    - Use cost-effective resources
    - Stop guessing (capacity requirements, etc)
- Reliability
    - Recover from failure
    - Test recovery procedures
    - Scale to increase availability
- Performance efficiency
    - Reduce latency
    - Use serverless architecture
    - Incorporate monitoring
- Operational excellence
    - Perform operations with code
    - Test response for unexpected events
- Sustainability
    - Understand your impact

# Module 2 - Account Security

- AWS Account Root User
    - Carte Blanche to everything
- IAM - Manage Users and Permissions
- Principal - an entity that needs permissions
- IAM Users
- IAM Roles are used when federated identity is being used
- Roles can be assumed
    - Trust Policy indicates who can assume a role.
- Policies
    - Effect (allow/deny, required)
    - Principal (entity)
    - Action (actions that policy allows or denies, required)
    - Resource (list of resources that the policy applies to, required)
    - Condition (circumstances under which the policy applies)
- Policy Evaluation
    - Deny if user is explicitly denied.
    - Allow if user is explicitly allowed.
    - Deny otherwise.
    - Service control policies are evaluated before other policies.
- Federation
    - SAML Federation
    - AWS SSO Service
- Managing Multiple Accounts
    - AWS Organizations
    - SCPs
    - AWS Control Tower
- Control Tower
    - Lets you spin up accounts for your organization.

# Module 3 - Networking 1

- Networking Architecture
    - Region
        - VPC
            - Public Subnet
            - Private Subnet
            - Internet Gateway
- CIDR
    - No more class A, B, or C
    - There are five CIDR blocks per VPC
    - No overlap is allowed.
    - x.x.0.0 is reserved - router
    - x.x.1 is reserved - local gateway
    - x.x.0.2 is reserved - DNS
    - x.x.0.3 is reserved - future
- Subnets
    - Public subnets have private IP addresses and can optionally have public IP addresses.
- Creating a VPC
    - Create the VPC itself
    - Create the subnets
    - Create the route tables
        - Add subnet associations
        - Add routes
            - Need to add route to internet and internet gateway.
    - Create the internet gateway
        - Create the gateway
        - Attach it to the VPC
- Elastic IPs are IPs that can be moved from one instance to another.
- Elastic network interface is a MAC address that can be moved from one instance to another.
- Network Address Translation (NAT)
    - NAT Gateway
        - Allow instances on private subnets to access the internet.
- Network ACLs
    - Stateless - have to allow inbound and outbound traffic explicitly.
    - Apply to subnets
- Security groups
    - Stateful
    - Apply to individual instances
    - Can chain security groups.

# Module 4 - Computing

- EC2
    - Amazon machine image (AMI)
    - EC2 Image Builder - can dynamically build images
    - AWS Compute Optimizer - can make suggestions for instance type
