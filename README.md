
# Angular Application Deployment to Azure VMs

This repository contains a GitHub Actions workflow for automated deployment of an Angular application to multiple Azure Virtual Machines using SSH and PM2 process management.

## Overview

The workflow automates the complete deployment pipeline, including code transfer, dependency installation, building, and process management across multiple Azure VMs in parallel. It uses PM2 to ensure the application runs continuously and restarts automatically after system reboots.

## Architecture

### Deployment Flow

1. **Code Checkout**: GitHub Actions checks out the latest code from the repository
2. **Parallel Deployment**: Code is deployed simultaneously to multiple VMs (max 2 at a time)
3. **File Transfer**: Source code is copied to each VM via SCP
4. **Build Process**: Dependencies are installed and the Angular app is built on each VM
5. **Process Management**: PM2 manages the application lifecycle
6. **Health Checks**: Automated verification ensures successful deployment
7. **Notification**: Summary of deployment status across all VMs

### Key Components

- **GitHub Actions**: Orchestrates the CI/CD pipeline
- **SCP (Secure Copy)**: Transfers files to VMs
- **SSH**: Executes remote commands on VMs
- **PM2**: Node.js process manager for application lifecycle
- **Angular Dev Server**: Serves the application on port 4200

## Prerequisites

### Azure Infrastructure

- **2 Azure Virtual Machines** running Ubuntu/Debian
- **SSH access** enabled on both VMs
- **Public IP addresses** assigned to each VM
- **Firewall rules** allowing:
  - Port 22 (SSH)
  - Port 4200 (Angular dev server)

### VM Requirements

Each VM must have:
- **Node.js** version 24.x installed
- **npm** (comes with Node.js)
- **PM2** installed globally: `npm install -g pm2`
- **curl** installed (for health checks)
- **systemd** (for PM2 startup scripts)

<img width="381" height="148" alt="image" src="https://github.com/user-attachments/assets/153f6822-e6e0-40b6-8857-73872a9055d6" />

### GitHub Repository Secrets

Configure the following secrets in your GitHub repository (Settings → Secrets and variables → Actions):

#### VM1 Secrets
- `VM1_PUBLIC_IP`: Public IP address of the first VM
- `VM1_USERNAME`: SSH username for VM1
- `VM1_SSH_PRIVATE_KEY`: Private SSH key for VM1 authentication

#### VM2 Secrets
- `VM2_PUBLIC_IP`: Public IP address of the second VM
- `VM2_USERNAME`: SSH username for VM2
- `VM2_SSH_PRIVATE_KEY`: Private SSH key for VM2 authentication

## Configuration

### Environment Variables

The workflow uses the following environment variables (configured in the `env` section):

```yaml
NODE_VERSION: '24'        # Node.js version
APP_NAME: 'ts-app'        # PM2 process name
APP_PORT: 4200            # Angular dev server port
REPO_PATH: 'app'          # Target directory on VMs
```

### Customization

To adapt this workflow for your needs:

1. **Change Node.js version**: Modify `NODE_VERSION` variable
2. **Update application name**: Change `APP_NAME` to your preferred process name
3. **Modify port**: Update `APP_PORT` if using a different port
4. **Change deployment path**: Adjust `REPO_PATH` for different target directories
5. **Add more VMs**: Extend the `matrix.vm` array with additional VM configurations

## Workflow Triggers

The deployment runs automatically when:

- **Push to main branch**: Any commit pushed to `main` triggers deployment
- **Manual trigger**: Use "Run workflow" button in GitHub Actions tab

## Deployment Process

### Job 1: Deploy (Matrix Strategy)

Runs in parallel across all defined VMs with a maximum of 2 concurrent deployments.

#### Step 1: Checkout Code
Uses `actions/checkout@v4` to clone the repository.

#### Step 2: Copy Code to VM
- **Action**: `appleboy/scp-action@v0.1.7`
- **Purpose**: Transfers all repository files to the VM
- **Options**:
  - `rm: true`: Removes existing files before copying
  - `overwrite: true`: Overwrites existing files
  - `target`: Copies to `~/app` directory on VM

#### Step 3: Build and Deploy
- **Action**: `appleboy/ssh-action@v1.0.0`
- **Timeout**: 20 minutes
- **Operations**:
  1. Navigates to application directory
  2. Installs npm dependencies with `npm install`
  3. Builds the application with `npm run build`
  4. Stops existing PM2 process (if running)
  5. Deletes old PM2 process configuration
  6. Starts new PM2 process with Angular dev server
  7. Configures PM2 to bind to `0.0.0.0` (accessible externally)
  8. Saves PM2 configuration
  9. Sets up PM2 to auto-start on system boot
  10. Waits 20 seconds for Angular compilation
  11. Displays PM2 status and application logs
  12. Verifies deployment success

#### Step 4: Health Check
- Runs only if deployment succeeds
- Waits 5 seconds for application stabilization
- Sends HTTP request to `http://localhost:4200`
- Reports health status
- Shows final PM2 process status

### Job 2: Notify

Runs after all VM deployments complete (success or failure).

- Displays deployment summary
- Shows commit SHA and branch name
- Exits with error code if any deployment failed

## PM2 Configuration

### Process Management

The workflow uses PM2 with the following configuration:

```bash
pm2 start npm \
  --name ts-app \
  --interpreter none \
  -- run start -- --host 0.0.0.0
```

**Explanation**:
- `start npm`: Runs npm as the process
- `--name ts-app`: Names the process "ts-app"
- `--interpreter none`: Treats npm as a binary (not a script)
- `-- run start`: Executes `npm run start`
- `-- --host 0.0.0.0`: Passes `--host 0.0.0.0` to Angular CLI

### Auto-Restart on Boot

The workflow configures PM2 to start automatically on system reboot:

```bash
pm2 startup systemd -u <username> --hp /home/<username>
pm2 save --force
```

This ensures the Angular application restarts automatically if the VM reboots.

## Error Handling

### Fail-Fast Disabled
```yaml
fail-fast: false
```
If one VM deployment fails, other VMs continue deploying.

### Command Error Handling
```bash
set -e
```
Bash script exits immediately if any command fails.

### Graceful Process Stops
```bash
pm2 stop $APP_NAME 2>/dev/null || true
pm2 delete $APP_NAME 2>/dev/null || true
```
Uses `|| true` to prevent errors if processes don't exist.

### Deployment Verification
The workflow checks PM2 status and exits with error code 1 if the application isn't running.

## Monitoring and Debugging

### View Logs in GitHub Actions
1. Navigate to **Actions** tab in GitHub
2. Click on the workflow run
3. Expand VM deployment steps to view logs

### View Logs on VM

SSH into the VM and run:

```bash
# View real-time logs
pm2 logs ts-app

# View last 100 lines
pm2 logs ts-app --lines 100

# View only error logs
pm2 logs ts-app --err

# Monitor process status
pm2 monit
```
<img width="1532" height="680" alt="image" src="https://github.com/user-attachments/assets/795924d6-d3de-4b44-b8a9-e14294ee7b0a" />

### Check Application Status

```bash
# List all PM2 processes
pm2 status

# Get detailed process info
pm2 show ts-app

# Check if app is responding
curl http://localhost:4200
```

### Common Issues

**Application won't start**:
- Check Node.js version: `node --version`
- Verify npm dependencies: `cd ~/app && npm install`
- Check PM2 logs: `pm2 logs ts-app --err`

**Port already in use**:
- Check what's using port 4200: `sudo lsof -i :4200`
- Kill the process or use a different port

**SSH connection fails**:
- Verify VM is running in Azure Portal
- Check firewall rules allow port 22
- Validate SSH key format in GitHub secrets

**Build fails**:
- Ensure VM has sufficient disk space: `df -h`
- Check memory availability: `free -h`
- Review build logs in GitHub Actions

## Security Considerations

1. **SSH Keys**: Use dedicated SSH keys for deployment, not personal keys
2. **Secret Management**: Never commit secrets to the repository
3. **VM Access**: Restrict SSH access to specific IP ranges using Azure NSG
4. **Firewall**: Only open necessary ports (22, 4200)
5. **User Permissions**: Use non-root user for deployments
6. **Key Rotation**: Regularly rotate SSH keys and update GitHub secrets

## Performance Optimization

### Parallel Deployments
```yaml
max-parallel: 2
```
Deploys to 2 VMs simultaneously to reduce total deployment time.

### Build Caching
Consider adding npm caching to speed up dependency installation:

```yaml
- name: Cache npm dependencies
  uses: actions/cache@v3
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
```

### Production Build
For production deployments, modify the build command:

```bash
npm run build -- --configuration production
```

## Scaling

### Adding More VMs

To deploy to additional VMs, add entries to the matrix:

```yaml
matrix:
  vm:
    - name: VM1
      host_secret: VM1_PUBLIC_IP
      username_secret: VM1_USERNAME
      key_secret: VM1_SSH_PRIVATE_KEY
    - name: VM2
      host_secret: VM2_PUBLIC_IP
      username_secret: VM2_USERNAME
      key_secret: VM2_SSH_PRIVATE_KEY
    - name: VM3
      host_secret: VM3_PUBLIC_IP
      username_secret: VM3_USERNAME
      key_secret: VM3_SSH_PRIVATE_KEY
```

Remember to add corresponding secrets to GitHub.

### Load Balancing

For production environments, consider adding:
- Azure Load Balancer to distribute traffic
- Azure Application Gateway for advanced routing
- Health probes to monitor VM availability

## Maintenance

### Updating Dependencies

SSH into each VM and run:

```bash
cd ~/app
npm update
pm2 restart ts-app
```

### PM2 Updates

```bash
npm install -g pm2@latest
pm2 update
```

### Rollback

To rollback to a previous version:
1. Find the previous commit SHA in GitHub
2. Manually trigger workflow with that commit
3. Or SSH into VM and checkout previous code:

```bash
cd ~/app
git checkout <previous-commit-sha>
npm install
npm run build
pm2 restart ts-app
```
<img width="1896" height="579" alt="image" src="https://github.com/user-attachments/assets/aa904fdb-b3b7-4ce3-89fd-1e97b0fef023" />

<img width="1917" height="600" alt="image" src="https://github.com/user-attachments/assets/ffa634e4-418b-4a20-9434-b8079d19b2d7" />
## Contact 

For issues or questions or comment:
- Open an issue in GitHub
- yakubiliyas12@gmail.com
- Contact your DevOps team
  
