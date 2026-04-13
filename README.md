# EpicBook Ansible Deployment

This project contains Ansible playbooks and roles for automated deployment of the EpicBook web application to Azure infrastructure.

## Overview

EpicBook is a web application that is built with Node.js, using PM2 as a process manager, and served through Nginx as a reverse proxy. The application connects to a MySQL database hosted on Azure Database for MySQL.

This Ansible project automates the deployment process across web servers, handling system preparation, Nginx configuration, and application deployment. The EpicBook application source code is maintained in a separate repository (infra-epicbook) on GitHub, while this repository contains the infrastructure-as-code for deploying and managing the application environment.

The included Azure DevOps pipeline (`azure-pipelines.yml`) provides continuous deployment capabilities, automatically deploying infrastructure changes and application updates when code is pushed to the main branch.

## Prerequisites

Before running this Ansible playbook, ensure you have:

- **Ansible** (version 2.9 or later)
- **SSH access** to target servers
- **Azure infrastructure** set up with:
  - Ubuntu-based web server
  - Azure Database for MySQL instance
- **Git** for cloning the application repository
- **Python 3** on target servers

## Project Structure

This repository contains the infrastructure-as-code for deploying EpicBook. The actual application code is maintained in the separate [infra-epicbook](https://github.com/pravinmishraaws/theepicbook.git) repository.

```
.
├── ansible.cfg                 # Ansible configuration
├── azure-pipelines.yml         # Azure DevOps CI/CD pipeline for automated deployment
├── inventory.ini              # Ansible inventory file defining target servers
├── site.yml                   # Main Ansible playbook orchestrating the deployment
├── group_vars/
│   └── web/
│       └── main.yml           # Variables for web servers (app repo, DB config, etc.)
└── roles/
    ├── common/                # System preparation tasks
    │   ├── handlers/
    │   └── tasks/
    ├── nginx/                 # Nginx installation and configuration
    │   ├── handlers/
    │   ├── tasks/
    │   └── templates/
    │       └── epicbook.conf.j2
    └── epicbook/              # Application deployment from GitHub repo
        ├── handlers/
        └── tasks/
```

## Setup and Installation

### 1. Clone this repository

```bash
git clone <repository-url>
cd app-epicbook
```

### 2. Configure Inventory

Edit `inventory.ini` to match your target servers:

```ini
[web]
your-server-ip

[web:vars]
ansible_user=your-ssh-user
ansible_password=your-ssh-password
ansible_connection=ssh
ansible_ssh_common_args='-o StrictHostKeyChecking=no'
ansible_python_interpreter=/usr/bin/python3
```

### 3. Configure Variables

Update `group_vars/web/main.yml` with your specific values:

```yaml
app_repo: "https://github.com/your-org/your-epicbook-repo.git"
app_dest: "/var/www/epicbook"
app_user: "www-data"
app_group: "www-data"
nginx_site_name: "epicbook"
app_port: 3000
db_user: "your-db-user"
db_name: "your-database"
db_host: "your-mysql-server.mysql.database.azure.com"
db_password: "your-db-password"
pm2_home: "/var/www/.pm2"
```

### 4. Run the Playbook

Execute the Ansible playbook:

```bash
ansible-playbook -i inventory.ini site.yml
```

For production deployments with SSH keys:

```bash
ansible-playbook -i inventory.ini site.yml --private-key /path/to/your/private/key
```

## Usage

### Manual Deployment

After configuration, run the playbook to deploy:

```bash
ansible-playbook -i inventory.ini site.yml -v
```

### CI/CD Deployment

This project includes an Azure DevOps pipeline (`azure-pipelines.yml`) that automates the deployment of the EpicBook application to Azure infrastructure. The pipeline is designed to work with the infra-epicbook repository already hosted on GitHub.

#### Pipeline Overview

The Azure DevOps pipeline provides a complete automated deployment workflow:

1. **Trigger**: Automatically triggers on pushes to the `main` branch of this repository
2. **Environment**: Runs on a self-hosted agent pool (`SelfHostedPool`)
3. **Ansible Installation**: Installs Ansible on the build agent
4. **SSH Key Setup**: Downloads and configures the SSH private key for secure server access
5. **Connectivity Testing**: Verifies Ansible can connect to target servers using ping
6. **Deployment Execution**: Runs the complete Ansible playbook to deploy the application
7. **Health Verification**: Tests that the deployed application is accessible via HTTP

#### Pipeline Stages

**Configure Stage - AnsibleDeploy Job:**
- **Checkout**: Retrieves the latest code from this repository
- **Secure File Download**: Downloads the SSH private key (`epicbook_key`) from Azure DevOps secure files
- **Ansible Setup**: Updates system packages and installs Ansible
- **SSH Configuration**: Sets up SSH key with proper permissions for deployment
- **Inventory Verification**: Displays inventory and group variables for validation
- **Connectivity Test**: Uses `ansible ping` to verify SSH connectivity to target servers
- **Playbook Execution**: Runs `ansible-playbook site.yml` with verbose output
- **Application Verification**: Tests HTTP accessibility of the deployed application

#### Pipeline Benefits

- **Automated Deployment**: Eliminates manual deployment steps and reduces human error
- **Consistent Environment**: Ensures deployments are identical across environments
- **Security**: Uses secure file storage for SSH keys and credentials
- **Monitoring**: Includes health checks to verify successful deployment
- **Integration**: Works seamlessly with the EpicBook application repository on GitHub

#### Pipeline Configuration

The pipeline uses several key configurations:
- **Self-hosted agents**: Allows for custom build environments with necessary tools
- **Secure file handling**: Protects sensitive SSH keys during deployment
- **Environment variables**: Disables host key checking for automated deployments
- **Working directory**: Ensures all commands run from the correct project directory

To use this pipeline, ensure your Azure DevOps project has:
- A self-hosted agent pool named `SelfHostedPool`
- The SSH private key uploaded as a secure file named `epicbook_key`
- Proper permissions for the pipeline to access secure files

### Verification

After deployment, verify the application is running:

```bash
# Check if Nginx is serving
curl -s -o /dev/null -w '%{http_code}' http://your-server-ip

# Check application logs
ansible web -i inventory.ini -m shell -a "pm2 logs epicbook --lines 50"
```

## Roles Overview

### Common Role
- Updates system packages
- Installs essential packages (git, curl, etc.)
- Configures system settings

### Nginx Role
- Installs and configures Nginx
- Sets up reverse proxy configuration for the Node.js application
- Enables and starts the Nginx service

### EpicBook Role
- Creates application directory structure
- Clones the EpicBook application from the GitHub repository (infra-epicbook)
- Installs Node.js dependencies via npm
- Configures PM2 process management for the Node.js application
- Starts the application and ensures it's running

## Configuration Details

### Nginx Configuration
The Nginx role uses a Jinja2 template (`epicbook.conf.j2`) that:
- Listens on port 80
- Proxies requests to `http://localhost:3000`
- Handles WebSocket upgrades for real-time features

### Application Configuration
The EpicBook application is configured to:
- Run on port 3000
- Use PM2 for process management
- Connect to Azure MySQL database
- Serve static files through Nginx

## Troubleshooting

### Common Issues

1. **SSH Connection Failed**
   - Verify SSH key permissions (`chmod 600 keyfile`)
   - Check firewall rules on target server
   - Ensure SSH service is running

2. **Ansible Ping Fails**
   - Verify Python 3 is installed on target
   - Check `ansible_python_interpreter` setting
   - Ensure user has sudo privileges

3. **Application Not Starting**
   - Check PM2 logs: `pm2 logs`
   - Verify database connectivity
   - Check Node.js dependencies installation

4. **Nginx Not Serving**
   - Check Nginx status: `sudo systemctl status nginx`
   - Verify configuration: `sudo nginx -t`
   - Check error logs: `sudo tail -f /var/log/nginx/error.log`

### Debug Mode

Run playbook with verbose output:

```bash
ansible-playbook -i inventory.ini site.yml -vvv
```

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test thoroughly
5. Submit a pull request

## License

This project is licensed under the MIT License - see the LICENSE file for details.