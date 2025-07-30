# Factorio Server on AWS with CloudFormation and S3 Persistence
This repository provides an automated solution for deploying a Factorio multiplayer server on AWS EC2 using CloudFormation. Game saves are stored in S3 for durability and easy recovery. The setup is cost-effective, leverages AWS Free Tier resources, and is ideal for hobbyists or small groups.
## How to Set it Up
- Create a new S3 Bucket or use an existing S3 bucket which stores your save file for persistence.
- Before launching the stack, you must upload your `server-settings.json` file to the S3 bucket
you specify in the CloudFormation parameters. The value of `autosave_slots` in your
`server-settings.json` **must be set to 1**. 
This ensures that only a single autosave file(`_autosave1.zip`) is managed and backed up to S3. 
Using more than one autosave slot will break the backup logic and may result in lost saves.
  - For a simple setup you can use [server-settings/server-settings.json](server-settings/server-settings.json)
  - To upload the file:
    - Go to your S3 bucket in the AWS Console.
    - Upload your customized `server-settings.json` to the root of the bucket.
    - Ensure the file is named exactly `server-settings.json`.
- Deploy the CloudFormation template (`factorio-server-template.yaml`) in your AWS account.
  - Rundown on how to deploy the CloudFormation Template via AWS Console:
    - Log in to your AWS Management Console.
    - Navigate to the CloudFormation service.
    - Click "Create stack" and select "With new resources (standard)".
    - Under "Specify template", choose "Upload a template file".
    - Click "Choose file" and select your `factorio-server-template.yaml` from your local machine.
    - Click "Next".
    - Fill in the required parameters (e.g., S3 bucket name, instance type, backup interval, SSH CIDR IP). 
      - See the [Parameters](#parameters) section below for details on each option.
    - Click "Next" and configure stack options as needed (tags, permissions, etc.).
    - Click "Next" and review your settings.
    - Click "Create stack" to start the deployment.
    - Wait for the stack status to change to "CREATE_COMPLETE". You can then find the public IP and port in the stack outputs.
### Costs
Minimal to no costs if you are eligible for the AWS Free Tier
The default instance type (`t2.micro`) and S3 usage are covered by the Free Tier for most new AWS accounts.
### Persistence
The S3 bucket used for persistence in this setup is designed to store a single game save file (`latest.zip`). This means each bucket can only reliably hold one Factorio game/server save at a time. If you want to run and persist multiple different games or servers, you must create a separate S3 bucket for each save/server you wish to keep. This ensures that backups and restores work correctly and prevents overwriting or mixing up saves between different games.

### Disclaimer 
The author is not responsible for any AWS costs that may apply. Always check your AWS billing dashboard and Free Tier eligibility before deploying.

---

## Technical Documentation (Step-by-Step)

This CloudFormation template automates the deployment of a Factorio multiplayer server on AWS EC2, with game saves stored in S3 for durability and portability. Below is a detailed explanation of each step and resource, including why it is necessary:

###  Parameters
- **BucketName**: The name of an existing S3 bucket where Factorio save files will be stored. This enables persistent, cloud-based storage for game saves, so you can recover or migrate your server easily.
- **InstanceType**: The EC2 instance type for the Factorio server. Default is `t2.micro`, which is cost-effective for small games. This parameter allows you to scale up for larger games if needed.
- **BackupInterval**: The interval (in minutes) for automatic save file backups to S3. Default is `10`. This allows you to adjust how frequently your game state is saved to S3.
- **SshCidrIp**: The CIDR IP range allowed to SSH into the server. Default is `0.0.0.0/0` (all IPs), but you can restrict this for better security.

###  IAM Role and Instance Profile
- **FactorioRole**: Grants the EC2 instance permissions to access the specified S3 bucket. This is required so the server can download the latest save file at startup and upload backups during operation.
  - **AssumeRolePolicyDocument**: Allows EC2 to assume this role.
  - **Policies**: Grants `s3:GetObject`, `s3:PutObject`, and `s3:ListBucket` permissions for the bucket and its contents. This ensures the server can read, write, and list saves in S3.
- **FactorioInstanceProfile**: Associates the IAM role with the EC2 instance, so the permissions are available to the OS and AWS CLI.

###  Security Group
- **FactorioSecurityGroup**: Controls network access to the EC2 instance.
  - **UDP port 34197**: Required for Factorio multiplayer traffic. Open to the world so players can connect.
  - **TCP port 22**: SSH access for administration. The allowed IP range is configurable via the `SshCidrIp` parameter (default: all IPs, but should be restricted for security reasons).

###  EC2 Instance
- **ImageId**: Uses the latest stable Ubuntu 24.04 AMI from SSM Parameter Store. This ensures the server is always up-to-date and avoids dependency issues with older AMIs.
- **InstanceType**: Uses the value from the parameter, allowing flexibility in server performance and cost.
- **IamInstanceProfile**: Attaches the IAM role for S3 access.
- **SecurityGroupIds**: Attaches the security group for network access.
- **UserData**: Bootstraps the server with the following steps:
  1. **System Update and Dependency Installation**: Runs `apt update` and installs required packages (`wget`, `unzip`, `curl`, `tmux`). These are needed for downloading, extracting, and running Factorio, and for installing AWS CLI.
  2. **AWS CLI Installation**: Downloads and installs the latest AWS CLI. This is necessary for interacting with S3 from the instance.
  3. **Factorio Installation**: Downloads the latest Factorio headless server, extracts it, and prepares the directory structure. This sets up the game server software.
  4. **Save File Handling**: Checks S3 for an existing save file (`latest.zip`). If found, downloads it; if not, creates a new save. This ensures the server always starts with the latest available game state.
  5. **Server Settings**: Optionally downloads server settings from S3 if present, allowing custom configuration.
  6. **Automated Backups**: Adds a cron job to upload the autosave file to S3 every 10 minutes. This provides regular, automated backups for disaster recovery and migration.
  7. **Systemd Service Setup**: Creates a systemd service to run Factorio as a background service. This ensures the server starts on boot, restarts on failure, and is managed by the OS (no need for tmux or manual intervention).
  8. **Service Activation**: Reloads systemd, enables the Factorio service, and starts it. This brings the server online and ensures it stays running.

###  Outputs
- **InstancePublicIP**: The public IP address of the EC2 instance, used to connect to the Factorio server.
- **FactorioPort**: The UDP port for Factorio multiplayer (34197).

---

## Why This Is a Good and Cost-Effective Approach

**Advantages:**
- **Low Cost:** Uses AWS Free Tier resources (t2.micro EC2, S3) for most new accounts, minimizing expenses for small or personal servers.
- **Pay-as-You-Go:** Only pay for what you use. You can scale up or down as needed, and stop the server when not in use to save costs.
- **No Custom AMI Needed:** Always uses the latest Ubuntu image from AWS, so you avoid maintenance and compatibility issues with custom images. Also there are no Snapshot costs for keeping the saves
- **Automated Backups:** Game saves are regularly uploaded to S3, protecting against data loss and making recovery easy.
- **Easy Recovery and Migration:** You can redeploy the server in any region and restore the latest save from S3, making it portable and resilient.
- **Minimal Manual Setup:** Everything is automated via CloudFormation and systemd, reducing setup time and human error.
- **Configurable:** Instance type, backup interval, and S3 bucket are parameters, so you can optimize for your needs and budget.
- **Security Options:** You can restrict SSH access to your IP for better security.

**Considerations:**
- **S3 bucket must exist:** The template does not create the bucket; you must provide one.
- **Performance:** Default instance type (`t2.micro`) is cost-effective but may be underpowered for large games. You can choose a larger instance if needed.
- **Security Defaults:** SSH is open to the world by default (should restrict for production).

This approach is ideal for hobbyists, small groups, and anyone looking for a reliable, low-cost Factorio server with cloud-based persistence and easy management.

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.


