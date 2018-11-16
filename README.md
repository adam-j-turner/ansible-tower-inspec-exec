# Ansible Role: Inspec

## Purpose
This role allows you to execute [Inspec](https://www.inspec.io/) tests that are contained within your project when executing on Ansible Tower / AWX. It works with both Linux and Windows targets (other targets not supported / tested).

*Note:* this is intended to test live targets after having ran your playbook(s). It is not related to [Kitchen](https://docs.chef.io/kitchen.html) or [Molecule](https://molecule.readthedocs.io/en/latest/).

## Usage
### Create Inspec profile structure
Create a folder **within** your role(s) named `inspec-profile` (the name must be exact!). Then, follow the [Inspec profile structure](https://www.inspec.io/docs/reference/profiles/) to create the contents of the profile.

You will most likely have the following basic structure -
```
inspec-profile
├── controls
│   ├── test_myfeature1.rb
│   └── test_myfeature2.rb
└── inspec.yml
```
Under the controls folder, the filenames should begin with `test_`, as shown above.

### Write tests
The `test_*.rb` file(s) in the controls folder will define the tests to run against your target machines. There are dozens of [resources](https://www.inspec.io/docs/reference/resources/) you can utilize to write your tests.

Here is a basic example that confirms **port 80** is listening on the target machine:
```
describe port(80) do
  it { should be_listening }
end
```

You can have multiple tests in each file, and you can have multiple files, if necessary, for cleaner organization.

### Executing tests
You must create a task within your role(s) to execute the tests. Insert the following block at the **bottom** of your `tasks/main.yml` file:

```
- tags: inspec
  block:
    - set_fact:
        current_role_path: '{{ role_path }}'

    - name: run inspec tests
      include_role:
        name: inspec
        tasks_from: inspec-exec.yml
      vars:
        inspec_imported_role_path: '{{ current_role_path }}'
```

Inserting this block at the bottom of the `main.yml` file ensures the tests run **after** your role has completed its tasks.

### Inspec attributes (variables)
You may have several environments, & between them, servers may be listening on different ports, contain different configurations, etc. To ensure your tests have the right information, you can use **attributes**.

The Inspec role will handle converting your Ansible *vars* into Inspec *attributes*. To do this, set the **inspec_vars_list** variable and provide the list in the following format:
```
inspec_vars_list:
  - { name: 'apache_root', value: '{{ apache_root }}' }
  - { name: 'apache_listen_addr', value: '{{ apache_listen_addr }}'}
  - { name: 'apache_listen_port', value: '{{ apache_listen_port }}' }
```

**Name** is the name of the attribute and **Value** is the value of the attribute. In most cases you will just pass the name and value of your existing Ansible vars.

Include the format above when executing your tests, as such:
```
- tags: inspec
  block:
    - set_fact:
        current_role_path: '{{ role_path }}'

    - name: run inspec tests
      include_role:
        name: inspec
        tasks_from: inspec-exec.yml
      vars:
        inspec_imported_role_path: '{{ current_role_path }}'
        inspec_vars_list:
          - { name: 'apache_root', value: '{{ apache_root }}' }
          - { name: 'apache_listen_addr', value: '{{ apache_listen_addr }}'}
          - { name: 'apache_listen_port', value: '{{ apache_listen_port }}' }
```

You must write your tests in such a way that they'll utilize the attributes. For example, the following code tests that apache is listening on a particular port, as defined by an attribute:
```
apache_listen_port = attribute(
  'apache_listen_port', default: 80, description: 'The port apache is listening on.'
)

describe port(apache_listen_port) do
  it { should be_listening }
  its('protocols') { should cmp 'tcp' }
end
```

The **default** value above, `80`, is used if no attribute was provided.
