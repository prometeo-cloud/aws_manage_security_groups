# aws_manage_security_groups

## Description:

Manages AWS Security Groups as follows:

- Running the role will create the Security Groups defined in `aws_sec_grps` when the variable `arg_action` is not defined or is defined and set to `create`
- Running the role will delete the Security Groups defined in `aws_sec_grps` when the variable `arg_action` is defined and is set to `delete`
- Running the role task  [get_security_group_facts.yml](./tasks/get_security_group_facts.yml) play will return information about the specified Security Group.

If no inbound or outbound rules are specified for a security group then when it is created it will default to no inbound or outbound rules.

It is sometimes necessary to create a security group with no rules first and then update its rules later, e.g. when two security groups, A and B,  have reciprocal rules that connect to each other.  This can be done as follows:

- create group A with no rules
- create group B with its rules to connect to group A
- update group A's rules to connect to group B (this is achieved by adding the `aws_sec_grp_update` variable and setting its value to `True` - see Usage) 

This role requires Boto3 to be installed as follows:

`sudo pip install boto3`

or:

```yaml
- name: Install boto3
  easy_install:
    name: boto3
    state: present
```

## Behaviour:

**Feature:** Create AWS security groups
- **Given** valid AWS credentials
- **When** the script is executed and `arg_action` not defined or is defined and not set to `delete`
- **Then** the security groups are created 

**Feature:** Delete AWS security groups
- **Given** valid AWS credentials
- **When** the script is executed with `arg_action` defined and set to `delete`
- **Then** the security groups are deleted 

## Configuration:

Common variables used by this role:

| Variable | Description | Example |
|---|---|---|
| **aws_region** | AWS region | See [AWS regions](http://docs.aws.amazon.com/general/latest/gr/rande.html#ec2_region) |
| **aws_access_key** | AWS access key | AKIAIOSFODNN7EXAMPLE |
| **aws_secret_key** | AWS secret key | wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY |
| **resource_tags** | List of project tags | see usage |

Variables specific to this role:

| Variable | Description | Example |
|---|---|---|
| **aws_sec_grps** | List of security groups to create | see usage |
| **sec_grp_tag** | The `Name:` tag of the security group to get information about | see usage |
| **arg_action** | Used to either create or delete security groups | not defined or `delete` |

## Usage:

How to invoke the role from a playbook (create Security Groups):

```yaml
- name: Create AWS Security Groups
  include_role:
    name: aws_manage_security_groups
  vars:
    aws_access_key: AKIAIOSFODNN7EXAMPLE
    aws_secret_key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
    aws_region: eu-west-2  # London
    resource_tags:
      Name: "{{ resource_name }}"   # Set to upper case aws_vpc_name
      Owner: "Acme Co."

    aws_sec_grps:
      # Create group with no rules
      - aws_sec_grp_name: "ctrl-grp-test-a2"
        aws_sec_grp_desc: "Test Security Group Control A2"
        aws_sec_grp_region: "{{ aws_region }}"
        aws_vpc_name: "vpc-test-ctrl"

      - aws_sec_grp_name: "ctrl-grp-test-a1"
        aws_sec_grp_desc: "Test Security Group Control A1"
        aws_sec_grp_region: "{{ aws_region }}"
        aws_vpc_name: "vpc-test-ctrl"
        aws_sec_grp_rules:
          - proto: tcp
            from_port: 80
            to_port: 80
            cidr_ip: 0.0.0.0/0
          - proto: tcp
            from_port: 22
            to_port: 22
            cidr_ip: 10.12.0.0/8
          - proto: tcp
            from_port: 443
            to_port: 443
            cidr_ip: 10.12.0.0/8
        aws_sec_grp_egress:
          - proto: tcp
            from_port: 80
            to_port: 80
            cidr_ip: 0.0.0.0/0

      # Update group rules
      - aws_sec_grp_name: "ctrl-grp-test-a2"
        aws_sec_grp_desc: "Test Security Group Control A2"
        aws_sec_grp_region: "{{ aws_region }}"
        aws_vpc_name: "vpc-test-ctrl"
        aws_sec_grp_update: True
        aws_sec_grp_rules:
          - proto: tcp
            from_port: 80
            to_port: 80
            cidr_ip: 0.0.0.0/0
          - proto: tcp
            from_port: 443
            to_port: 443
            group_name: "ctrl-grp-test-a1"
        aws_sec_grp_egress:
          - proto: all
            cidr_ip: 0.0.0.0/0

      - aws_sec_grp_name: "infra-grp-test-a1"
        aws_sec_grp_desc: "Test Security Group A1"
        aws_sec_grp_region: "{{ aws_region }}"
        aws_vpc_name: "vpc-test-iaas"
```

How to invoke the role from a playbook (delete Security Groups):

```yaml
- name: Delete AWS Security Groups
  include_role:
    name: aws_manage_security_groups
  vars:
    arg_action: "delete"
    aws_access_key: AKIAIOSFODNN7EXAMPLE
    aws_secret_key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
    aws_region: eu-west-2  # London
    resource_tags:
      Name: "{{ resource_name }}"   # Set to upper case aws_vpc_name
      Owner: "Acme Co."

    aws_sec_grps:
      # Create group with no rules
      - aws_sec_grp_name: "ctrl-grp-test-a2"
        aws_sec_grp_desc: "Test Security Group Control A2"
        aws_sec_grp_region: "{{ aws_region }}"
        aws_vpc_name: "vpc-test-ctrl"

      - aws_sec_grp_name: "ctrl-grp-test-a1"
        aws_sec_grp_desc: "Test Security Group Control A1"
        aws_sec_grp_region: "{{ aws_region }}"
        aws_vpc_name: "vpc-test-ctrl"
```


How to get information about a security group:

```yaml
tasks:
- name: Get facts
  include_role:
    name: "aws_manage_security_groups"
    tasks_from: "get_security_group_facts"
  vars:
    sec_grp_tag: "TEST-SEC-GROUP"
```

This will return facts in the variable `sec_grp_facts`.  The data returned is as follows:

```yaml
"sec_grp_facts": {
    "changed": false, 
    "failed": false, 
    "security_groups": [
        {
            "description": "A test Security Group", 
            "group_id": "sg-f1637477", 
            "group_name": "TEST-SEC-GROUP", 
            "ip_permissions": [
                {
                    "from_port": 80, 
                    "ip_protocol": "tcp", 
                    "ip_ranges": [
                        {
                            "cidr_ip": "0.0.0.0/0"
                        }
                    ], 
                    "ipv6_ranges": [], 
                    "prefix_list_ids": [], 
                    "to_port": 80, 
                    "user_id_group_pairs": []
                }, 
                {
                    "from_port": 443, 
                    "ip_protocol": "tcp", 
                    "ip_ranges": [
                        {
                            "cidr_ip": "0.0.0.0/0"
                        }
                    ], 
                    "ipv6_ranges": [], 
                    "prefix_list_ids": [], 
                    "to_port": 443, 
                    "user_id_group_pairs": []
                }
            ], 
            "ip_permissions_egress": [
                {
                    "from_port": 80, 
                    "ip_protocol": "tcp", 
                    "ip_ranges": [], 
                    "ipv6_ranges": [], 
                    "prefix_list_ids": [], 
                    "to_port": 80, 
                    "user_id_group_pairs": [
                        {
                            "group_id": "sg-af368534", 
                            "user_id": "121212121212"
                        }
                    ]
                }, 
                {
                    "from_port": 443, 
                    "ip_protocol": "tcp", 
                    "ip_ranges": [], 
                    "ipv6_ranges": [], 
                    "prefix_list_ids": [], 
                    "to_port": 443, 
                    "user_id_group_pairs": [
                        {
                            "group_id": "sg-af375335", 
                            "user_id": "121212121212"
                        }
                    ]
                }
            ], 
            "owner_id": "121212121212", 
            "tags": {
                "Name": "TEST-SEC-GROUP"
            }, 
            "vpc_id": "vpc-b37aa7df"
        }
    ]
}
```