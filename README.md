# VMware CD-ROM Detachment Automation

An enterprise-grade Ansible playbook for identifying and detaching CD-ROM drives from VMware virtual machines, designed for execution within Ansible Automation Platform 2.5 using Red Hat certified collections.

## 🎯 **Features**

- **Automated Discovery**: Scans all virtual machines in vCenter to identify CD-ROM devices
- **Selective Detachment**: Safely detaches CD-ROM drives from identified virtual machines  
- **Multiple Execution Modes**: Supports dry-run, report-only, and execute modes
- **Enterprise Integration**: Full AAP 2.5 workflow compatibility with artifact generation
- **Comprehensive Logging**: Detailed execution reports and status tracking
- **Report Generation**: Creates detailed reports and transfers them to specified servers
- **Error Handling**: Robust error handling with graceful failure management
- **Red Hat Certified**: Uses only Red Hat certified and validated collections

## 📋 **Prerequisites**

- Ansible Automation Platform 2.5 or later
- VMware vCenter Server 6.7 or later
- Valid vCenter credentials with VM management permissions
- Red Hat certified `vmware.vmware_rest` collection (v2.3.1+)

## 🚀 **Quick Start**

### AAP 2.5 Setup

1. **Import Project**: Import this repository into AAP as a new project
2. **Install Collections**: Ensure `vmware.vmware_rest` collection is available
3. **Create Credentials**: Set up VMware credential type with vCenter connection details
4. **Configure Inventory**: Create inventory with localhost in AAP web UI
5. **Create Job Template**: Configure job template with appropriate settings

### Local Development

```bash
# Install required collections
ansible-galaxy collection install -r requirements.yml

# Run syntax check
ansible-navigator run playbooks_vmware/detach_cdrom.yml --syntax-check

# Execute in check mode (dry-run)
ansible-navigator run playbooks_vmware/detach_cdrom.yml --check

# Execute with custom vCenter
ansible-navigator run playbooks_vmware/detach_cdrom.yml -e "vcenter_hostname=vcenter.example.com"
```

## 📖 **Usage**

### Execution Modes

The playbook supports three distinct execution modes:

#### 1. **Execute Mode** (Default)
```yaml
# Detaches CD-ROM drives from all identified VMs (default mode)
report_only: false
```

#### 2. **Check Mode** (Ansible Native)
```bash
# Identifies VMs and shows what would be changed without making modifications
# Use AAP Job Template "Check" option or --check flag
ansible-navigator run playbooks_vmware/detach_cdrom.yml --check
```

#### 3. **Report Only Mode**  
```yaml
# Scans and reports VMs with CD-ROM devices for planning purposes
report_only: true
```

### Required Variables

| Variable | Description | Example | Source |
|----------|-------------|---------|---------|
| `vcenter_hostname` | vCenter server FQDN/IP | `vcenter.example.com` | AAP Credential |
| `vcenter_username` | vCenter username | `administrator@vsphere.local` | AAP Credential |
| `vcenter_password` | vCenter password | `SecurePassword123!` | AAP Credential |

### Optional Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `vcenter_validate_certs` | `true` | Validate SSL certificates |
| `report_only` | `false` | Generate report without changes |
| `report_server_user` | `ansible` | SSH username for report server |
| `report_server_key` | `(default SSH key)` | SSH private key path for report server |

**Note**: For dry-run functionality, use Ansible's native `--check` mode instead of a custom variable.

## 🔧 **AAP 2.5 Integration**

### Credential Configuration

Create a custom credential type with the following fields:

```yaml
# Input Configuration
fields:
  - id: vcenter_hostname
    type: string
    label: vCenter Hostname
  - id: vcenter_username  
    type: string
    label: vCenter Username
  - id: vcenter_password
    type: string
    label: vCenter Password
    secret: true

# Injector Configuration  
extra_vars:
  vcenter_hostname: '{{ vcenter_hostname }}'
  vcenter_username: '{{ vcenter_username }}'
  vcenter_password: '{{ vcenter_password }}'
```

### Job Template Configuration

```yaml
Name: VMware CD-ROM Detachment
Inventory: [Inventory with localhost]
Project: [This Project]
Playbook: playbooks_vmware/detach_cdrom.yml
Credentials: [VMware Credential]
Variables:
  vcenter_validate_certs: true
  report_only: false
  report_server_user: ansible
  # report_server_key: /path/to/key (optional)
Options:
  - Enable "Check" for dry-run mode
```

### Inventory Setup

Create an inventory in AAP with the following hosts:
```yaml
localhost:
  ansible_connection: local
  ansible_python_interpreter: "{{ ansible_playbook_python }}"

server1.test.com:
  ansible_user: "{{ report_server_user | default('ansible') }}"
  ansible_ssh_private_key_file: "{{ report_server_key | default(omit) }}"
```

**Note**: Ensure `server1.test.com` is accessible via SSH and the specified user has write permissions to `/tmp`.

### Survey Configuration (Optional)

Add survey questions for runtime customization:

```yaml
- variable: report_only
  question: "Report only mode (identify VMs)?"
  type: boolean 
  default: false

- variable: vcenter_validate_certs
  question: "Validate SSL certificates?"
  type: boolean
  default: true
```

### Workflow Integration

The playbook generates workflow artifacts via `set_stats` for downstream consumption:

```yaml
# Available Artifacts
cdrom_detachment_report: # Complete execution report
vms_processed_count: # Total VMs scanned  
vms_with_cdrom_count: # VMs with CD-ROM devices
total_cdroms_detached: # Successfully detached count
execution_mode: # Mode used (execute/check_mode/report_only)
execution_timestamp: # ISO 8601 timestamp
```

## 📄 **Report Generation**

The playbook automatically generates detailed reports and transfers them to `server1.test.com:/tmp/`. 

### Report Features
- **Jinja2 templating**: Clean, maintainable template-based report generation
- **Timestamped filenames**: `vmware_cdrom_detachment_YYYY-MM-DD_HH-MM-SSZ.txt`
- **Comprehensive details**: VM information, CD-ROM device details, operation results
- **Error tracking**: Detailed failure information and troubleshooting data
- **Backup creation**: Automatic backup of existing reports

### Report Contents
- Execution summary with timestamps and mode
- Complete VM inventory with CD-ROM device details
- Detailed results of all detachment operations
- Error details for failed operations
- Power state and configuration information

### Report Security
- Uses SSH key-based authentication
- Configurable SSH user (default: `ansible`)
- File permissions set to 0644
- Automatic cleanup of temporary local files

### Report Customization
- **Template-based**: Modify `templates/cdrom_detachment_report.j2` to customize report format
- **Flexible formatting**: Full Jinja2 templating capabilities for custom layouts
- **Maintainable**: Separate template file for easy updates and version control
- **Reusable**: Template can be shared across multiple playbooks

## 📊 **Output Examples**

### Successful Execution
```
TASK [Final execution summary] ***
ok: [localhost] => {
    "msg": [
        "=== VMware CD-ROM Detachment Playbook Complete ===",
        "Execution Mode: EXECUTE",
        "Total VMs Scanned: 25",
        "VMs with CD-ROM: 8", 
        "Total CD-ROM Devices Found: 12",
        "Successful Detachments: 12",
        "Failed Detachments: 0",
        "Timestamp: 2024-01-15T14:30:25Z"
    ]
}
```

### Check Mode
```
TASK [Display check mode information] ***
ok: [localhost] => {
    "msg": [
        "CHECK MODE: No changes will be made",
        "8 VMs would have CD-ROM devices detached", 
        "Total CD-ROM devices that would be detached: 12"
    ]
}
```

### Report Transfer Status
```
TASK [Display report transfer status] ***
ok: [localhost] => {
    "msg": [
        "Report successfully generated and transferred",
        "Remote location: server1.test.com:/tmp/vmware_cdrom_detachment_2024-01-15_14-30-25Z.txt",
        "Report contains detailed execution results and VM information"
    ]
}
```

## 🔍 **Troubleshooting**

### Common Issues

#### Connection Failures
```bash
# Verify vCenter connectivity
curl -k https://vcenter.example.com/ui/

# Check credential configuration in AAP
# Ensure vcenter_hostname, vcenter_username, vcenter_password are set
```

#### SSL Certificate Issues
```yaml
# Disable certificate validation for testing
vcenter_validate_certs: false
```

#### Permission Errors
```
Required vCenter Privileges:
- Virtual machine.Configuration.Modify device settings
- Virtual machine.Inventory.Modify existing  
- System.View (on vCenter root)
```

#### Collection Issues
```bash
# Verify collection installation
ansible-galaxy collection list vmware.vmware_rest

# Install/upgrade collection
ansible-galaxy collection install vmware.vmware_rest --upgrade
```

#### Report Server Issues
```bash
# Test SSH connectivity to report server
ssh ansible@server1.test.com "ls -la /tmp/"

# Check SSH key permissions (if using custom key)
chmod 600 /path/to/private/key

# Verify write permissions on target directory
ssh ansible@server1.test.com "touch /tmp/test_file && rm /tmp/test_file"
```

```yaml
# Disable report server functionality temporarily
- name: Skip report transfer on connection issues
  when: false  # Add this condition to report transfer tasks
```

### Debug Mode

Enable verbose output for troubleshooting:

```bash
# Local execution with debug
ansible-navigator run playbooks_vmware/detach_cdrom.yml -vvv

# AAP Job Template: Add to Extra Variables
debug_mode: true
```

## 🏗️ **Architecture**

### Collection Dependencies
- `vmware.vmware_rest`: Red Hat certified VMware REST API collection
- `ansible.builtin`: Core Ansible modules (included with AAP)

### Files Structure
- `playbooks_vmware/detach_cdrom.yml`: Main playbook for CD-ROM detachment
- `templates/cdrom_detachment_report.j2`: Jinja2 template for report generation
- `requirements.yml`: Collection dependencies
- `ansible.cfg`: Configuration file (for local development)
- `README.md`: Documentation

### Execution Flow
1. **Validation**: Verify required connection parameters
2. **Authentication**: Establish vCenter session
3. **Discovery**: Retrieve all virtual machines from vCenter
4. **Analysis**: Identify VMs with attached CD-ROM devices
5. **Processing**: Detach CD-ROM devices (unless dry-run/report-only)
6. **Reporting**: Generate comprehensive execution report
7. **Artifacts**: Save results for workflow integration

### Security Considerations
- Uses AAP credential management for secure parameter handling
- Supports SSL certificate validation (configurable)
- Implements least-privilege access patterns
- Encrypts sensitive data in transit via HTTPS/REST API

## 🔄 **Development**

### Testing
```bash
# Syntax validation
ansible-navigator run playbooks_vmware/detach_cdrom.yml --syntax-check

# Lint checking (requires ansible-lint)
ansible-lint playbooks_vmware/detach_cdrom.yml

# Check mode testing (dry-run)
ansible-navigator run playbooks_vmware/detach_cdrom.yml --check
```

### Contributing
1. Follow Red Hat Ansible best practices
2. Maintain idempotent operations
3. Update documentation for new features
4. Test with multiple vCenter versions
5. Ensure AAP 2.5 compatibility

## 📝 **License**

This project follows enterprise automation standards and is designed for use within Red Hat Ansible Automation Platform environments.

## 🆘 **Support**

For issues and questions:
1. Review troubleshooting section
2. Check AAP 2.5 documentation
3. Verify VMware collection documentation
4. Review execution logs and artifacts

---

**Note**: This playbook is optimized for enterprise environments using Red Hat certified collections and Ansible Automation Platform 2.5. For community environments, consider using the `community.vmware` collection with appropriate modifications.