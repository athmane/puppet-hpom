Puppet Operations Manager Module
=================================

HP Operations Manager is a monitoring tool by Hewlett Packard. This repository
tries to provide puppet types and providers to ease the management of HP Operations
Manager nodes.

Installation
------------

Generally you need the following

* Your HP Operations Manager server should run on Unix (OMU) or Linux (OML), Windows (OMW) is
  currently not supported. The server types have only been tested on Linux and with OML 9. Older
  versions /might/ work out of the box or with slight changes (different paths, command line
  parameters etc).
* For the om\_node type to work you need the HP OVO API installed. You should be able to get
  the API for free if you already own a HP OML license. You also have to install the perl
  bindings for the OVO API. Installation instructions can be found in the links section.
  If you have correctly installed the OVO API you should be able to run the `opcnodehelper.pl`
  script inside the opcnode provider directory.
* To be able to use the custom types in this repository make sure that you copy the lib directory
  in a valid module directory. You also have to active `pluginsync` in your agent's config.

New facts
---------
* `opcagtversion`: The version of the agent that is running on the node

New functions
-------------
(currently none)

New custom types
----------------

### om\_node type

The om\_node type can be used on your management server to describe a node with
its label, machine type and communication type. You can also assign
layoutgroups and nodegroups to your node. It is currently not possible to
describe direct policy assignments or policy group assignments.

The provider for that type uses the `opcnodehelper.pl` script to read current
node information from the OVO database, so you have to install the OVO API
and the perl bindings to be able to use the om\_node type. The perl script
does only do read operations. If a om\_node resource is out sync the provider
uses the `opcnode` executable to perfom the change. Because the command does not have
command line arguments for all properties a few state changes are not
supported and have to be performed manually (e.g. changing a node's label).

Example usage:

    om_node { 'testnode01.example.com':
      ensure             => present,
      label              => 'testnode01',
      ipaddress          => '10.0.0.1',
      network_type       => 'NETWORK_IP',
      machine_type       => 'MACH_BBC_LX26RPM_X64',
      communication_type => 'COMM_BBC',
      node_type          => 'CONTROLLED',
      dynamic_ip         => no,
      layout_groups      => [
        'Linux/Webserver',
        '/CustomHierarchy/physical/rz10',
      ],
      node_groups        => [
        'linux',
        'webserver',
        'basic',
      ],
    }

because the type makes use of some default values the above example can also be written as

    om_node { 'testnode01.example.com':
      ensure             => present,
      label              => 'testnode01',
      ipaddress          => '10.0.0.1',
      machine_type       => 'MACH_BBC_LX26RPM_X64',
      dynamic_ip         => no,
      layout_groups      => [
        'Linux/Webserver',
        '/CustomHierarchy/physical/rz10',
      ],
      node_groups        => [
        'linux',
        'webserver',
        'basic',
      ],
    }

You get the best benefit if you use storeconfig and every puppet node exports a `om_node` resource.
On you OML master you now just have to collect all OM nodes:

    Om_node<<| |>>

### om\_heartbeat type

The om\_heartbeat type configures different heartbeats for a specific node.
This type is probably useless to you because it configures an xml file that is used by an
external (an closed source) program. I just wanted to share it anyways because it may work
as an example on how to parse xml files and configure entries with puppet

The heartbeat configuration file has the following basic structure:

    <OVOHeartBeat>
      <Hosts>
        <Host name="foo.domain.tld"/>
      </Hosts>
      <HeartBeats>
        <HeartBeat name="hearbeat01"/>
          <Rule host="foo.domain.tld"/>
        </HeartBeat>
      </HeartBeats>
    </OVOHeartBeat>

The puppet type can now be used to describe the heartbeat configuration. The above example can be expressed like this

    om_heartbeat { 'foo.domain.tld':
      ensure     => present,
      heartbeats => [ 'heartbeat01' ],
    }

You have to have nokogiri installed to be able to use the nokogiri provider (the only one so far).

### om\_fsmon

If you use the HP System Infrastructure SPI to monitor your filesystems you can define custom
thresholds on a per node basis. The custom thresholds are stored as agent config variables.

The om\_fsmon type allows you to set the `SpaceUtilWarningThreshold`, `SpaceUtilMinorThreshold`,
`SpaceUtilMajorThreshold` and `SpaceUtilCriticalThreshold` settings by treating mountpoint you
want to monitor as a resource.

Sample usage:

    om_fsmon { '/mnt/a':
      warning  => 10,
      minor    => 20,
      major    => 30,
      critical => 40,
    }
    om_fsmon { '/mnt/b':
      warning  => '50',
      minor    => absent, # remove any special threshold
      major    => absent, # remove any special threshold
      critical => absent, # remove any special threshold
    )
    om_fsmon { '/mnt/c':
      warning  => 101, # never raise a warning
      minor    => 101, # never raise a minor message
      major    => 101, # never raise a major message
      critical => 101, # never raise a critical message
    )

If you apply a catalog with the resources above, puppet will finally set the following
configuration options:

    SpaceUtilWarningThreshold=80,/mnt/a=10,/mnt/b=50,/mnt/c=101
    SpaceUtilMinorThreshold=85,/mnt/a=20,/mnt/c=101
    SpaceUtilMajorThreshold=90,/mnt/a=30,/mnt/c=101
    SpaceUtilCriticalThreshold=95,/mnt/a=40,/mnt/c=101

The provider uses the `ovconfget` and `ovconfchg` binaries to change the
agent's config

Links
-----
* about the OVO API: http://www.blue-elephant-systems.com/content/view/294/314/
* installation instructions: http://www.blue-elephant-systems.com/component/option,com_docman/task,doc_download/gid,118/Itemid,141/
