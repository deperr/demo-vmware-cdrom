# VMware CD-ROM Drive Check Automation

This Ansible project provides automation for checking VMware vSphere virtual machines that have CD-ROM drives attached, designed for use with Ansible Automation Platform 2.5.

## Project Structure

```
test_demo-vmware-cdrom/
├── ansible.cfg                          # Ansible configuration with certified collections
├── ansible-navigator.yml                # Navigator configuration for EE execution
├── site.yml                            # Main playbook entry point
├── collections/
│   └── requirements.yml                 # Collection dependencies (certified + community)
├── execution_environment/
│   ├── execution-environment.yml        # EE build configuration
│   ├── requirements.yml                 # Collection requirements for EE
│   ├── requirements.txt                 # Python dependencies (pyvmomi, etc.)
│   └── bindep.txt                      # System package dependencies
├── group_vars/
│   └── all.yml                         # VMware connection and configuration variables
├── playbooks/
│   ├── check_mounted_cd_drives.yml      # Main CD-ROM checking playbook
│   └── remove_mounted_cd_drives.yml     # CD-ROM removal playbook with safety controls
├── templates/
│   ├── cdrom_removal_report.j2         # Template for CD-ROM removal operations
│   └── cdrom_report_table.j2           # Report template for table format
└── reports/                            # Generated reports directory
```

## Features

- **AAP 2.5 Compatibility**: Designed for Ansible Automation Platform 2.5
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

The removal playbook includes comprehensive safety controls:

```yaml
cdrom_removal_config:
  dry_run: true                       # Safe mode - no actual changes
  confirm_removal: true               # Require user confirmation
  show_details: true                  # Display ISO details before removal
  generate_report: true               # Create removal reports
  output_format: "table"              # Report format
  skip_powered_on_vms: false          # Optional: skip running VMs
```

#### Safety Features:
- **Dry Run Mode**: Default mode shows what would be removed without making changes
- **User Confirmation**: Interactive prompt before performing live removal
- **Detailed Reporting**: Comprehensive before/after reports
- **Error Handling**: Graceful handling of removal failures
- **Power State Awareness**: Optional filtering based on VM power state

## Usage

### With Ansible Navigator (Outside AAP)

1. Build the execution environment:
```bash
ansible-builder build --tag localhost/vmware-automation-ee:latest
```

2. Run the check playbook (default):
```bash
ansible-navigator run site.yml
```

3. Run the removal playbook:
```bash
ansible-navigator run site.yml -e operation=remove
```

4. Run specific playbooks directly:
```bash
# Check for mounted CD-ROMs
ansible-navigator run playbooks/check_mounted_cd_drives.yml

# Remove mounted CD-ROMs (with safety controls)
ansible-navigator run playbooks/remove_mounted_cd_drives.yml
```

### Within AAP 2.5

1. Import this project into AAP
2. Configure VMware credentials in AAP
3. Set up dynamic inventory for VMware
4. Create job template pointing to `site.yml`
5. Execute the job template

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

The `reports/` directory contains three sample reports that demonstrate the output formats and content generated by the playbooks:

### CD-ROM Inventory Report (`sample_cdrom_table_report.txt`)

This report shows the comprehensive inventory scanning capabilities:
- **Purpose**: Provides detailed inventory of all VMs with CD-ROM devices
- **Content**: VM names, IDs, power states, CD-ROM device counts, and mounted ISO status
- **Use Case**: Initial assessment and documentation of CD-ROM device usage across the environment
- **Key Metrics**: 
  - Total VMs scanned: 87
  - VMs with CD-ROM devices: 65
  - VMs with mounted ISOs: 12

### CD-ROM Removal Report - Dry Run (`sample_cdrom_removal_report_dry_run.txt`)

This report demonstrates the safe planning mode for ISO removal:
- **Purpose**: Shows what ISOs would be removed without making actual changes
- **Content**: Detailed list of VMs with mounted ISOs, including device information and ISO file paths
- **Use Case**: Pre-removal planning and stakeholder review
- **Safety Feature**: Dry run mode ensures no changes are made during assessment
- **Key Information**:
  - ISO file locations and names
  - VM power states for change planning
  - Device-specific details for each CD-ROM drive

### CD-ROM Removal Report - Live (`sample_cdrom_removal_report_live.txt`)

This report shows actual removal operation results:
- **Purpose**: Documents completed ISO removal operations with success/failure status
- **Content**: Operation timestamps, success rates, error messages, and post-removal guidance
- **Use Case**: Audit trail and troubleshooting for completed operations
- **Key Metrics**:
  - Successful removals: 7
  - Failed removals: 1
  - Detailed error reporting for failed operations

### Report Template Usage

These samples are generated from the Jinja2 templates in the `templates/` directory:
- `cdrom_report_table.j2`: Generates inventory-style reports
- `cdrom_removal_report.j2`: Generates removal operation reports (supports both dry run and live modes)

The templates support dynamic content based on operation mode and provide comprehensive documentation for compliance and operational needs.

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