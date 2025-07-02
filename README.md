# Shell-Scripting-Error-Handling

## Learning Summary

### porting through structured logging with timestamps and severity levels, and resource cleanup using signal handlers to prevent orphaned resources when scripts are interrupted. The project demonstrated that effective error handling is not just about catching failures, but about anticipating potential issues, providing meaningful feedback to users, maintaining system reliability, and ensuring scripts can be safely run multiple times without unintended side effects - ultimately transforming fragile scripts into production-ready automation tools.

### Create an AWS error handling script.

* Run `nano error_handling.sh`

![](./img/Pasted%20image.png)

```bash
#!/bin/bash

# Error Handling Shell Script for AWS Resource Management
# This script demonstrates comprehensive error handling techniques

# Enable strict error handling
set -euo pipefail

# Global variables
COMPANY="datawise"
DEPARTMENTS=("Marketing" "Sales" "HR" "Operations" "Media")
LOG_FILE="deployment_$(date +%Y%m%d_%H%M%S).log"
REGION="us-east-1"

# Function to log messages with timestamps
log_message() {
    local level="$1"
    local message="$2"
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [$level] $message" | tee -a "$LOG_FILE"
}

# Function to check AWS CLI installation and configuration
check_aws_prerequisites() {
    log_message "INFO" "Checking AWS prerequisites..."
    
    # Check if AWS CLI is installed
    if ! command -v aws &> /dev/null; then
        log_message "ERROR" "AWS CLI is not installed. Please install AWS CLI first."
        exit 1
    fi
    
    # Check if AWS credentials are configured
    if ! aws sts get-caller-identity &> /dev/null; then
        log_message "ERROR" "AWS credentials not configured. Run 'aws configure' first."
        exit 1
    fi
    
    log_message "INFO" "AWS prerequisites check passed."
}

# Function to validate user input
validate_input() {
    local input="$1"
    local input_type="$2"
    
    case "$input_type" in
        "region")
            if [[ ! "$input" =~ ^[a-z]{2}-[a-z]+-[0-9]$ ]]; then
                log_message "ERROR" "Invalid region format: $input"
                return 1
            fi
            ;;
        "instance_type")
            local valid_types=("t2.micro" "t2.small" "t2.medium" "t3.micro" "t3.small")
            if [[ ! " ${valid_types[*]} " =~ " $input " ]]; then
                log_message "ERROR" "Invalid instance type: $input"
                return 1
            fi
            ;;
    esac
    return 0
}

# Function to create S3 buckets with comprehensive error handling
create_s3_buckets() {
    log_message "INFO" "Starting S3 bucket creation process..."
    local success_count=0
    local error_count=0
    
    for department in "${DEPARTMENTS[@]}"; do
        local bucket_name="${COMPANY}-${department,,}-data-bucket-$(date +%s)"
        
        log_message "INFO" "Processing bucket: $bucket_name"
        
        # Check if bucket already exists
        if aws s3api head-bucket --bucket "$bucket_name" --region "$REGION" &>/dev/null; then
            log_message "WARN" "S3 bucket '$bucket_name' already exists. Skipping creation."
            continue
        fi
        
        # Attempt to create bucket with error handling
        local create_output
        if create_output=$(aws s3api create-bucket --bucket "$bucket_name" --region "$REGION" 2>&1); then
            log_message "SUCCESS" "S3 bucket '$bucket_name' created successfully."
            
            # Apply bucket versioning with error handling
            if aws s3api put-bucket-versioning --bucket "$bucket_name" --versioning-configuration Status=Enabled &>/dev/null; then
                log_message "INFO" "Versioning enabled for bucket '$bucket_name'."
            else
                log_message "WARN" "Failed to enable versioning for bucket '$bucket_name'."
            fi
            
            ((success_count++))
        else
            log_message "ERROR" "Failed to create S3 bucket '$bucket_name'. Error: $create_output"
            ((error_count++))
        fi
    done
    
    log_message "INFO" "S3 bucket creation summary: $success_count successful, $error_count failed."
    return $error_count
}

# Function to create EC2 instances with error handling
create_ec2_instances() {
    log_message "INFO" "Starting EC2 instance creation process..."
    local instance_type="${1:-t2.micro}"
    local key_name="${2:-}"
    
    # Validate instance type
    if ! validate_input "$instance_type" "instance_type"; then
        return 1
    fi
    
    # Check if key pair exists if provided
    if [[ -n "$key_name" ]]; then
        if ! aws ec2 describe-key-pairs --key-names "$key_name" --region "$REGION" &>/dev/null; then
            log_message "ERROR" "Key pair '$key_name' does not exist in region '$REGION'."
            return 1
        fi
    fi
    
    # Get latest Amazon Linux AMI ID
    local ami_id
    if ! ami_id=$(aws ec2 describe-images --owners amazon --filters "Name=name,Values=amzn2-ami-hvm-*" "Name=state,Values=available" --query 'Images | sort_by(@, &CreationDate) | [-1].ImageId' --output text --region "$REGION" 2>/dev/null); then
        log_message "ERROR" "Failed to retrieve AMI ID for region '$REGION'."
        return 1
    fi
    
    local success_count=0
    local error_count=0
    
    for department in "${DEPARTMENTS[@]}"; do
        local instance_name="${COMPANY}-${department}-server"
        
        log_message "INFO" "Creating EC2 instance: $instance_name"
        
        # Check if instance with same name already exists
        local existing_instance
        existing_instance=$(aws ec2 describe-instances --filters "Name=tag:Name,Values=$instance_name" "Name=instance-state-name,Values=running,pending,stopping,stopped" --query 'Reservations[*].Instances[*].InstanceId' --output text --region "$REGION" 2>/dev/null)
        
        if [[ -n "$existing_instance" ]]; then
            log_message "WARN" "EC2 instance '$instance_name' already exists (ID: $existing_instance). Skipping creation."
            continue
        fi
        
        # Create EC2 instance
        local run_params="--image-id $ami_id --count 1 --instance-type $instance_type --region $REGION"
        [[ -n "$key_name" ]] && run_params+=" --key-name $key_name"
        
        local instance_id
        if instance_id=$(aws ec2 run-instances $run_params --query 'Instances[0].InstanceId' --output text 2>/dev/null); then
            log_message "SUCCESS" "EC2 instance created with ID: $instance_id"
            
            # Add name tag with error handling
            if aws ec2 create-tags --resources "$instance_id" --tags Key=Name,Value="$instance_name" --region "$REGION" &>/dev/null; then
                log_message "INFO" "Name tag added to instance $instance_id."
            else
                log_message "WARN" "Failed to add name tag to instance $instance_id."
            fi
            
            ((success_count++))
        else
            log_message "ERROR" "Failed to create EC2 instance for department: $department"
            ((error_count++))
        fi
    done
    
    log_message "INFO" "EC2 instance creation summary: $success_count successful, $error_count failed."
    return $error_count
}

# Function to cleanup resources on script interruption
cleanup() {
    log_message "WARN" "Script interrupted. Performing cleanup..."
    # Add any cleanup logic here
    exit 1
}

# Function to display usage information
usage() {
    echo "Usage: $0 [OPTIONS]"
    echo "Options:"
    echo "  -r, --region REGION        AWS region (default: us-east-1)"
    echo "  -t, --instance-type TYPE   EC2 instance type (default: t2.micro)"
    echo "  -k, --key-name KEY         EC2 key pair name (optional)"
    echo "  -h, --help                 Display this help message"
    echo ""
    echo "Example: $0 --region us-west-2 --instance-type t2.small --key-name my-key"
}

# Main function with comprehensive error handling
main() {
    # Set up signal handlers for cleanup
    trap cleanup SIGINT SIGTERM
    
    # Parse command line arguments
    while [[ $# -gt 0 ]]; do
        case $1 in
            -r|--region)
                REGION="$2"
                if ! validate_input "$REGION" "region"; then
                    exit 1
                fi
                shift 2
                ;;
            -t|--instance-type)
                INSTANCE_TYPE="$2"
                shift 2
                ;;
            -k|--key-name)
                KEY_NAME="$2"
                shift 2
                ;;
            -h|--help)
                usage
                exit 0
                ;;
            *)
                log_message "ERROR" "Unknown option: $1"
                usage
                exit 1
                ;;
        esac
    done
    
    log_message "INFO" "Starting AWS resource deployment script..."
    log_message "INFO" "Region: $REGION, Instance Type: ${INSTANCE_TYPE:-t2.micro}"
    
    # Check prerequisites
    check_aws_prerequisites
    
    local overall_exit_code=0
    
    # Create S3 buckets
    if ! create_s3_buckets; then
        log_message "ERROR" "S3 bucket creation encountered errors."
        overall_exit_code=1
    fi
    
    # Create EC2 instances
    if ! create_ec2_instances "${INSTANCE_TYPE:-t2.micro}" "${KEY_NAME:-}"; then
        log_message "ERROR" "EC2 instance creation encountered errors."
        overall_exit_code=1
    fi
    
    if [[ $overall_exit_code -eq 0 ]]; then
        log_message "SUCCESS" "All resources created successfully!"
        log_message "INFO" "Log file saved to: $LOG_FILE"
    else
        log_message "ERROR" "Script completed with errors. Check log file: $LOG_FILE"
    fi
    
    exit $overall_exit_code
}

# Execute main function with all arguments
main "$@"
```