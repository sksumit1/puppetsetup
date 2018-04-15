Puppet Master

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
Info: Caching certificate for puppetagent.local
Info: Caching certificate for puppetagent.local
Info: Using configured environment 'production'
Info: Retrieving pluginfacts
Info: Retrieving plugin
Info: Retrieving locales
Info: Caching catalog for puppetagent.local
Info: Applying configuration version '1523746532'
Notice: Applied catalog in 0.03 seconds
