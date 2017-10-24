# Bootstrap

This section is used to bootstrap a new account, before any infrastructure exists.

This is currently a work in progress.

## Mandatory Manual Account Setup

### Configure the Account (root)

Login to the Account (root) should be rare after this procedure. The Account credentials should
be shared with very few people, as they allow complete and unrestricted access to everything in
the Account.

- Login as the Account (root), and configure the following:
  - Turn on Access to Billing Information
  - Configure the Security Questions
  - Create Account-level X509 Certificates and CloudFront Keys
  - Enable MFA on the Account
  - Configure the Password Policy for 20 characters, 1 upper, 1 lower, 1 number minimum
  - Customize the IAM users sign-in link with the account alias: dxcp
  - All of this information should be saved in the 1Password Team account or a similar
    method to share credentials across key team members.
- Logout as the Account (root)

### Configure the BootstrapAdministrators Group and bootstrapadministrator User

This User will be used (by default) to create the remainder of the infrastructure.

- Login as the Account (root)
  - Confirm the MFA is working properly
  - Create the BootstrapAdministrators Group
    - Attach the AdministratorAccess Policy
  - Create the bootstrapadministrator User
    - Allow Programmatic Access
    - Allow AWS Management Console access
    - Add as a member of the BootstrapAdministrators Group
  - Enable MFA on the bootstrapadministrator User
  - Create an ssh key, using a secure passphrase (this must be run on a UNIX command line):
    ```bash
    ssh-keygen -t rsa -b 4096 -C bootstrap@dxcp.technology -f ~/.ssh/dxcp_bootstrap_id_rsa
    ```
  - Import the bootstrap SSH public key as an EC2 KeyPair with the name "bootstrap"
    - Repeat this step in us-east-1, us-east-2, us-west-2, eu-west-2
  - Create an ssh key, using a secure passphrase (this must be run on a UNIX command line):
    ```bash
    ssh-keygen -t rsa -b 4096 -C bootstrapadministrator@dxcp.technology -f ~/.ssh/dxcp_bootstrapadministrator_id_rsa
    ```
  - Import the bootstrapadministrator SSH public key to the bootstrapadministrator User
    - This is under the User's Security credentials as "SSH keys for AWS CodeCommit"
- Logout as the Account (root)

### Configure the BootstrapUsers Group and bootstrapuser User

This User is intended only as a way to let new users get into the Bootstrap LinuxBastion for
initial setup, specifically to create their SSH key.

- Login as the bootstrapadministrator User
  - Confirm the MFA is working properly
  - Create the BootstrapUsers Group
    - Do not associate this Group with any Policies (only used for guest login)
  - Create the bootstrapuser User
    - Do **not** allow Programmatic Access
    - Do **not** allow AWS Management Console access
    - Add as a member of the BootstrapUsers Group
  - Create an ssh key, using a secure passphrase (this must be run on a UNIX command line):
    ```bash
    ssh-keygen -t rsa -b 4096 -C bootstrapuser@dxcp.technology -f ~/.ssh/dxcp_bootstrapuser_id_rsa
    ```
  - Import the bootstrapuser SSH public key to the bootstrapuser User
    - This is under the User's Security credentials as "SSH keys for AWS CodeCommit"
- Logout as the bootstrapadministrator User

## Optional Manual Account Setup

### Configure additional members of the BootstrapAdministrators Group

The following additional steps can be run manually to create additional administrators within the
BootstrapAdministrators Group.

- Create additional members of the BootstrapAdministrators Group, using the same process above
- Except, use each person's normal ssh key (i.e. dxcp_mcrawford_id_rsa) when importing an SSH
  key into the corresponding bootstrap<user>'s (i.e. bootstrapmcrawford) User SSH keys

## Raise Account Limits

Certain limits on the Account must be raised before you can build the complete architecture. This
list can change depending on modifications to the Instance Types and growth.

Initially, we will only raise limits for the Unstable MultiEnvironment to confirm the architecture
will build in the new Account. It's likely any re-use of this framework will modify the Applications
and InstanceTypes used by them, so further limit adjustments in the Stable MultiEnvironment should
be delayed until more is known about what needs to run there.

### Region: us-east-2 (Ohio)
The list below shows the limits raised to build the initial Unstable MultiEnvironment. These must
be raised via multiple Support Cases.

Case 1: Service VPC
- VPCs per Region: 8
- NAT Gateway per Availability Zone: 8
- Virtual Private Gateways per Region: 8

Case 2: Service Elastic IPs
- New VPC Elastic IP Address Limit: 32

Case 3: EC2 Instances
- Instance Limit: 20 t2.large
- Instance Limit: 20 t2.medium
- Instance Limit: 40 t2.small
- Instance Limit: 40 t2.micro
- Instance Limit: 40 t2.nano


## Register Marketplace AMIs
We're currently using the OpenVPN Access Server Marketplace AMI for secure remote access. Other Marketplace
AMIs may eventually be used. We must register use of these AMIs and accept terms before we're allowed to use them.

### OpenVPN Access Server
Browse to: https://aws.amazon.com/marketplace/search/results?x=0&y=0&searchTerms=OpenVPN+Access+Server to see the
list of all AMI variants. Then, for the following list, go into each, click Continue, Select the Manual Launch Tab,
Then click Accept Software Terms
- OpenVPN Access Server
- OpenVPN Access Server (10 Connected Devices)
- OpenVPN Access Server (25 Connected Devices)
- OpenVPN Access Server (50 Connected Devices)


## Secure Shell Workstation Setup

The following steps can be run manually on a new account to prepare for both the initial bootstrap
process and additional per-user bootstrap actions once the system is up.

- Create the bootstrap-dxcp S3 bucket. This will hold the initial RoyalTS Document containing
  the bootstrap session, then once created, the additional bastions.
- New users who have credentials can use this document, along with the free version of RoyalTS,
  for access to the system of a better quality than using PuTTY.

## Create Bootstrap Bastion

### Create the Bootstrap-LinuxBastion CloudFormation Stack

This Stack creates the Bootstrap-LinuxBastion, a server which can be used to create the remainder
of the infrastructure. This server does not need to exist once the infrastructure has been created.

There are two methods to do this.

1.  Use a Linux Host or Virtual Machine, which has the dxc-prototype project downloaded, and the
    bootstrapadministrator access keys and bootstrap ssh key installed in the appropriate locations.
    This is the easiest and most robust method, and should be used if possible.
2.  Create the Bootstrap-LinuxBastion CloudFormation Stack manually, by uploading the Template and
    setting all properties by hand. This method is easier in that you don't need to have any existing
    Linux workstation, but harder in that more steps have to be performed, and the risk of human
    error is considerably higher.

Creation of the Bootstrap Bastion is optional. It's an option if your workstation uses Windows.
If you are on a Mac or Linux Workstation, you can download and run the code direct from your
Workstation.

Once you have finished using the Bootstrap Bastion, you can delete the Stack which created it, or
you can update the Stack, changing the EnvironmentType to "standby", which will reduce the
AutoScaling Group which wraps it to 0, so nothing normally will run. This will allow you to again
update the Stack in the future, changing the EnvironmentType back to "small" (or larger), to start
using it again.

### Configure a DNS record to allow login to the Bootstrap Bastion

There should be an initial Domain created within Route53, named dxcp.technology. We need to create a
single new record with the Public IP Address of the Bootstrap Bastion, named bootstrap.dxcp.technology.

## Login to Bootstrap LinuxBastion as a member of the BootstrapAdministrators Group

The bootstrapadministrator User, or another member of the BootstrapAdministrators Group, can login
to the Bootstrap LinuxBastion with their own ssh key and username.
