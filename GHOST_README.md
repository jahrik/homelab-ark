# Deploying and hosting a public Ark server on the homelab

## Requirements

## Ansible

## Docker

* https://github.com/TuRz4m/Ark-docker

## Molecule

* https://github.com/metacloud/molecule

Testing was done with molecule, vagrant, and virtualbox.  Using vagrant over docker because I eventually want to test this in an environment that can install and deploy a docker container running ark.  Trying to run docker in a docker container might be challenging.

    ark molecule converge

    --> Validating schema /home/wgill/homelab-ark/ark/molecule/default/molecule.yml.
    Validation completed successfully.
    --> Test matrix

    └── default
        ├── dependency
        ├── create
        ├── prepare
        └── converge

    --> Scenario: 'default'
    --> Action: 'dependency'
    Skipping, missing the requirements file.
    --> Scenario: 'default'
    --> Action: 'create'
    Skipping, instances already created.
    --> Scenario: 'default'
    --> Action: 'prepare'
    Skipping, instances already prepared.
    --> Scenario: 'default'
    --> Action: 'converge'

        PLAY [Converge] ****************************************************************

        TASK [Gathering Facts] *********************************************************
        ok: [instance]

        TASK [ark : Grab Ark-docker off github (https://github.com/TuRz4m/Ark-docker)] ***
        ok: [instance]

        PLAY RECAP *********************************************************************
        instance                   : ok=2    changed=0    unreachable=0    failed=0

