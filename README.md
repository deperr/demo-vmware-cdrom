# VMware CD-ROM Drive Check Automation

This Ansible project provides automation for checking VMware vSphere virtual machines that have CD-ROM drives attached, designed for use with Ansible Automation Platform 2.5.

## Project Structure

```
demo-vmware-cdrom/
├── ansible.cfg                          # Ansible configuration with certified collections
├── ansible-navigator.yml                # Navigator configuration for EE execution
├── manage_cd_drives.yml                 # Main CD-ROM removal playbook (supports check mode)
├── collections/
│   └── requirements.yml                 # Collection dependencies (certified + community)
├── execution_environment/
│   ├── execution-environment.yml        # EE build configuration
│   ├── requirements.yml                 # Collection requirements for EE
│   ├── requirements.txt                 # Python dependencies (pyvmomi, etc.)
│   └── bindep.txt                      # System package dependencies
├── group_vars/
│   └── all.yml                         # VMware connection and configuration variables
├── templates/
│   ├── cdrom_removal_report.j2         # Template for CD-ROM removal operations
│   └── cdrom_report_table.j2           # Legacy template (not used by removal playbook)
└── reports/                            # Generated reports directory
```

## Features

- **AAP 2.5 Compatibility**: Designed for Ansible Automation Platform 2.5
- **Focused CD-ROM Removal**: Single playbook dedicated to removing mounted ISOs
- **Check Mode Support**: Use `--check` for safe dry-run operations
- **Dynamic Inventory**: Works with AAP's dynamic inventory generation
- **Secure Credentials**: Uses AAP's built-in credential management
- **Comprehensive Reporting**: Generates detailed reports in multiple formats
- **Modular Design**: Reusable variables and modular task structure
- **Module Defaults**: Clean code using Ansible module_defaults for connection parameters
- **Execution Environment First**: Relies on EE-provided collections for cleaner playbooks

## Configuration

### VMware Connection (group_vars/all.yml)

The playbook uses variables that integrate with AAP credentials:

```yaml
vcenter_hostname: "{{ vcenter_server | default('vcenter.example.com') }}"
vcenter_username: "{{ vcenter_user | default(ansible_user) }}"
vcenter_password: "{{ vcenter_pass | default(ansible_password) }}"
```

### Module Defaults

The playbook uses Ansible's `module_defaults` feature to avoid repeating connection parameters:

```yaml
module_defaults:
  group/vmware.vmware_rest.all:
    vcenter_hostname: "{{ vcenter_hostname }}"
    vcenter_username: "{{ vcenter_username }}"
    vcenter_password: "{{ vcenter_password }}"
    vcenter_validate_certs: "{{ vcenter_validate_certs }}"
```

### Execution Environment Dependencies

This playbook is designed to run within an execution environment that provides the required collections:

- Collections are defined in `execution_environment/requirements.yml`
- The playbook does not explicitly declare collections, relying on the EE
- This approach ensures consistency and reduces playbook boilerplate

If running outside an execution environment, ensure these collections are installed:
```bash
ansible-galaxy collection install vmware.vmware_rest ansible.posix
```

### Report Configuration

Configure report output in `group_vars/all.yml`:

```yaml
cdrom_report_config:
  output_format: "table"              # Options: json, yaml, table
  include_device_details: true
  timestamp_reports: true
```

### CD-ROM Removal Configuration

The playbook focuses on CD-ROM removal operations with check mode support:

```yaml
cdrom_removal_config:
  confirm_removal: true               # Require user confirmation for live operations
  show_details: true                  # Display ISO details before removal
  generate_report: true               # Create operation reports
  output_format: "table"              # Report format
  skip_powered_on_vms: false          # Optional: skip running VMs
```

#### Safety Features:
- **Check Mode**: Native Ansible `--check` flag for safe dry-run operations
- **User Confirmation**: Interactive prompt before performing live removal
- **Automatic Mode Detection**: Behaves differently in check vs run mode
- **Detailed Reporting**: Comprehensive reports for both dry-run and live operations
- **Error Handling**: Graceful handling of removal failures
- **Power State Awareness**: Optional filtering based on VM power state

## Usage

### With Ansible Navigator (Outside AAP)

1. Build the execution environment:
```bash
ansible-builder build --tag localhost/vmware-automation-ee:latest
```

2. Run dry-run preview (safe mode):
```bash
ansible-navigator run manage_cd_drives.yml --check
```

3. Run live CD-ROM removal operations:
```bash
ansible-navigator run manage_cd_drives.yml
```

4. Run with specific options:
```bash
# Dry-run preview with detailed output
ansible-navigator run manage_cd_drives.yml --check -v

# Live removal with extra variables
ansible-navigator run manage_cd_drives.yml -e "confirm_removal=false"
```

### Within AAP 2.5

1. Import this project into AAP
2. Configure VMware credentials in AAP
3. Set up dynamic inventory for VMware
4. Create job templates:
   - **Dry-Run Template**: Use `manage_cd_drives.yml` with "Check Mode" enabled for safe preview
   - **Removal Template**: Use `manage_cd_drives.yml` with "Check Mode" disabled for live operations
5. Execute the appropriate job template based on your needs

## AAP Integration Notes

### Credentials
- Create a **VMware vCenter** credential type in AAP
- Map credential fields to the expected variable names:
  - `vcenter_server` → vCenter hostname
  - `vcenter_user` → Username
  - `vcenter_pass` → Password

### Execution Environment
- Use the provided `execution_environment/` configuration
- Build EE with VMware dependencies included
- Configure job templates to use the custom EE

### Dynamic Inventory
- Configure VMware dynamic inventory in AAP
- Ensure inventory provides necessary VM metadata
- Target `localhost` for vSphere API operations

## Report Output

The playbook generates reports containing:

- **Summary Statistics**: Total VMs, VMs with CD-ROMs, VMs with mounted ISOs
- **VM Details**: Name, UUID, power state, datacenter
- **CD-ROM Information**: Device count, mount status, device labels
- **Timestamps**: Report generation time for auditing

Example output:
```
=== VMware CD-ROM Drive Report ===
Total VMs scanned: 150
VMs with CD-ROM devices: 12
VMs with mounted ISOs: 3
```

## Sample Reports

The `reports/` directory contains sample reports that demonstrate the output formats and content generated by the CD-ROM removal playbook:

### CD-ROM Removal Report - Check Mode (`sample_cdrom_removal_check_mode.txt`)

This report shows what the playbook generates when run with `--check` flag:
- **Purpose**: Preview what ISOs would be removed without making actual changes
- **Content**: Detailed list of VMs with mounted ISOs, including device information and ISO file paths
- **Use Case**: Pre-removal planning and stakeholder review
- **Safety Feature**: Check mode ensures no changes are made during assessment
- **Key Information**:
  - Total VMs scanned: 87
  - VMs with mounted ISOs: 12
  - ISO file locations and device details
  - VM power states for change planning

### CD-ROM Removal Report - Run Mode (`sample_cdrom_removal_run_mode.txt`)

This report shows what the playbook generates during actual removal operations:
- **Purpose**: Documents completed ISO removal operations with success/failure status
- **Content**: Operation timestamps, success rates, error messages, and post-removal guidance
- **Use Case**: Audit trail and troubleshooting for completed operations
- **Key Metrics**:
  - Total VMs scanned: 87
  - VMs with mounted ISOs: 8
  - Successful removals: 7
  - Failed removals: 1
  - Detailed error reporting for failed operations

### Report Template Usage

These samples are generated from the `cdrom_removal_report.j2` template in the `templates/` directory. The template automatically adjusts its content based on whether the playbook runs in check mode or run mode, providing:

- **Check Mode**: Dry-run preview with "Next Steps" guidance
- **Run Mode**: Live operation results with success/failure details and post-removal notes

The template supports dynamic content based on operation mode and provides comprehensive documentation for compliance and operational needs.

## Troubleshooting

### Collection Issues
- Ensure certified collections are available in AAP
- Check `ansible.cfg` server configuration
- Verify token authentication for Red Hat Automation Hub

### Connection Issues
- Verify vCenter credentials in AAP
- Check network connectivity from execution environment
- Validate SSL certificate settings (`vcenter_validate_certs`)

### Performance
- Adjust `max_concurrent_tasks` in group_vars for large environments
- Increase `api_timeout` for slow vCenter responses
- Use VM filtering options to reduce scope

## Security Considerations

- All credentials managed through AAP credential system
- SSL certificate validation configurable
- Execution environment isolation
- Principle of least privilege for vCenter access

## Dependencies

### Collections
- `vmware.vmware_rest` (certified) - Primary collection for vSphere REST API
- `ansible.posix` - For utility functions and JSON processing

### Python Packages
- `pyvmomi>=8.0.2`
- `requests>=2.31.0`
- `PyYAML>=6.0.1`

### System Packages
- SSL development libraries
- XML processing libraries
- CA certificates