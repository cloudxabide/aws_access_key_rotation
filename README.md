# AWS Access Key Rotation Tool (keyctl)

A Python tool to safely rotate AWS IAM user access keys and automatically update your `~/.aws/credentials` file.

## Overview

`keyctl` automates the process of rotating AWS access keys, which is a critical security best practice. The tool:

1. Allows you to choose which profile to use for authentication
2. Allows you to choose which profile's keys to rotate
3. Creates a new AWS access key for the target IAM user
4. Backs up old credentials to a `{profile}-previous` profile
5. Updates the target profile with the new credentials
6. Creates a timestamped backup of your credentials file

## Features

- **Two-profile system**: Separate authentication and target profiles for flexible key management
- **Interactive mode**: Select profiles from a menu
- **Command-line mode**: Specify profiles via `-a` (auth) and `-t` (target) flags
- **Admin support**: Use an admin profile to rotate keys for other users
- **Self-service**: Rotate your own keys (use same profile for both auth and target)
- **Safe rotation**: Backs up old credentials before updating
- **Automatic cleanup**: Handles the AWS 2-key limit by removing the oldest key if needed
- **Preserves settings**: Maintains region and other profile settings during rotation
- **Timestamped backups**: Creates a backup of your credentials file before any changes

## Requirements

- Python 3.6+
- boto3 library
- AWS CLI credentials configured in `~/.aws/credentials`
- IAM permissions for the user to manage their own keys:
  - `iam:CreateAccessKey`
  - `iam:DeleteAccessKey`
  - `iam:ListAccessKeys`
  - `iam:GetUser`

## Installation

1. Clone or download this repository:
```bash
cd ~/Developer/Projects/aws_access_key_rotation
```

2. Install required Python dependencies:
```bash
pip install boto3
```

3. Make the script executable (already done):
```bash
chmod +x keyctl
```

4. (Optional) Add to your PATH for easy access:
```bash
# Add to ~/.bashrc or ~/.zshrc
export PATH="$PATH:$HOME/Developer/Projects/aws_access_key_rotation"
```

## Common Use Cases

### Self-Service Rotation
Rotate your own keys using your own profile:
```bash
# Interactive: Select the same profile for both steps
./keyctl

# Command line: Specify only target (uses it for auth too)
./keyctl -t my-profile
```

### Admin-Managed Rotation
Use an admin profile to rotate keys for other users:
```bash
# Interactive: Select admin for auth, user profile for target
./keyctl

# Command line: Specify different profiles
./keyctl -a admin -t developer-user
```

### Automated Rotation
Rotate multiple users in a script:
```bash
#!/bin/bash
for user in dev1 dev2 dev3; do
    ./keyctl -a admin -t $user
done
```

## Usage

### Interactive Menu Mode

Run without arguments to see a two-step menu:

```bash
./keyctl
```

Example:
```
Select the profile to use for AWS API authentication:
(This profile will be used to call AWS IAM APIs)

Available profiles for authentication:
  1. admin
  2. default
  3. production

Select profile number (or 'q' to quit): 1

============================================================
Select the profile whose keys you want to rotate:
(This profile will be updated with new credentials)

Available profiles to rotate:
  1. admin
  2. default
  3. production

Select profile number (or 'q' to quit): 3
```

This two-step approach allows you to:
- Use an admin profile to rotate keys for other users
- Rotate your own keys (select same profile for both)
- Safely manage multiple user credentials

### Command Line Mode

Specify both profiles directly:

```bash
# Use admin profile to rotate production profile
./keyctl -a admin -t production

# Rotate a profile using itself for authentication
./keyctl -t production

# Or specify both the same
./keyctl -a production -t production
```

### List Profiles

List all available profiles without rotating:

```bash
./keyctl --list
```

### Custom Credentials File

Specify a different credentials file location:

```bash
./keyctl --credentials /path/to/credentials -a admin -t production
```

## How It Works

1. **Select Profiles**: Choose authentication profile and target profile (or specify via command line)
2. **Read Current Configuration**: Parses `~/.aws/credentials` to find the target profile
3. **Authenticate**: Uses the auth profile credentials to authenticate with AWS IAM
4. **Identify User**: Determines which IAM user owns the target profile's access key
5. **Check Key Limit**: AWS allows max 2 access keys per user. If the user has 2, the oldest is deleted
6. **Create New Key**: Generates a new access key for the target user via the IAM API
7. **Backup Old Credentials**: Moves current credentials to `{target-profile}-previous`
8. **Update Profile**: Writes new credentials to the target profile
9. **Create File Backup**: Saves a timestamped backup of the entire credentials file

## Example Workflow

**Before rotation** (`~/.aws/credentials`):
```ini
[production]
aws_access_key_id = AKIAIOSFODNN7EXAMPLE
aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
region = us-east-1
```

**After running `./keyctl production`**:
```ini
[production]
aws_access_key_id = AKIAI44QH8DHBEXAMPLE
aws_secret_access_key = je7MtGbClwBF/2Zp9Utk/h3yCo8nvbEXAMPLEKEY
region = us-east-1

[production-previous]
aws_access_key_id = AKIAIOSFODNN7EXAMPLE
aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
region = us-east-1
```

A backup file is also created: `~/.aws/credentials.backup.20250118_143022`

## IAM Policies

### Self-Service Key Rotation

To allow IAM users to rotate their own keys, attach this policy (available in `keyctl-self-service-policy.json`):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "iam:CreateAccessKey",
        "iam:DeleteAccessKey",
        "iam:ListAccessKeys",
        "iam:GetUser"
      ],
      "Resource": "arn:aws:iam::*:user/${aws:username}"
    }
  ]
}
```

This policy ensures users can only manage their own access keys, not those of other users.

**To apply via AWS CLI:**
```bash
aws iam create-policy \
  --policy-name KeyctlSelfServiceRotation \
  --policy-document file://keyctl-self-service-policy.json

aws iam attach-user-policy \
  --user-name YOUR_USERNAME \
  --policy-arn arn:aws:iam::YOUR_ACCOUNT_ID:policy/KeyctlSelfServiceRotation
```

### Admin Key Rotation (for rotating other users' keys)

If you want to use an admin profile to rotate keys for other users, the admin needs these permissions (available in `keyctl-admin-policy.json`):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "iam:CreateAccessKey",
        "iam:DeleteAccessKey",
        "iam:ListAccessKeys",
        "iam:GetUser",
        "iam:ListUsers"
      ],
      "Resource": "*"
    }
  ]
}
```

**Note**: This grants broad IAM permissions. Use carefully and consider restricting the `Resource` to specific users or paths.

**To apply via AWS CLI:**
```bash
aws iam create-policy \
  --policy-name KeyctlAdminRotation \
  --policy-document file://keyctl-admin-policy.json

aws iam attach-user-policy \
  --user-name ADMIN_USERNAME \
  --policy-arn arn:aws:iam::YOUR_ACCOUNT_ID:policy/KeyctlAdminRotation
```

## Security Best Practices

1. **Rotate regularly**: Set a reminder to rotate keys every 90 days
2. **Delete old keys**: After verifying the new keys work, you can safely delete the `-previous` profile
3. **Monitor key age**: Use AWS CloudWatch or AWS Config to monitor access key age
4. **Use MFA**: Enable MFA for your IAM user account
5. **Least privilege**: Only grant the minimum permissions needed

## Troubleshooting

### "Error: boto3 is required"
Install boto3: `pip install boto3`

### "Error: Profile not found"
Check that the profile exists in `~/.aws/credentials` using `./keyctl --list`

### "AWS API Error: AccessDenied"
This can happen for several reasons:
- **Self-rotation**: Ensure the auth profile has permissions to manage its own keys (see Self-Service IAM Policy)
- **Admin rotation**: Ensure the auth profile has permissions to manage other users' keys (see Admin IAM Policy)
- **Invalid credentials**: The auth profile credentials may be expired or invalid

### "Error: Could not determine which user owns the target access key"
When using different auth and target profiles, the script needs to identify which IAM user owns the target access key. This requires:
- The auth profile to have `iam:ListUsers` permission
- Access to list all users in the account
- Or you can specify the username manually with `--user` flag (future enhancement)

### "User has maximum number of access keys (2)"
The script will prompt you to delete the oldest key. You can also manually delete an old key in the AWS Console first.

### New credentials don't work
The old credentials are saved in the `{profile}-previous` profile. You can temporarily use those while troubleshooting:
```bash
aws s3 ls --profile production-previous
```

### Different profiles for same user?
Yes! You can have multiple profiles pointing to the same IAM user (e.g., `production` and `prod-backup`). When rotating one, it won't affect the other until you rotate that one too.

## Backup and Recovery

The tool creates two types of backups:

1. **Profile backup**: Old credentials saved as `{profile}-previous` in the same file
2. **File backup**: Complete credentials file saved as `~/.aws/credentials.backup.YYYYMMDD_HHMMSS`

To restore from a backup:
```bash
cp ~/.aws/credentials.backup.20250118_143022 ~/.aws/credentials
```

## Contributing

Feel free to submit issues or pull requests to improve this tool.

## License

MIT License - feel free to use and modify as needed.

## Changelog

### Version 2.0.0
- **Breaking change**: Two-profile system (authentication + target)
- Interactive mode now asks for both auth and target profiles
- Command-line flags changed to `-a/--auth-profile` and `-t/--target-profile`
- Support for admin-managed key rotation
- Automatic user detection when auth and target differ

### Version 1.0.0
- Initial release
- Interactive and direct profile selection
- Automatic key rotation with backup
- Support for 2-key limit handling
