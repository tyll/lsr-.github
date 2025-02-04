# .github

The configuration source for linux-system-roles repositories.  This uses Ansible
to manage configuration, github actions, and other common files used by
repositories in the linux-system-roles organization.  This allows org admins to
easily rollout updates to all repos.

# File structure

The structure of the files/directories under `playbooks/files` and
`playbooks/templates` should match exactly the name and location of the
files/directories in the role repositories.  For example,
`playbooks/templates/.ansible-lint` corresponds to the `.ansible-lint` file in
the root directory of the role repositories.
`playbooks/.github/workflows/weekly_ci.yml` corresponds to the file
`.github/workflows/weekly_ci.yml` in the role repositories.

The file `inventory.yml` is the list of all roles and contains the groups
`active_roles` for all of the actively maintained and supported roles, and the
group `python_roles` for the roles that provide Ansible python plugins such as
modules, filters, etc.

The file `inventory/group_vars/active_roles.yml` is used for settings common to
all roles.

The file `inventory/group_vars/python_roles.yml` is used for settings common to
all roles that have python modules, filters, and other Ansible plugin python
code.

The file `inventory/host_vars/$ROLENAME.yml` is used for settings that are
specific to that role.  Some examples:
* The scheduled time for a github action
* .ansible-lint or .yamllint.yml customizations

# Add a new role

* Edit inventory.yml
* Add the role in alphabetical order to the `all.hosts` section:
```yaml
all:
  hosts:
    ...
    postgresql:
      ansible_host: localhost
    quite_a_good_new_role:
      ansible_host: localhost
    rhc:
      ansible_host: localhost
```
* Add the role to the `active_roles.hosts` section:
```yaml
        postgresql:
        quite_a_good_new_role:
        rhc:
```
* If the role has python modules or filters or other plugins,
  add to the `python_roles.hosts` section:
```yaml
        network:
        quite_a_good_new_role:
        selinux:
```
* Add the new file `inventory/host_vars/$ROLENAME.yml` - add any customizations
  for the github actions weekly_ci, ansible_lint, etc.

# Add a new config or github action file

* Add the file under `playbooks/files` or `playbooks/templates`

Add the file according to the location in the role repository under
`playbooks/files` or `playbooks/templates`.  If the file is static, and needs no
per-role configuration (such as a github action cron schedule), then add under
`playbooks/files`.

NOTE: github action files will almost always be templates, due to the checkout
action being template-ized.

* Add the file to the appropriate list in
  `inventory/group_vars/active_roles.yml` or
  `inventory/group_vars/python_roles.yml`

`present_templates` are files that should be present in all roles that are
generated by templates.
`present_files` are files that should be present in all roles that are static.
`absent_files` are files that should be removed from all roles.
`present_python_templates` are files that should be present in roles that
provide Ansible python code that are generated by templates.
`present_python_files` are files that should be present in roles that provide
Ansible python code that are static.
`absent_python_files` are files that should be removed from roles that provide
Ansible python code.

# Creating PRs in every role with updated files

The playbook `playbooks/update_files.yml` will create PRs in all roles with the
new/updated/deleted files.  This uses the [gh](https://cli.github.com/) command
line tool provided by the `gh` package on Fedora.  You will need to configure
`gh` to authenticate to github using `~/.config/gh/hosts.yml`:
```yaml
github.com:
  user: my_user_name
  oauth_token: my_oauth_token
  git_protocol: ssh
```

If you just want to see what the playbook will do without actually creating
anything on github, add `-e lsr_dry_run=true` to the ansible-playbook command.

## Parameters

* `update_files_commit_file` - REQUIRED, no default - This is the path to the
  file containing the git commit message to use for the commit, and will also be
  used as the PR title and body.  Please use good practices for creating the
  commit message as described in
  [Contributing](https://linux-system-roles.github.io/contribute.html) under
  "Write a good commit message".
* `update_files_branch` - default "update_role_files" - this is the name of the
  git branch that will be used for the PR.  You probably don't want to change
  this unless you have some conflict.

## Run it

Run it like this:
```
ansible-playbook -vv -i inventory -e update_files_commit_file=/path/to/git-commit-msg playbooks/update_files.yml
```

## How it works

* A temp directory is created
* All of the roles are cloned into that directory
* Figure out the name of the main branch
* If the branch `update_files_branch` does not exist, it is
  created from the main branch
* If the branch `update_files_branch` already exists, it will
  be rebased on top of the main branch
* Add/update/remove the files to be managed in each role
* If there are no updates to be done, just exit
* Create a git commit using `update_files_commit_file` for the message
* Push the commit to `update_files_branch` in `github.com/linux-system-roles/$ROLE`
  If the branch already exists, it will be pushed with `git push -f`
* Create the PR if it doesn't already exist
* Wait for review feedback

NOTE: This process may create multiple commits if you need to make edits to an
existing PR.  Use the `Squash commits and merge` functionality in the github PR
to merge.
