

# Puppet Master

rpm -Uvh https://yum.puppet.com/puppet5/puppet5-release-el-7.noarch.rpm

yum install puppetserver

vi /etc/sysconfig/puppetserver

 JAVA_ARGS="-Xms2g -Xmx2g -Djruby.logger.class=com.puppetlabs.jruby_utils.jruby.Slf4jLogger"

yum install -y ntpdate man perl wget
ntpdate 0.pool.ntp.org
hostnamectl set-hostname puppetmaster.local

vi /etc/puppetlabs/puppet/puppet.conf

    [main]
    certname = puppetmaster.local
    server = puppetmaster.local
    environment = production
    runinterval = 1h
    strict_variables = true
    [master]
    vardir = /opt/puppetlabs/server/data/puppetserver
    logdir = /var/log/puppetlabs/puppetserver
    rundir = /var/run/puppetlabs/puppetserver
    pidfile = /var/run/puppetlabs/puppetserver/puppetserver.pid
    codedir = /etc/puppetlabs/code

systemctl start puppetserver
iptables --flush

After first puppet agent run

List unsigned certifiates and sign it

[root@localhost ~]# puppet cert list

      "puppetagent.local" (SHA256) 35:6D:73:F8:CB:8F:DF:67:D4:81:93:11:AC:5D:0A:5B:D8:8A:69:C5:1F:87:38:BD:CF:77:26:36:7E:8A:AB:3F
    [root@localhost ~]# puppet cert sign puppetagent.local
    Signing Certificate Request for:
      "puppetagent.local" (SHA256) 35:6D:73:F8:CB:8F:DF:67:D4:81:93:11:AC:5D:0A:5B:D8:8A:69:C5:1F:87:38:BD:CF:77:26:36:7E:8A:AB:3F
    Notice: Signed certificate request for puppetagent.local
    Notice: Removing file Puppet::SSL::CertificateRequest puppetagent.local at '/etc/puppetlabs/puppet/ssl/ca/requests/puppetagent.local.pem'

   vi /etc/puppetlabs/code/environments/production/manifests/site.pp

     notify {"Agent connection is successful": }

Puppet Agent

rpm -Uvh https://yum.puppet.com/puppet5/puppet5-release-el-7.noarch.rpm

yum install puppet-agent
puppet resource service puppet ensure=running enable=true
yum install -y ntpdate man perl wget
ntpdate 0.pool.ntp.org
hostnamectl set-hostname puppetagent.local

Set puppet master in /etc/puppetlabs/puppet/puppet.conf

1st puppet run without signed certificate

[root@localhost ~]# puppet agent -tv

    Info: Creating a new SSL key for puppetagent.local
    Info: csr_attributes file loading from /etc/puppetlabs/puppet/csr_attributes.yaml
    Info: Creating a new SSL certificate request for puppetagent.local
    Info: Certificate Request fingerprint (SHA256): 35:6D:73:F8:CB:8F:DF:67:D4:81:93:11:AC:5D:0A:5B:D8:8A:69:C5:1F:87:38:BD:CF:77:26:36:7E:8A:AB:3F
    Exiting; no certificate found and waitforcert is disabled

2nd puppet run after signed certificate

[root@localhost ~]# puppet agent -tv

    Info: Using configured environment 'production'
    Info: Retrieving pluginfacts
    Info: Retrieving plugin
    Info: Retrieving locales
    Info: Caching catalog for puppetagent.local
    Info: Applying configuration version '1523748297'
    Notice: Agent connection is successful
    Notice: /Stage[main]/Main/Notify[Agent connection is successful]/message: defined 'message' as 'Agent connection is successful'
    Notice: Applied catalog in 0.03 seconds

---------------------
**Installing Ntp**

## Puppet Master

**Searching modules**

puppet module search ntp

**Install modules**

puppet module install <name>

[root@localhost ~]# puppet module install puppetlabs-ntp

    Notice: Preparing to install into /etc/puppetlabs/code/environments/production/modules ...
    Notice: Downloading from https://forgeapi.puppet.com ...
    Notice: Installing -- do not interrupt ...
    /etc/puppetlabs/code/environments/production/modules
    └─┬ puppetlabs-ntp (v7.1.1)
      └── puppetlabs-stdlib (v4.25.1)

puppet module uninstall <name>

**List modules**

[root@localhost ~]# puppet module list

    /etc/puppetlabs/code/environments/production/modules
    ├── puppetlabs-ntp (v7.1.1)
    └── puppetlabs-stdlib (v4.25.1)
    /etc/puppetlabs/code/modules (no modules installed)
    /opt/puppetlabs/puppet/modules (no modules installed)

vi /etc/puppetlabs/code/environments/production/manifests/site.pp

    notify {"Agent connection is successful": }
    include '::ntp'

**On Puppet Agent**

    [root@localhost ~]# rpm -qa | grep ntp
    ntpdate-4.2.6p5-25.el7.centos.2.x86_64
    [root@localhost ~]# puppet agent -tv
    ———
    [root@localhost ~]# rpm -qa | grep ntp
    ntp-4.2.6p5-25.el7.centos.2.x86_64
    ntpdate-4.2.6p5-25.el7.centos.2.x86_64
    ———

# Setting profiles & roles

[root@puppetmaster data]# puppet config print hiera_config
Warning: No section specified; defaulting to 'main'.
Set the config section by using the `--section` flag.
For example, `puppet config --section user print foo`.
For more information, see https://puppet.com/docs/puppet/latest/configuration.html
Resolving settings from section 'main' in environment 'production'
/etc/puppetlabs/puppet/hiera.yaml

[root@puppetmaster data]# mv /etc/puppetlabs/puppet/hiera.yaml /etc/puppetlabs/puppet/hiera.yaml.bak
[root@puppetmaster data]# ln -s /etc/puppetlabs/code/environments/production/hiera.yaml /etc/puppetlabs/puppet/hiera.yaml

    [root@puppetmaster data]# vi /etc/puppetlabs/code/environments/production/hiera.yaml
    ---
    :backends:
      - yaml
    :hierarchy:
      - "nodes/%{::fqdn}.yaml"
      - “%{::environment}/role/{::role}“
      - “%{::environment}"
      - common.yaml
    
    :yaml:
    # datadir is empty here, so hiera uses its defaults:
    # - /var/lib/hiera on *nix
    # - %CommonAppData%\PuppetLabs\hiera\var on Windows
    # When specifying a datadir, make sure the directory exists.
      :datadir: /etc/puppetlabs/code/environments/production/data

vi /etc/puppetlabs/code/environments/production/data/common.yaml

    motd::message: Hello from Hiera

[root@puppetmaster data]# tree 

    /etc/puppetlabs/code/environments/production/data
    /etc/puppetlabs/code/environments/production/data
    ├── common.yaml
    ├── nodes
    │   └── puppetagent.local.yaml
    └── cloud
        ├── aws.yaml
        └── role
            └── role_db.yaml


[root@puppetmaster data]# cat /etc/puppetlabs/code/environments/production/manifests/site.pp

    notify { hiera(motd::message): }
    notify {"Agent connection is successful": }
    class { '::ntp':
      servers => [ '0.pool.ntp.org', '2.centos.pool.ntp.org', '1.rhel.pool.ntp.org'],
    }
    group { 'wheel':
      ensure => 'present',
      gid    => '10',
    }
    user { 'skumar':
      ensure     => present,
      uid        => '507',
      gid        => '10',
      shell      => '/bin/bash',
      home       => '/home/skumar',
      managehome => true,
    }

**On Puppet Agent**

    [root@puppetagent ~]# puppet agent -t
    Info: Using configured environment 'production'
    Info: Retrieving pluginfacts
    Info: Retrieving plugin
    Info: Retrieving locales
    Info: Loading facts
    Info: Caching catalog for puppetagent.local
    Info: Applying configuration version '1523767646'
    Notice: Hello from Hiera
    Notice: /Stage[main]/Main/Notify[Hello from Hiera]/message: defined 'message' as 'Hello from Hiera'
    Notice: Agent connection is successful
    Notice: /Stage[main]/Main/Notify[Agent connection is successful]/message: defined 'message' as 'Agent connection is successful'
    Notice: Applied catalog in 0.18 seconds

**Creating Profiles**

    vi /etc/puppetlabs/code/environments/production/modules/profile/manifests/apache.pp
    class profile::apache {
      class { 'apache':
        default_vhost => true,
      }
    }
    
    vi /etc/puppetlabs/code/environments/production/modules/profile/manifests/mysql.pp
    class profile::mysql {
      class { 'mysql::server':
        root_password  => 'root',
        remove_default_accounts => false,
      }
      include mysql::client
    }
    
    vi /etc/puppetlabs/code/environments/production/modules/profile/manifests/base.pp
    class profile::base {
        include tree
        include rubygems
        include lint
        include puppet_vim
    }
**Install 3rd party modules in puppet master**

    [root@puppetmaster manifests]# puppet module install puppetlabs/apache
    Notice: Preparing to install into /etc/puppetlabs/code/environments/production/modules ...
    Notice: Downloading from https://forgeapi.puppet.com ...
    Notice: Installing -- do not interrupt ...
    /etc/puppetlabs/code/environments/production/modules
    └─┬ puppetlabs-apache (v3.1.0)
      ├── puppetlabs-concat (v4.2.1)
      └── puppetlabs-stdlib (v4.25.1)
    [root@puppetmaster manifests]# puppet module install puppetlabs/mysql
    Notice: Preparing to install into /etc/puppetlabs/code/environments/production/modules ...
    Notice: Downloading from https://forgeapi.puppet.com ...
    Notice: Installing -- do not interrupt ...
    /etc/puppetlabs/code/environments/production/modules
    └─┬ puppetlabs-mysql (v5.3.0)
      ├── puppet-staging (v3.2.0)
      ├── puppetlabs-stdlib (v4.25.1)
      └── puppetlabs-translate (v1.1.0)

**Create role** 

    vi /etc/puppetlabs/code/environments/production/modules/role/manifests/webserver.pp
    class role::webserver {
      include profile::apache
      include profile::mysql
    }

**Assign role to node**

    vi /etc/puppetlabs/code/environments/production/manifests/site.pp
    notify { hiera(motd::message): }
    notify {"Agent connection is successful": }
    class { '::ntp':
      servers => [ '0.pool.ntp.org', '2.centos.pool.ntp.org', '1.rhel.pool.ntp.org'],
    }
    group { 'wheel':
      ensure => 'present',
      gid    => '10',
    }
    user { 'skumar':
      ensure     => present,
      uid        => '507',
      gid        => '10',
      shell      => '/bin/bash',
      home       => '/home/skumar',
      managehome => true,
    }
    node 'puppetagent.local' {
      include role::webserver
    }

Run puppet agent

        [root@puppetagent ~]# puppet agent -t
        Info: Using configured environment 'production'
        Info: Retrieving pluginfacts
        Info: Retrieving plugin
        Info: Retrieving locales
        Info: Loading facts
        Info: Caching catalog for puppetagent.local
        Info: Applying configuration version '1523776437'
        Notice: This is puppetagent.local
        Notice: /Stage[main]/Main/Notify[This is puppetagent.local]/message: defined 'message' as 'This is puppetagent.local'
        Notice: Agent connection is successful
        Notice: /Stage[main]/Main/Notify[Agent connection is successful]/message: defined 'message' as 'Agent connection is successful'
        Notice: /Stage[main]/Apache/Package[httpd]/ensure: created
        Info: /Stage[main]/Apache/Package[httpd]: Scheduling refresh of Class[Apache::Service]
        Info: Computing checksum on file /etc/httpd/conf.d/README
        Info: /Stage[main]/Apache/File[/etc/httpd/conf.d/README]: Filebucketed /etc/httpd/conf.d/README to puppet with sum 20b886e8496027dcbc31ed28d404ebb1
        Notice: /Stage[main]/Apache/File[/etc/httpd/conf.d/README]/ensure: removed
        Info: Computing checksum on file /etc/httpd/conf.d/autoindex.conf
        Info: /Stage[main]/Apache/File[/etc/httpd/conf.d/autoindex.conf]: Filebucketed /etc/httpd/conf.d/autoindex.conf to puppet with sum 09726332c2fd6fc73a57fbe69fc10427
        Notice: /Stage[main]/Apache/File[/etc/httpd/conf.d/autoindex.conf]/ensure: removed
    -
    -
    -
        Info: Concat[15-default.conf]: Scheduling refresh of Class[Apache::Service]
        Info: Class[Apache::Service]: Scheduling refresh of Service[httpd]
        Notice: /Stage[main]/Apache::Service/Service[httpd]/ensure: ensure changed 'stopped' to 'running'
        Info: /Stage[main]/Apache::Service/Service[httpd]: Unscheduling refresh on Service[httpd]
        Notice: Applied catalog in 66.27 seconds
