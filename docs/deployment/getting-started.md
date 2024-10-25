# Getting started with Parallel Virtual Cluster: Deploying a Cluster

For more information about what PVC is and what it's for, please see the [about page](/about).

One of PVC's design goals is administrator simplicity. Thus, it is relatively easy to get a cluster up and running in about 2 hours with only a few configuration steps, a set of nodes (with physical or remote vKVM access), and the provided tooling. This guide will walk you through setting up a simple 3-node PVC cluster from scratch, ending with a fully-usable cluster ready to provision virtual machines.

❕ **NOTE** All domains, IP addresses, etc. used in this guide are **examples**. Be sure to modify the commands and configurations to suit your specific systems and needs.

## Part One: Cluster Design, Node Procurement & Setup, and Management Host Configuration

### Cluster Design

It is important to consider the design of your PVC cluster before deploying it. While PVC is very flexible, and a cluster easily expanded later, some aspects are fixed after bootstrapping, so consider these carefully.

Read through the [Cluster Architecture documentation](/cluster-architecture). This documentation details and explains the requirements of a PVC cluster. It is important to understand these requirements before proceeding.

To set up a PVC cluster, you must pick several networks and vLANs to use in later steps, and have these ready for later steps. As outlined in that document:

  1. The "upstream" vLAN and network (ideally RFC1918 but with outbound Internet access and inbound management access).

  2. The "cluster" vLAN and network (RFC1918 unrouted).

  3. The "storage" vLAN and network (optional, RFC1918 unrouted).

Within each of these networks, pick an RFC1918 subnet for use by the cluster. The easiest configuration is to use a `/24` RFC1918 network for each, with the gateway at the top of the subnet (i.e. `.254`). PVC will assign nodes IPs sequentially by default, with `hv1` at `.1`, `hv2` at `.2`, etc. For example, if using `10.100.0.0/24` for the "upstream" network and `10.100.1.0/24` for the "cluster" network, `hv1` would be assigned `10.100.0.1` and `10.100.1.1`, `hv2` would be assigned `10.100.0.2` and `10.100.1.2`, etc.

You will also need a switch to connect the nodes, capable of vLAN trunks passing these networks into the various nodes.

### Node Procurement

0. Select your physical nodes. Some examples are outlined in the Cluster Architecture documentation linked above. For purposes of this guide, we will be using a set of 3 Dell PowerEdge R430 servers.

    ❕ **NOTE** This example selection sets some definitions below. For instance, we will refer to the "iDRAC" rather than using any other term for the integrated lights-out management/IPMI system, for clarity and consistency going forward. Adjust this to your cluster accordingly.

### Node Physical Setup

0. Connect your physical nodes to power and networking according to your cluster design worksheet. While you don't need to connect everything now (e.g. only 1 NIC in a bond or one power cable), you must be able to bring up the cluster as you would in production during the initial deployment.

0. Ensure your systems start up and are running the latest available vendor firmware. While stock firmware is usually OK, this can be the source of elusive bugs later and is easiest to do early on.

0. If applicable to your systems, create any hardware RAID arrays for system disks now. As outlined in the Cluster Architecture documentation, the system disk of a PVC system should be a hardware RAID-1 of two relatively low-capacity SSDs (120-240GB). In our example we only use a single system SSD disk, and even in production this may be fine, but will cause the loss of a node should the disk ever fail.

0. Ensure that all data OSD disks are set to "non-RAID" mode, i.e. direct host pass-through. These disks should be exposed directly to the operating system unmolested.

    ❕ **NOTE** Some RAID controllers, for instance HP "Smart" Array controllers, do not permit direct pass-through. While we do not recommend using such systems for PVC, you can work around this by creating single-disk RAID-0 volumes, though be aware that doing so will result in missing SMART data for the disks and potential instability. As outlined in the architecture documentation, avoid such systems if at all possible!

### Management Host Configuration

0. Prepare a management host for your PVC cluster(s). The management host can be nearly anything from your local workstation or laptop, to a dedicated machine, to a VM in another virtualization cluster; you can even move it into the PVC cluster later if you so choose. The purpose of the management host is to provision and maintain the cluster with `pvc-ansible` and perform tasks against it without connecting directly to the hypervisors.

    The management host must meet the following requirements:

    a. It must be running a recent version of some flavour of Debian GNU/Linux with access to BASH and Python 3. This should ideally be Debian itself (version 11 or newer) but could also be a derivative like Ubuntu. The Windows Subsystem for Linux may work, but has not been tested by the author. Non-Debian Linux, or MacOS, may be functional but the deployment of dependencies will be manual.

    b. It must have at least 10GB of free space to hold temporary files during later steps, though the overall steady-state utilization of the PVC management framework is very small (<1GB).

    c. It must have network access to the PVC cluster, both during bootstrap as well as afterwards. Ideally, it will be in the "upstream" network defined in your worksheet, but an external management host is fine. At the very least, ports 22 (SSH) and 7370 (HTTP API) must be permitted inbound to the cluster from the management host; any other traffic can be tunnelled over SSH if required.

    d. It must have [Ansible](https://www.ansible.com) installed.

0. Download the [`create-local-repo.sh` BASH script](https://github.com/parallelvirtualcluster/pvc-ansible/raw/master/create-local-repo.sh) to your management system. Do not try to pipe directly into a BASH shell or you might miss the prompts, as the script is not set up for this.

0. Run the `create-local-repo.sh` BASH script to prepare a local copy of the repositories needed for managing a PVC cluster. Follow the steps upon completion to finish preparing your local repository.

0. It is recommended, though not required, to add the PVC CLI client to the management host as well. You can do this either by downloading the [`pvc-client-cli*.deb` file from the latest release](https://github.com/parallelvirtualcluster/pvc/releases) and installing it manually with `dpkg --install`, or (recommended) by adding the PVC repository and installing via `apt` by running the following commands:

    ```
    $ sudo mkdir -p /etc/apt/keyrings
    ```

    ```
    $ wget -O- https://repo.parallelvirtualcluster.org/debian/pvc.pub | sudo gpg --dearmor --output /etc/apt/keyrings/pvc.gpg
    ```

    ```
    $ CODENAME="$( awk -F'=' '/^VERSION_CODENAME=/{ print $NF }' /etc/os-release )"
    ```

    ```
    $ echo "deb [signed-by=/etc/apt/keyrings/pvc.gpg] https://repo.parallelvirtualcluster.org/debian/ ${CODENAME} pvc" | sudo tee /etc/apt/sources.list.d/pvc.list
    ```

    ```
    $ sudo apt update
    ```

    ```
    $ sudo apt install pvc-client-cli
    ```

    ❕ **NOTE** Valid `CODENAME` values for the above commands are: `bookworm`. Ubuntu codenames are not supported; use the latest Debian codename instead.

## Part Two: Prepare your Ansible variables

0. In your local repository, edit the `hosts` file and add a new cluster. How you do so is technically up to you, but for those without advanced Ansible experience, the following format is simplest:

    ---
    [cluster1]
    hv1.cluster1.mydomain.tld
    hv2.cluster1.mydomain.tld
    hv3.cluster1.mydomain.tld

    [cluster2]
    hv1.cluster2.mydomain.tld
    hv2.cluster2.mydomain.tld
    hv3.cluster2.mydomain.tld

    Note that the hostnames given here must be the actual reachable FQDNs of the hypervisor nodes; if they do not resolve in DNS, you can use the `ansible_host=` per-entry variable to set the IP address in the "upstream" network for each node.

0. In your local repository, enter the `group_vars` directory, and create a new directory for the cluster which matches the title (inside `[`/`]` square brackets) in the above `hosts` file. For example, `cluster1`.

0. Copy the contents of the `default/` directory into the new directory. This will provide a convenient, well-documented reference for setting the various values below.

     ❕ **NOTE** We will assume for this guide that you intend to use all PVC features, and will thus explain all features. While it is possible to exclude some, that is beyond the scope of this walk-through. If you wish to skip a particular feature (for example, CPU tuning or VM autobackups), simply skip that section.

### `base.yml`

The `base.yml` file defines variables used both by the `base` role, and cluster-wide variables used by the `pvc` role and the `pvc.yml` file below.

The `default` version of this file is well-commented, and should hopefully provide a good explanation of each option. The entire file is mirrored here for posterity:

❕ **NOTE** Pay close attention to any "Use X to generate" comments; these are recommendations to use the program "X" to generate a value that must be filled.

    ---
    # The name of the Ansible cluster group, used to set file paths and determine hosts in the cluster
    # This should match the lowest-level group in the Ansible `hosts` file that defines this cluster
    cluster_group: default
 
    # Local timezone for the cluster
    timezone_location: Canada/Eastern
 
    # Cluster domain for node FQDNs
    local_domain: upstream.local
 
    # Email address of the manager of the cluster
    manager_email: team@company.tld
 
    # DNS recursive servers and search domains for nodes
    recursive_dns_servers:
      - 8.8.8.8
      - 8.8.4.4
    recursive_dns_search_domains:
      - "{{ local_domain }}"
 
    # Cluster hardware model, used in pvc_user_configuration and grub_configuration below
    cluster_hardware: default
 
    # CPU governor, sets power and performance statistics of the system CPUs; default is ondemand
    # > Valid options are (usually): conservative, ondemand, powersave, userspace, performance, schedutil
    cpu_governor: ondemand
 
    # Debian package repository URL
    debian_main_repository: http://ftp.debian.org/debian
    debian_security_repository: http://security.debian.org
    debian_pvc_repository: https://repo.parallelvirtualcluster.org/debian
 
    # Enable Prometheus metric reporting from PVC nodes; installs prometheus-node-exporter and enables
    # (unauthenticated) metrics endpoints within the PVC API. Set "no" to turn off Prometheus metric
    # functionality.
    enable_prometheus_exporters: yes
 
    # Root user password
    # > Use pwgen to generate
    root_password: ""
 
    # GRUB configuration
    # > Generally this is a good default, though some systems use console 1 for serial-over-IPMI
    #   consoles, so set this based on your actual hardware.
    grub:
      serial_console:
        "default":
          console: 0
 
    # IPMI configuration
    # > For the "pvc" user password, use pwgen to generate.
    # > Set the "pvc"user with permissions in IPMI to reboot the host as this user will be use for
    #   any fencing operations.
    # > Set the IP networking to match your expected IPMI configuration.
    ipmi:
      users:
        admin:
          username: "root"
          password: "{{ root_password }}"
        pvc:
          username: "host"
          password: ""  # Set a random password here
                        # > use pwgen to generate
      hosts:
        "hv1":                      # This name MUST match the Ansible inventory_hostname's first portion, i.e. "inventory_hostname.split('.')[0]"
          hostname: hv1-lom         # A valid short name (e.g. from /etc/hosts) or an FQDN must be used here and it must resolve to address.
                                    # PVC connects to this *hostname* for fencing.
          address: 10.100.0.101     # The IPMI address should usually be in the "upstream" network, but can be routed if required
          netmask: 255.255.255.0
          gateway: 10.100.0.254
          channel: 1                # Optional: defaults to "1" if not set; defines the IPMI LAN channel which is usually 1
        "hv2":                      # This name MUST match the Ansible inventory_hostname's first portion, i.e. "inventory_hostname.split('.')[0]"
          hostname: hv2-lom         # A valid short name (e.g. from /etc/hosts) or an FQDN must be used here and it must resolve to address.
                                    # PVC connects to this *hostname* for fencing.
          address: 192.168.100.102
          netmask: 255.255.255.0
          gateway: 192.168.100.1
          channel: 1                # Optional: defaults to "1" if not set; defines the IPMI LAN channel which is usually 1
        "hv3":                      # This name MUST match the Ansible inventory_hostname's first portion, i.e. "inventory_hostname.split('.')[0]"
          hostname: hv3-lom         # A valid short name (e.g. from /etc/hosts) or an FQDN must be used here and it must resolve to address.
                                    # PVC connects to this *hostname* for fencing.
          address: 192.168.100.103
          netmask: 255.255.255.0
          gateway: 192.168.100.1
          channel: 1                # Optional: defaults to "1" if not set; defines the IPMI LAN channel which is usually 1
 
    # IPMI user configuration
    # > Adjust this based on the specific hardware you are using; the cluster_hardware variable is
    #   used as the key in this dictionary.
    ipmi_user_configuration:
      "default":
        channel: 1                  # The IPMI user channel, usually 1
        admin:                      # Configuration for the Admin user
          id: 1                     # The user ID, usually 1 for the Admin user
          role: 0x4                 # ADMINISTRATOR privileges
          username: "{{ ipmi['users']['admin']['username'] }}"  # Loaded from the above section
          password: "{{ ipmi['users']['admin']['password'] }}"  # Loaded from the above section
        pvc:                        # Configuration for the PVC user
          id: 2                     # The user ID, usually 2 for the PVC user
          role: 0x4                 # ADMINISTRATOR privileges
          username: "{{ ipmi['users']['pvc']['username'] }}"
          password: "{{ ipmi['users']['pvc']['password'] }}"
 
    # Log rotation configuration
    # > The defaults here are usually sufficient and should not need to be changed without good reason
    logrotate_keepcount: 7
    logrotate_interval: daily
 
    # Root email name (usually "root")
    # > Can be used to send email destined for the root user (e.g. cron reports) to a real email address if desired
    username_email_root: root
 
    # Hosts entries
    # > Define any static `/etc/hosts` entries here; the provided example shows the format but should be removed
    hosts:
      - name: test
        ip: 1.2.3.4
 
    # Administrative shell users for the cluster
    # > These users will be permitted SSH access to the cluster, with the user created automatically and its
    #   SSH public keys set based on the provided lists. In addition, all keys will be allowed access to the
    #   Ansible deploy user for managing the cluster
    admin_users:
      - name: "myuser"      # Set the username
        uid: 500            # Set the UID; the first admin user should be 500, then 501, 502, etc.
        keys:
          # These SSH public keys will be added if missing
          - "ssh-ed25519 MyKey 2019-06"
        removed:
          # These SSH public keys will be removed if present
          - "ssh-ed25519 ObsoleteKey 2017-01"
 
    # Backup user SSH user keys, for remote backups separate from administrative users (e.g. rsync)
    # > Uncomment to activate this functionality.
    # > Useful for tools like BackupPC (the authors preferred backup tool) or remote rsync backups.
    #backup_keys:
    #  - "ssh-ed25519 MyKey 2019-06"
 
    # Node network definitions (used by /etc/network/interfaces and PVC)
    # > The "type" can be one of three NIC types: "nic" for raw NIC devices, "bond" for ifenslave bonds,
    #   or "vlan" for vLAN interfaces. The PVC role will write out an interfaces file matching these specs.
    # > Three names are reserved for the PVC-specific interfaces: upstream, cluster, and storage; others
    #   may be used at will to describe the other devices. These devices have IP info which is then written
    #   into `pvc.conf`.
    # > All devices should be using the predictable device name format (i.e. enp1s0f0 instead of eth0). If
    #   you do not know these names, consult the manual of your selected node hardware, or boot a Linux
    #   LiveCD to see the generated interface configuration.
    # > This example configuration is one the author uses frequently, to demonstrate all possible options.
    #   First, two base NIC devices are set with some custom ethtool options; these are optional of course.
    #   The "timing" value for a "custom_options" entry must be "pre" or "post". The command can include $IFACE
    #   which is written as-is (to be interpreted by Debian ifupdown at runtime).
    #   Second, a bond interface is created on top of the two NIC devices in 802.3ad (LACP) mode with high MTU.
    #   Third, the 3 PVC interfaces are created as vLANs (1000, 1001, and 1002) on top of the bond.
    #   This should cover most normal usecases, though consult the template files for more detail if needed.
    networks:
      enp1s0f0:
        device: enp1s0f0
        type: nic
        mtu: 9000  # Forms a post-up ip link set $IFACE mtu statement; a high MTU is recommended for optimal backend network performance
        custom_options:
          - timing: pre  # Forms a pre-up statement
            command: ethtool -K $IFACE rx-gro-hw off
          - timing: post  # Forms a post-up statement
            command: sysctl -w net.ipv6.conf.$IFACE.accept_ra=0
      enp1s0f1:
        device: enp1s0f1
        type: nic
        mtu: 9000  # Forms a post-up ip link set $IFACE mtu statement; a high MTU is recommended for optimal backend network performance
        custom_options:
          - timing: pre  # Forms a pre-up statement
            command: ethtool -K $IFACE rx-gro-hw off
          - timing: post  # Forms a post-up statement
            command: sysctl -w net.ipv6.conf.$IFACE.accept_ra=0
      bond0:
        device: bond0
        type: bond
        bond_mode: 802.3ad  # Can also be active-backup for active-passive failover, but LACP is advised
        bond_devices:
          - enp1s0f0
          - enp1s0f1
        mtu: 9000  # Forms a post-up ip link set $IFACE mtu statement; a high MTU is recommended for optimal backend network performance
      upstream:
        device: vlan1000
        type: vlan
        raw_device: bond0
        mtu: 1500                     # Use a lower MTU on upstream for compatibility with upstream networks to avoid fragmentation
        domain: "{{ local_domain }}"  # This should be the local_domain for the upstream network
        subnet: 10.100.0.0            # The CIDR subnet address without the netmask
        netmask: 24                   # The CIDR netmask
        floating_ip: 10.100.0.250     # The floating IP used by the cluster primary coordinator; should be a high IP that won't conflict with any node IDs
        gateway_ip: 10.100.0.254      # The default gateway IP
      cluster:
        device: vlan1001
        type: vlan
        raw_device: bond0
        mtu: 9000                     # Use a higher MTU on cluster for performance
        domain: pvc-cluster.local     # This domain is arbitrary; using this default example is a good practice
        subnet: 10.0.0.0              # The CIDR subnet address without the netmask; this should be an UNROUTED network (no gateway)
        netmask: 24                   # The CIDR netmask
        floating_ip: 10.0.0.254       # The floating IP used by the cluster primary coordinator; should be a high IP that won't conflict with any node IDs
      storage:
        device: vlan1002
        type: vlan
        raw_device: bond0
        mtu: 9000                     # Use a higher MTU on storage for performance
        domain: pvc-storage.local     # This domain is arbitrary; using this default example is a good practice
        subnet: 10.0.1.0              # The CIDR subnet address without the netmask; this should be an UNROUTED network (no gateway)
        netmask: 24                   # The CIDR netmask
        floating_ip: 10.0.1.254       # The floating IP used by the cluster primary coordinator; should be a high IP that won't conflict with any node IDs

### `pvc.yml`

The `pvc.yml` file defines variables used by the `pvc` role to create individual node configurations and populate the various configurations of the PVC services.

The `default` version of this file is well-commented, and should hopefully provide a good explanation of each option. The entire file is mirrored here for posterity:

❕ **NOTE** A large number of options in this file are commented out; you only need to uncomment these if you wish to change the specified default value.

❕ **NOTE** Pay close attention to any "Use X to generate" comments; these are recommendations to use the program "X" to generate a value that must be filled.

Of special note is the `pvc_nodes` section. This must contain a listing of all nodes in the cluster. For most clusters, start only with the 3 (or 5 if a large cluster is planned) coordinator nodes, and add the remainder in later.

    ---
    # Logging configuration (uncomment to override defaults)
    # These default options are generally best for most clusters; override these if you want more granular
    # control over the logging output of the PVC system.
    #pvc_log_to_stdout: True                  # Log to stdout (i.e. journald)
    #pvc_log_to_file: False                   # Log to files in /var/log/pvc
    #pvc_log_to_zookeeper: True               # Log to Zookeeper; required for 'node log' commands to function, but writes a lot of data to Zookeeper - disable if using very small system disks
    #pvc_log_colours: True                    # Log colourful prompts for states instead of text
    #pvc_log_dates: False                     # Log dates (useful with log_to_file, not useful with log_to_stdout as journald adds these)
    #pvc_log_keepalives: True                 # Log each keepalive event every pvc_keepalive_interval seconds
    #pvc_log_keepalive_cluster_details: True  # Log cluster details (VMs, OSDs, load, etc.) duing keepalive events
    #pvc_log_keepalive_plugin_details: True   # Log health plugin details (messages) suring keepalive events
    #pvc_log_console_lines: 1000              # The number of VM console log lines to store in Zookeeper for 'vm log' commands.
    #pvc_log_node_lines: 2000                 # The number of node log lines to store in Zookeeper for 'node log' commands.
 
    # Timing and fencing configuration (uncomment to override defaults)
    # These default options are generally best for most clusters; override these if you want more granular
    # control over the timings of various areas of the cluster, for instance if your hardware is slow or error-prone.
    # DO NOT lower most of these values; this will NOT provide "more reliability", but the contrary.
    #pvc_vm_shutdown_timeout: 180             # Number of seconds before a 'shutdown' VM is forced off
    #pvc_keepalive_interval: 5                # Number of seconds between keepalive ticks
    #pvc_monitoring_interval: 15              # Number of seconds between monitoring plugin runs
    #pvc_fence_intervals: 6                   # Number of keepalive ticks before a node is considered dead
    #pvc_suicide_intervals: 0                 # Number of keepalive ticks before a node consideres itself dead and forcibly restarts itself (0 to disable, recommended)
    #pvc_fence_successful_action: migrate     # What to do with VMs when a fence is successful (migrate, None)
    #pvc_fence_failed_action: None            # What to do with VMs when a fence is failed (migrate, None) - migrate is DANGEROUS without pvc_suicide_intervals set to < pvc_fence_intervals
    #pvc_migrate_target_selector: mem         # The selector to use for migrating VMs if not explicitly set by the VM; one of mem, memfree, load, vcpus, vms
 
    # Client API basic configuration
    pvc_api_listen_address: "{{ pvc_upstream_floatingip }}"  # This should usually be the upstream floating IP
    pvc_api_listen_port: "7370"                              # This can be any port, including low ports, if desired, but be mindful of port conflicts
    pvc_api_secret_key: ""                                   # Use pwgen to generate
 
    # Client API user tokens
    # Create a token (random UUID or password) for each user you wish to have access to the PVC API.
    # The first token will always be used for the "local" connection, and thus at least one token MUST be defined.
    # WARNING: All tokens function at the same privilege level and provide FULL CONTROL over the cluster. Keep them secret!
    pvc_api_enable_authentication: True
    pvc_api_tokens:
      - description: "myuser"                           # The description is purely cosmetic for current iterations of PVC
        token: "a3945326-d36c-4024-83b3-2a8931d7785a"   # The token should be random for security; use uuidgen or pwgen to generate
 
    # PVC API SSL configuration
    # Use these options to enable SSL for the API listener, providing security over WAN connections.
    # There are two options for defining the SSL certificate and key to use:
    #   a) Set both pvc_api_ssl_cert_path and pvc_api_ssl_key_path to paths to an existing SSL combined (CA + intermediate + cert) certificate and key, respectively, on the system.
    #   b) Set both pvc_api_ssl_cert and pvc_api_ssl_key to the raw PEM-encoded contents of an SSL combined (CA + intermediate + cert) certificate and key, respectively, which will be installed under /etc/pvc.
    # If the _path options are non-empty, the raw entries are ignored and will not be used.
    pvc_api_enable_ssl: False   # Enable SSL listening; this is highly recommended when using the API over the Internet!
    pvc_api_ssl_cert_path: ""   # Set a path here, or...
    pvc_api_ssl_cert: >
      # Enter a A RAW CERTIFICATE FILE content, installed to /etc/pvc/api-cert.pem
    pvc_api_ssl_key_path: ""    # Set a path here, or...
    pvc_api_ssl_key: >
      # Enter a A RAW KEY FILE content, installed to /etc/pvc/api-key.pem
 
    # Ceph storage configuration
    pvc_ceph_storage_secret_uuid: ""            # Use uuidgen to generate
 
    # Database configuration
    pvc_dns_database_name: "pvcdns"             # Should usually be "pvcdns" unless there is good reason to change it
    pvc_dns_database_user: "pvcdns"             # Should usually be "pvcdns" unless there is good reason to change it
    pvc_dns_database_password: ""               # Use pwgen to generate
    pvc_api_database_name: "pvcapi"             # Should usually be "pvcapi" unless there is good reason to change it
    pvc_api_database_user: "pvcapi"             # Should usually be "pvcapi" unless there is good reason to change it
    pvc_api_database_password: ""               # Use pwgen to generate
    pvc_replication_database_user: "replicator" # Should be "replicator" for Patroni
    pvc_replication_database_password: ""       # Use pwgen to generate
    pvc_superuser_database_user: "postgres"     # Should be "postgres"
    pvc_superuser_database_password: ""         # Use pwgen to generate
 
    # Network routing configuration
    # The ASN should be a private ASN number, usually "65500"
    # The list of routers are those which will learn routes to the PVC client networks via BGP;
    # they should speak BGP and allow sessions from the PVC nodes.
    # If you do not have any upstream BGP routers, e.g. if you wish to use static routing to and from managed networks, leave the list empty
    pvc_asn: "65500"
    pvc_routers:
      - "10.100.0.254"
 
    # PVC Node list
    # Every node configured with this playbook must be specified in this list.
    pvc_nodes:
      - hostname: "hv1"  # This name MUST match the Ansible inventory_hostname's first portion, i.e. "inventory_hostname.split('.')[0]"
        is_coordinator: yes  # Coordinators should be set to "yes", hypervisors to "no"
        node_id: 1  # Should match the number portion of the hostname
        router_id: "10.100.0.1"
        upstream_ip: "10.100.0.1"
        cluster_ip: "10.0.0.1"
        storage_ip: "10.0.1.1"
        ipmi_host: "{{ ipmi['hosts']['hv1']['hostname'] }}"  # Note the node hostname as the key here
        ipmi_user: "{{ ipmi['users']['pvc']['username'] }}"
        ipmi_password: "{{ ipmi['users']['pvc']['password'] }}"
        cpu_tuning:         # Example of cpu_tuning overrides per-node, only relevant if enabled; see below
          system_cpus: 2    # Number of CPU cores (+ their hyperthreads) to allocate to the system
          osd_cpus: 2       # Number of CPU cores (+ their hyperthreads) to allocate to the storage OSDs
      - hostname: "hv2" # This name MUST match the Ansible inventory_hostname's first portion, i.e. "inventory_hostname.split('.')[0]"
        is_coordinator: yes
        node_id: 2
        router_id: "10.100.0.2"
        upstream_ip: "10.100.0.2"
        cluster_ip: "10.0.0.2"
        storage_ip: "10.0.1.2"
        ipmi_host: "{{ ipmi['hosts']['hv2']['hostname'] }}"  # Note the node hostname as the key here
        ipmi_user: "{{ ipmi['users']['pvc']['username'] }}"
        ipmi_password: "{{ ipmi['users']['pvc']['password'] }}"
      - hostname: "hv3" # This name MUST match the Ansible inventory_hostname's first portion, i.e. "inventory_hostname.split('.')[0]"
        is_coordinator: yes
        node_id: 3
        router_id: "10.100.0.3"
        upstream_ip: "10.100.0.3"
        cluster_ip: "10.0.0.3"
        storage_ip: "10.0.1.3"
        ipmi_host: "{{ ipmi['hosts']['hv3']['hostname'] }}"  # Note the node hostname as the key here
        ipmi_user: "{{ ipmi['users']['pvc']['username'] }}"
        ipmi_password: "{{ ipmi['users']['pvc']['password'] }}"
 
    # Bridge device entry
    # This device is used when creating bridged networks. Normal managed networks are created on top of the
    # "cluster" interface defined below, however bridged networks must be created directly on an underlying
    # non-vLAN network device. This can be the same underlying device as the upstream/cluster/storage networks
    # (especially if the upstream network device is not a vLAN itself), or a different device separate from the
    # other 3 main networks.
    pvc_bridge_device: bond0        # Replace based on your network configuration
    pvc_bridge_mtu: 1500            # Replace based on your network configuration; bridges will have this MTU by default unless otherwise specified
 
    # SR-IOV device configuration
    # SR-IOV enables the passing of hardware-virtualized network devices (VFs), created on top of SR-IOV-enabled
    # physical NICs (PFs), into virtual machines. SR-IOV is a complex topic, and will not be discussed in detail
    # here. Instead, the SR-IOV mode is disabled by default and a commented out example configuration is shown.
    pvc_sriov_enable: False
    #pvc_sriov_device:
    #  - phy: ens1f0
    #    mtu: 9000
    #    vfcount: 6
 
    # Memory tuning
    # > ADVANCED TUNING: For most users, this is unnecessary and PVC will run fine with the default memory
    #   allocations. Uncomment these options only low-memory situations (nodes with <32GB RAM).
    #
    # OSD memory limit - 939524096 (~900MB) is the lowest possible value; default is 4GB.
    # > This option is *only* applied at cluster bootstrap and cannot be changed later
    #   here, only by editing the `files/ceph/<cluster>/ceph.conf` file directly.
    #pvc_osd_memory_limit: 939524096
    #
    # Zookeeper heap memory limit, sets Xms and Xmx values to the Java process; default is 512M.
    # > WARNING: Unless you have an extremely limited amount of RAM, changing this setting is NOT RECOMMENDED.
    #            Lowering the heap limit may cause poor performance or crashes in Zookeeper during some tasks.
    #pvc_zookeeper_heap_limit: 128M   # 1/4 of default
    #
    # Zookeeper stack memory limit, sets Xss value to the Java process; default is 1024M.
    # > WARNING: Unless you have an extremely limited amount of RAM, changing this setting is NOT RECOMMENDED.
    #            Lowering the stack limit may cause poor performance or crashes in Zookeeper during some tasks.
    #pvc_zookeeper_stack_limit: 256M  # 1/4 of default
 
    # CPU tuning
    # > ADVANCED TUNING: These options are strongly recommended due to the performance gains possible, but
    #   most users would be able to use the default without too much issue. Read the following notes
    #   carefully to determine if this setting should be enabled in your cluster.
    # > NOTE: CPU tuning playbooks require jmespath (e.g. python3-jmespath) installed on the controller node
    # > Defines CPU tuning/affinity options for various subsystems within PVC. This is useful to
    #   help limit the impact that noisy elements may have on other elements, e.g. busy VMs on
    #   OSDs, or system processes on latency-sensitive VMs.
    # > To enable tuning, set enabled to yes.
    # > Within "nodes", two counts are specified:
    #    * system_cpus: The number of CPUs to assign to the "system" slice, i.e. all non-VM,
    #                   non-OSD processes on the system. Should usually be at least 2, and be
    #                   higher on the coordinators of larger clusters (i.e. >5 nodes).
    #    * osd_cpus: The number of CPUs to assign to the "osd" slice, i.e. all OSD processes.
    #                Should be at least 1 per OSD, and ideally 2 per OSD for best performance.
    #   A third count, for the VM CPUs, is autogenerated based on the total node CPU count and
    #   the above two values (using all remaining CPUs).
    # > Tuning is done based on cores; for systems with SMT (>1 thread-per-core), all SMTs within
    #   a given core are also assigned to the same CPU set. So for example, if the "system" group is
    #   assigned 2 system_cpus, there are 16 cores, and there are 2 threads per core, the list will be:
    #     0,1,16,17
    #   leveraging the assumption that Linux puts all cores before all threads.
    # > This tuning section under "nodes" is global to the cluster; to override these values on
    #   a per-node basis, use the corresponding "cpu_tuning" section of a given "pvc_nodes" entry
    #   as shown below.
    # > If disabled after being enabled, the tuning configurations on each node will be removed
    #   on the next run. A reboot of all nodes is required to fully disable the tuning.
    # > IMPORTANT NOTE: Enabling CPU tuning requires a decent number of CPU cores on the system. For very
    #   low-spec systems (i.e. less than at least 12 cores per node), it is advisable to leave this tuning
    #   off, as otherwise very few cores will actually be allocated to VMs. With a larger number (>16 or so),
    #   this tuning is likely to greatly increase storage performance, though balance between the VM workload
    #   and total number of cores must be carefully considered.
    cpu_tuning:
      enabled: no           # Disable or enable CPU tuning; recommended to enable for optimal storage performance
      nodes:
        system_cpus: 2      # Set based on your actual system configuration (min 2, increase on coordinators if many nodes)
        osd_cpus: 2         # Set based on your actual number of OSDs (for optimal performance, 2 per OSD)
 
    # PVC VM autobackups
    # > PVC supports autobackups, which can perform automatic snapshot-level VM backups of selected
    #   virtual machines based on tags. The backups are fully managed on a consistent schedule, and
    #   include both full and incremental varieties.
    # > To solve the shared storage issue and ensure backups are taken off-cluster, automaticmounting
    #   of remote filesystems is supported by autobackup.
    pvc_autobackup:
      # Enable or disable autobackup
      # > If disabled, no timers or "/etc/pvc/autobackup.yaml" configuration will be installed, and any
      #   existing timers or configuration will be REMOVED on each run (even if manually created).
      # > Since autobackup is an integrated PVC CLI feature, the command will always be available regardless
      #   of this setting, but without this option enabled, the lack of a "/etc/pvc/autobackup.yaml" will
      #   prevent its use.
      enabled: no
      # Set the backup root path and (optional) suffix
      # > This directory will be used for autobackups, optionally suffixed with the suffix if it is present
      # > If remote mounting is enabled, the remote filesystem will be mounted at the root path; if it is
      #   not enabled, there must be a valid large(!) filesystem mounted on all coordinator nodes at this
      #   path.
      # > The suffix can be used to allow a single backup root path to back up multiple clusters without
      #   conflicts should those clusters share VM names. It is optional unless this matches your situation.
      # > The path "/tmp/backups" is usually recommended for remote mounting
      # > NOTE: If you specify it, the suffix must begin with a '/', but is relative to the root path!
      root_path:   "/tmp/backups"
      root_suffix: "/mycluster"
      # Set the VM tag(s) which will be selected for autobackup
      # > Autobackup selects VMs based on their tags. If a VM has a tag present in this list, it will be
      #   selected for autobackup at runtime; if not it will be ignored.
      # > Usually, the tag "autobackup" here is sufficient; the administrator should then add this tag
      #   to any VM(s) they want to use autobackups. However, any tag may be specified to keep the tag list
      #   cleaner and more focused, should the administrator choose to.
      tags:
        - autobackup
      # Autobackup scheduling
      schedule:
        # Backups are performed at regular intervals via a systemd timer
        # > Optionally, forced-full backups can also be specified, which ensures consistent rotation
        #   between VMs regardless of when they are added; if forced_full_time is empty or missing, this
        #   feature is disabled
        # > This default schedule performs a (forced) full backup every Monday at midnight, then normal backups
        #   every other day at midnight (these may be full or incremental depending on the options below
        # > These options use a systemd timer date string; see "man systemd.time" for details
        normal_time:      "Tue..Sun *-*-* 0:0:00"
        forced_full_time: "Mon *-*-* 0:0:00"
        # The interval between full backups determines which backups are full and which are incrementals
        # > When a backup is run, if there are this many (inclusive) backups since the last full backup,
        #   then a new full backup is taken and rotation occurs; otherwise, an incremental backup is taken
        # > For example, a value of 1 means every backup is a full backup; a value of 2 means every other
        #   bakcup is a full backup; a value of 7 means every 7th backup is a full backup (i.e. once per week
        #   with a daily backup time).
        full_interval: 7
        # The retention count specifies how many full backups should be kept
        # > Retention cleanup is run after each full backup, and thus, that backup is counted in this number
        # > For example, a value of 2 means that there will always be at least 2 full backups. When a new
        #   full backup is taken, the oldest (i.e. 3rd) full backup is removed.
        # > When a full backup is removed, all incremental backups with that full backup as their parent are
        #   also removed.
        # > Thus, this schedule combined with a full_interval of 7 ensures there is always 2 full weekly backups,
        #   plus at least 1 full week's worth of incremental backups.
        full_retention: 2
      # Set reporting options for autobackups
      # NOTE: By default, pvc-ansible installs a local Postfix MTA and Postfix sendmail to send emails
      # This may not be what you want! If you want an alternate sendmail MTA (e.g. msmtp) you must install it
      # yourself in a custom role!
      reporting:
        # Enable or disable email reporting; if disabled ("no"), no reports are ever sent
        enabled: no
        # Email a report to these addresses; at least one MUST be specified if enabled
        emails:
          - myuser@domain.tld
          - otheruser@domain.tld
        # Email a report on the specified jobs
        report_on:
            # Send a report on a forced_full backup (usually, weekly)
            forced_full: yes
            # Send a report on a normal backup (usually, daily)
            normal: yes
      # Configure automatic mounting support
      # > PVC autobackup features the ability to automatically and dynamically mount and unmount remote
      #   filesystems, or, indeed, perform any arbitrary pre- or post-run tasks, using a set of arbitrary
      #   commands
      # > Automatic mountoing is optional if you choose to use a static mount on all PVC coordinators
      # > While the examples here show absolute paths, that is not required; they will run with the $PATH of the
      #   executing environment (either the "pvc" command on a CLI or a cron/systemd timer)
      # > A "{backup_root_path}" f-string/str.format type variable MAY be present in any cmds string to represent
      #   the above configured root backup path, and is which is interpolated at runtime
      # > If multiple commands are given, they will be executed in the order given; if no commands are given,
      #   nothing is executed, but the keys MUST be present
      auto_mount:
        # Enable or disable automatic mounting
        enabled: no
        # These Debian packages will be automatically installed if automatic mounting is enabled
        packages:
          # This example installs nfs-common, required for NFS mounts
          # - nfs-common
        # These commands are executed at the start of the backup run and should mount a filesystem or otherwise
        # prepare the system for the backups
        mount_cmds:
          # This example shows an NFS mount leveraging the backup_root_path variable
          # - "/usr/sbin/mount.nfs -o nfsvers=3 10.0.0.10:/backups {backup_root_path}"
          # This example shows an SSHFS mount leveraging the backup_root_path variable
          # - "/usr/bin/sshfs user@hostname:/path {backup_root_path} -o default_permissions -o sshfs_sync -o IdentityFile=/path/to/id_rsa"
        # These commands are executed at the end of the backup run and should unmount a filesystem
        unmount_cmds:
          # This example shows a generic umount leveraging the backup_root_path variable
          # - "/usr/bin/umount {backup_root_path}"
          # This example shows an fusermount3 unmount (e.g. for SSHFS) leveraging the backup_root_path variable
          # - "/usr/bin/fusermount3 -u {backup_root_path}"
 
    # Configuration file networks
    # > Taken from base.yml's configuration; DO NOT MODIFY THIS SECTION.
    pvc_upstream_device: "{{ networks['upstream']['device'] }}"
    pvc_upstream_mtu: "{{ networks['upstream']['mtu'] }}"
    pvc_upstream_domain: "{{ networks['upstream']['domain'] }}"
    pvc_upstream_netmask: "{{ networks['upstream']['netmask'] }}"
    pvc_upstream_subnet: "{{ networks['upstream']['subnet'] }}"
    pvc_upstream_floatingip: "{{ networks['upstream']['floating_ip'] }}"
    pvc_upstream_gatewayip: "{{ networks['upstream']['gateway_ip'] }}"
    pvc_cluster_device: "{{ networks['cluster']['device'] }}"
    pvc_cluster_mtu: "{{ networks['cluster']['mtu'] }}"
    pvc_cluster_domain: "{{ networks['cluster']['domain'] }}"
    pvc_cluster_netmask: "{{ networks['cluster']['netmask'] }}"
    pvc_cluster_subnet: "{{ networks['cluster']['subnet'] }}"
    pvc_cluster_floatingip: "{{ networks['cluster']['floating_ip'] }}"
    pvc_storage_device: "{{ networks['storage']['device'] }}"
    pvc_storage_mtu: "{{ networks['storage']['mtu'] }}"
    pvc_storage_domain: "{{ networks['storage']['domain'] }}"
    pvc_storage_netmask: "{{ networks['storage']['netmask'] }}"
    pvc_storage_subnet: "{{ networks['storage']['subnet'] }}"
    pvc_storage_floatingip: "{{ networks['storage']['floating_ip'] }}"

## Part Three: Prepare the Installer ISO and install the node base OS

0. Install the Debian live-build package `live-build`, which is used to generate live ISO images.

0. In your local repository, enter the `pvc-installer` directory.

0. Run the `./buildiso.sh` script to generate a live installer ISO. This will take anywhere from 10 to 30 minutes to run depending on your system performance.

    ❔ **Why?** *Why not provide a pre-built ISO?* Debian can change frequently, both in terms of releases used for the live installer (we've gone through 3 since first creating PVC) and also just normal security updates. Plus the live ISO is relatively large (about 700MB), so distributing that would be a burden. Generating these on-demand, using the latest version of the script, is the best method to ensure it's up-to-date and ready for your system.

    ❕ **NOTE** The output file will be dated, for example `pvc-installer_2024-09-05_amd64.iso`.

0. Mount the generated ISO onto your nodes. This can be accomplished in several ways, depending on what your server supports; choose the one that makes the most sense for your environment.

    a. Virtual media in the iDRAC or similar out-of-band management system's console interface.

    a. Virtual media in the iDRAC or similar out-of-band management system over HTTP, NFS, or similar.

    a. Writing the ISO to a USB flash drive (or several).

    a. Writing the ISO to a CD-ROM or DVD-ROM (or several).

0. If you have not already done so, prepare you system disks as a (hardware) RAID-1, so it is ready to be installed to. Note that PVC does not support software RAID (`mdraid` or similar).

0. Boot each of your **coordinator** nodes off the ISO. We will install and configure these first, then add any additional hypervisor nodes later if required. Essentially each node that you configured in the `pvc_nodes` list in `pvc.yml` should be booted off the ISO.

0. Once the ISO boots, the installer script will run, and you will be prompted for several pieces of information about each node. Repeat this step for each node.

    a. For the hostname, ensure this is a FQDN that matches the name set in your `hosts` and `group_vars` - specifically, the short name (e.g. `hv1`) followed by the `local_domain` (e.g. `cluster1.mydomain.tld`).

    b. For disks, all options presented are supported, though the defaults are recommended.

    c. For networking, during this initial state we only need a single interface to get basic connectivity and prepare for Ansible. Generally speaking, setup and bootstrapping is easier if you have a dedicated "setup" NIC in a network directly reachable by your management host (`upstream` or another network), then allow the `pvc-ansible` system to configure the "main" interfaces from there. If this is not possible, you can configure both a bond and a vLAN on top during the installer to pre-configure your `upstream` interface. You can use either DHCP (if you are using a dedicated "setup" network) or a static IP (if you are directly configuring the `upstream` network now).

        ❕ **NOTE** The installer won't proceed until networking is up. If you need to stop and troubleshoot, you can launch another virtual console using Ctrl+Alt+F2 or similar, cancel the installer script, and interrogate the installer environment in more detail.

    d. For the Debian configuration, you can choose a specific mirror if you wish, but otherwise the defaults are recommended. If you require any additional packages for the system to boot (e.g. firmware, drivers, etc.), ensure you list them in the additional packages step.

    e. For SSH, you have two options: the recommended option is to provide an HTTP URL to an SSH `id_x.pub` file or existing `authorized_keys` file; or you can provide a password which can then be used to upload keys later. Keys will be required for long-term maintenance of the system, so it is recommended that you prepare this now, place the public key(s) in a file on a reachable web server, and use that.

0. Installation will now begin, and once completed, there will be an option to launch a shell in the new environment if required. If not, you will be prompted with the final notes. Take note of the default root password: this will provide you access should the networking be unreachable, then press Enter to reboot into the newly installed system. Repeat this step for each node.

## Part Four - Initial bootstrap with Ansible

0. Make note of the IP addresses of the newly installed nodes, and configure DNS, `/etc/hosts`, or Ansible `ansible_host=` hostvars to map these IP addresses to the hostnames set in the Ansible `hosts` and `group_vars` files.

0. Verify connectivity from your administrative host to the installed nodes, including SSH access as the `deploy` user. Accept their host keys as required before proceeding, or Ansible will likely complain. If you did not configure SSH key auth during the PVC installer process, log in with the configured password and set that up now, as it greatly simplifies Ansible configuration.

0. Verify your `group_vars` one last time, as many aspects of the cluster cannot be changed after bootstrapping.

0. Perform the initial bootstrap. From your local configuration repository directory, execute the following `ansible-playbook` command, replacing `<cluster_name>` with the Ansible group name from the `hosts` file. Make special note of the additional `bootstrap=yes` variable, which tells the playbook that this is an initial bootstrap run.

    ```
    $ ansible-playbook -v -i hosts -l <cluster_name> -e bootstrap=yes pvc.yml
    ```

    ⚠️ **WARNING** Never run the playbook with the `-e bootstrap=yes` option against an active, already-bootstrapped cluster. This will have **disastrous consequences** including the **loss of all data** in the Ceph system as well as any configured VMs, networks, etc.

0. Wait for the Ansible playbook run to finish. Once completed, the cluster bootstrap will be finished, and all 3 nodes will have rebooted into a working PVC cluster. If any errors occur, carefully evaluate them and re-run the playbook (with `-o bootstrap=yes` - your cluster is not active yet!) as required.

0. Download and install the CLI client package (`pvc-client-cli_*.deb`) on your administrative host, and add and verify connectivity to the cluster with `pvc connection add`. This will also verify that the API is working. You will need to know the cluster upstream floating IP address you configured in the `networks` section of the `base.yml` playbook, and if you configured SSL or authentication for the API in your `group_vars`, adjust the first command as needed (see `pvc connection add -h` for details). A human-readable description can also be specified, which is useful if you manage multiple clusters and their names become unwieldy.

    ```
    $ pvc connection add -a <upstream_floating_ip> -d "My first PVC cluster" cluster1
    ```

    ```
    $ pvc -c mycluster node list
    ```

    You can also set a default cluster for your shell session by exporting the `PVC_CLUSTER` environment variable, to avoid requiring `-c cluster` with every subsequent command:

    ```
    $ export PVC_CLUSTER="cluster1"
    ```

    ❕ **NOTE** It is fully possible to administer the cluster from the nodes themselves via SSH should you so choose, to avoid requiring the PVC client on your local machine.

## Part Five - Configuring the Ceph storage cluster

0. Determine the Ceph OSD block devices on each host via an `ssh` shell. For instance, use `lsblk` or check `/dev/disk/by-path` to show the block devices by their physical SAS/SATA bus location, and obtain the relevant `/dev/sdX` name for each disk you wish to be a Ceph OSD on each host. You will need at least 1 OSD disk per coordinator node at a minimum.

0. Cofigure an OSD device for each data disk in each host. The general command is:

    ```
    $ pvc storage osd add <node> <device>
    ```

    For example, if each node has two data disks, as `/dev/sdb` and `/dev/sdc`, run the commands as follows to add the first disk to each node, then the second disk to each node:

    ```
    $ pvc storage osd add --weight 1.0 pvchv1 /dev/sdb
    ```

    ```
    $ pvc storage osd add --weight 1.0 pvchv2 /dev/sdb
    ```

    ```
    $ pvc storage osd add --weight 1.0 pvchv3 /dev/sdb
    ```

    ```
    $ pvc storage osd add --weight 1.0 pvchv1 /dev/sdc
    ```

    ```
    $ pvc storage osd add --weight 1.0 pvchv2 /dev/sdc
    ```

    ```
    $ pvc storage osd add --weight 1.0 pvchv3 /dev/sdc
    ```

    ❕ **NOTE** On the CLI, the `--weight` argument is optional, and defaults to `1.0`. In the API, it must be specified explicitly, but the CLI sets a default value. OSD weights determine the relative amount of data which can fit onto each OSD. Under normal circumstances, you would want all OSDs to be of identical size, and hence all should have the same weight. If your OSDs are instead different sizes, the weight should be proportional to the size, e.g. `1.0` for a 100GB disk, `2.0` for a 200GB disk, etc. For more details, see the [Cluster Architecture](/cluster-architecture) and Ceph documentation.

    ❕ **NOTE** You can add OSDs in any order you wish, for instance you can add the first OSD to each node and then add the second to each node, or you can add all nodes' OSDs together at once like the example. This ordering does not affect the cluster in any way.

0. Verify that the OSDs were successfully added and are functional (`up` and `in`):

    ```
    $ pvc storage osd list
    ```

0. Create an RBD pool to store VM images on. The general command is:

    ```
    $ pvc storage pool add <name> <placement_groups>
    ```

    ❕ **NOTE** Ceph placement groups are a complex topic; as a general rule it's easier to grow than shrink, so start small and grow as your cluster grows. The following are some good starting numbers for 3-node clusters, though the Ceph documentation and the [Ceph placement group calculator](https://ceph.com/pgcalc/) are advisable for anything more complex. There is a trade-off between CPU usage and the number of total PGs for all pools in the cluster, with more PGs meaning more CPU usage.  

    * 3-6 OSDs total: 64 PGs (1 pool) or 32 PGs (2 or more pools, each)  
    * 9+ OSDs total: 128 PGs (1 pool) or 64 PGs (2 or more pools, each)  

    For example, to create a pool named `vms` with 128 placement groups, run the command as follows:

    ```
    $ pvc storage pool add vms 128
    ```

    ❕ **NOTE** As detailed in the [cluster architecture documentation](/cluster-architecture), you can also set a custom replica configuration for each pool if the default of 3 replica copies with 2 minimum copies is not acceptable. See `pvc storage pool add -h` or that document for full details.

0. Verify that the pool was successfully added:

    ```
    $ pvc storage pool list`
    ```

## Part Six - Creating client networks

0. Determine a domain name and IPv4, and/or IPv6 network for your first client network, and any other client networks you may wish to create. These networks must not overlap with the cluster networks. For full details on the client network types, see the [cluster architecture documentation](/cluster-architecture).

0. Create the virtual network. There are many options here, so see `pvc network add -h` for details. Generally, you will create either `managed` or `bridged` networks:

    * To create a managed (EVPN VXLAN) network `10000` with subnet `10.100.0.0/24`,  gateway `.1` and DHCP from `.100` to `.199`, run the command as follows:

        ```
        $ pvc network add 10000 --type managed --description my-managed-network --domain myhosts.local --ipnet 10.100.0.0/24 --gateway 10.100.0.1 --dhcp --dhcp-start 10.100.0.100 --dhcp-end 10.100.0.199
        ```

    * To create a bridged (switch-configured, tagged VLAN, with no PVC management of IPs) network `200`, run the command as follows:

        ```
        $ pvc network add 200 --type bridged --description my-bridged-network
        ```

    ❕ **NOTE** Network descriptions cannot contain spaces or special characters; keep them short, sweet, and dash or underscore delimited.

    ❕ **NOTE** At least one `managed` network with DHCP support will be required to use the PVC provisioner functionality.

0. Verify that the network(s) were successfully added:

    ```
    $ pvc network list
    ```

0. If you are using managed networks, on the upstream router, configure one of:

    a) A BGP neighbour relationship with the cluster upstream floating address to automatically learn routes.

    b) Static routes for the configured client IP networks towards the cluster upstream floating address.

0. On the upstream router, if required, configure NAT for the managed client networks.

0. Verify the client networks are reachable by pinging the gateway IP from outside the cluster.


## Activate the nodes

0. Set all 3 nodes to `ready` state, allowing them to run virtual machines. The general command is:

    ```
    $ pvc node ready <node>
    ```

Congratulations, you now have a PVC cluster, ready to run your VMs.

### Adding additional nodes

If you want to add additional non-coordinator nodes, you can do so now. Run through the node installer on them, and then bootstrap them using the Ansible playbook **without** the `-e bootstrap` option. The nodes will be automatically configured and join the cluster when they reboot, ready to provide compute for VMs or additional OSD disks for storage.

## Next steps

For the next steps, see the [Provisioner manual](/manuals/provisioner) for details on how to use the PVC provisioner to create new Virtual Machines, as well as the [CLI manual](/manuals/cli) and [API manual](/manuals/api) for details on day-to-day usage of PVC.
