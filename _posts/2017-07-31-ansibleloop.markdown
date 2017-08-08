---
layout: post
title:  "Getting content and file names from a fileglob with Ansible"
date:   2017-07-31 08:00:56 -0700
categories: ansible
---

I recently had a use case for [Ansible][ansible] loops that seemed obvious, but wasn't particularly well documented or straightforward. Say you had a bunch of files that you needed to loop over, read the contents, and spit that into another template along with each file's name. In my case, this was necessary for building a [Cloudformation][cf] template from another git repository that had YAML files in it defining various AWS roles and policies.

The Cloudformation resource name needed to be an edited version of the file name, and the contents were the contents of that particular file. I needed to take about a dozen of these policy files and create a single Cloudformation template we could use to make various roles and policies.

Say I had a couple policies in a ansible subdirectory like so:
```
~/code/ansible > ls -l aws_iam_policies/policy/*
-rw-r--r--  1 username  somegroup  114 Aug  7 09:01 aws_iam_policies/policy/cfadmin.yml
-rw-r--r--  1 username  somegroup  209 Aug  7 09:06 aws_iam_policies/policy/user.yml
```
The policies themselves are just CF YAML files with one IAM policy per file:
```
~/code/ansible > cat aws_iam_policies/policy/cfadmin.yml
Policy:
  Statement:
  - Action: "cloudformation:*"
    Effect: "Allow"
    Resource: '*'
  Version: "2012-10-17"

~/code/ansible > cat aws_iam_policies/policy/user.yml
Policy:
  Statement:
  - Action:
    - "iam:GetAccountPasswordPolicy"
    - "iam:GetAccountSummary"
    - "iam:ListAccount*"
    - "iam:ListUsers"
    Effect: "Allow"
    Resource: '*'
  Version: "2012-10-17"
```
My goal was to create a CF template that ended up looking like so:
```
Resources:
  CfadminPolicy:
    Type: "AWS::IAM::ManagedPolicy"
    Properties:
      ManagedPolicyName: "cfadmin"
      PolicyDocument:
        Statement:
        - Action: "cloudformation:*"
          Effect: "Allow"
          Resource: '*'
        Version: "2012-10-17"
  UserPolicy:
    Type: "AWS::IAM::ManagedPolicy"
    Properties:
      ManagedPolicyName: "user"
      PolicyDocument:
        Statement:
        - Action:
          - "iam:GetAccountPasswordPolicy"
          - "iam:GetAccountSummary"
          - "iam:ListAccount*"
          - "iam:ListUsers"
          Effect: "Allow"
          Resource: '*'
        Version: "2012-10-17"
```
The resource name for each policy is the file name (without the extension and camel cased) with a "Policy" suffix, while the contents of the "PolicyDocument" are the contents of the yaml files.

To accomplish this, I needed to use a [with_fileglob][ansible_fileglob] loop to get every file in the directory, combined with a [file lookup][ansible_lookup] to read the contents. I used [set_fact][ansible_setfact] with the lookup to create a "policy" variable I could use for the contents, then [registered][ansible_register] this all as a data structure I could iterate over in my template. The playbook looks like this:
```
~/code/ansible > cat policies.yml
---
- hosts: localhost
  connection: local
  gather_facts: False
  tasks:
  - name: Get data for IAM policies
    set_fact:
      policy: {% raw %}"{{ lookup('file', item) }}"{% endraw %}
    with_fileglob:
      - {% raw %}"{{ playbook_dir }}/aws_iam_policies/policy/*"{% endraw %}
    register: output

  - name: Render Cloudformation template
    template:
      src: "iam_policies.j2"
      dest: "{% raw %}{{ playbook_dir }}/cloudformation/iam_policies.yml"{% endraw %}
```
If you were to print out the "output" we registered above via a debug task, you would see this data structure:
```
ok: [localhost] => {
    "msg": {
        "changed": false,
        "msg": "All items completed",
        "results": [
            {
                "_ansible_item_result": true,
                "_ansible_no_log": false,
                "ansible_facts": {
                    "policy": "Policy:\n  Statement:\n  - Action: \"cloudformation:*\"\n    Effect: \"Allow\"\n    Resource: '*'\n  Version: \"2012-10-17\""
                },
                "changed": false,
                "item": "/Users/rorski/code/ansible/aws_iam_policies/policy/cfadmin.yml"
            },
            {
                "_ansible_item_result": true,
                "_ansible_no_log": false,
                "ansible_facts": {
                    "policy": "Policy:\n  Statement:\n  - Action:\n    - \"iam:GetAccountPasswordPolicy\"\n    - \"iam:GetAccountSummary\"\n    - \"iam:ListAccount*\"\n    - \"iam:ListUsers\"\n    Effect: \"Allow\"\n    Resource: '*'\n  Version: \"2012-10-17\""
                },
                "changed": false,
                "item": "/Users/rorski/code/ansible/aws_iam_policies/policy/user.yml"
            }
        ]
    }
}
```
Once I had this structure in place it was relatively straightforward. I just looped through the "results" list, grabbed the "item" key (the file name) for the policy and resource name and filtered it accordingly. I then grabbed ansible_facts.policy for the actual contents.

The template to produce the above from these files looks like this:
```
---
AWSTemplateFormatVersion: '2010-09-09'
Description: "IAM policies from ansible"
Resources:
{% raw %}{% for data in output.results %}
  {{ data.item | basename | splitext | first | camelize }}Policy:
    Type: "AWS::IAM::ManagedPolicy"
    Properties:
      ManagedPolicyName: "{{ data.item | basename | splitext | first }}"
      PolicyDocument:
        {{ data.ansible_facts.policy | regex_replace('---|Policy:', '') | trim | indent(6, false) }}
{% endfor %}{% endraw %}
```
You can check out the [filters][ansible_filters] page for some more detail on the filters used above - and see my next post to read about the "camelize" filter I created to simplify switching strings with underscores and dashes to camelcase (Cloudformation resources can't use dashes or underscores). The "indent" filter here was especially important since we're creating a YAML file that needs to be correctly indented.

This ended up creating a syntactically correct CF template that we could then load into AWS to make these policies.

[ansible]: https://www.ansible.com/
[ansible_lookup]: http://docs.ansible.com/ansible/latest/playbooks_lookups.html#intro-to-lookups-getting-file-contents
[ansible_filters]: http://docs.ansible.com/ansible/latest/playbooks_filters.html
[ansible_fileglob]: http://docs.ansible.com/ansible/latest/playbooks_loops.html#id4
[ansible_register]: http://docs.ansible.com/ansible/latest/playbooks_conditionals.html#register-variables
[ansible_setfact]: http://docs.ansible.com/ansible/latest/set_fact_module.html
[cf]: https://aws.amazon.com/cloudformation/
