# Feature

DiSTAF - Distributed Systems Test Automation Framework

# Summary

DiSTAF (distaf) is a test automation framework for glusterfs or similar distributed systems. The project is written with glusterfs in mind but in theory its core APIs can be used to write test automation for any distributed system. The project already has a working codebase and is hosted at [GitHub](https://github.com/gluster/distaf).

# Owners

M S Vishwanath Bhat <vbhat@redhat.com>

# Current status

Although project has a working code base, there are very few testcases written in it. The project needs some enhancements to make it more efficient and user friendly.

The current status and architecture of distaf project can be found [here](http://redhat.slides.com/msvbhat/distaf#)

# Detailed Description of Enhancements Planned

There are two major improvements planned. One is to use a yaml config file for specifying test environment and configuration parameters. The other is improving the test runner.

## Using a yaml config file for specifying global paramaters.

Currently there is global `config.sh` file containing all env and configuration parameters. But this is kind of pseudo config file because distaf expects these variables to be present in ${shell} environment variables. The plan to is move to yaml config file. With yaml config file we get proper hierarchy of specifying test machines and its respective details. Using yaml we can also specify heirarchical volume related config parameters. Below is the sample tree structure.

        nodes:
            - hostname: gluster-server0.glusterlab.org
            - devices: /dev/vda /dev/vdb /dev/vdc
            - brick_mountpoints: /bricks/brick0 /bricks/brick1 /bricks/brick2
        
            - hostname: gluster-server1.glusterlab.org
            - devices: /dev/vda /dev/vdb /dev/vdc
            - brick_mountpoints: /bricks/brick0 /bricks/brick1 /bricks/brick2
        
        clients:
            - hostname: gluster-client0.glusterlab.org
            - mount_protocol: [ 'glusterfs', 'rdma', '/mnt/glusterfs']
        
        run_global_mode: False

This is just a sample config file and there will be some changes to the final one used. Once this is implemented, we can still improve upon this by supporting range specification and other things.


## Ability to specify configuration per test case.

We write testcases to validate something and in many cases the volume and mount types do not matter. But in some cases you will have to run them with particular mount and/or volume types. So we need a way to specify configurations per test case. We plan to write these configs as docstring of each testcase. The below example will make things clear.


    @testcase("verify_xattr_mountpoint")
    def verify_xattr_mountpoint():
        """
            runs_on_volume: ALL (distribute, distribute-replicate, disperse, tiered, replicate)
            runs_on_protocol: glusterfs
            do_not_run_on: nfs
            needs_client_packages: glusterfs-api
            min_clients_required: 1
        """
        <testcase definition>

So this way, a test case and its related configuration parameters will be together in same file. We need not have to go through different config file for a testcase's config details. These details should be specified in yaml structure as a doc string. This allows us to use the same parser we use for global config file. Also this allows us to specify lot of things like, what packages are required for each testcase and how many clients are required for each test case etc. A test case will be skipped if the required packages are not found in the specified test machine(s).

Also some times we may want to run the testcase against all possible types of volumes and mountpoints and some other times we may want to run against a particular type of volume and/or particular mount protocol. In those scenarios, we can use `run_global_mode` in the global config file. When that is set, these per test case configs are ignored and the details specified in the global yaml config files will be considered.


## Test runner or running a test case with all possible permutations and combinations

So once we have above two things ready, the next step is to run a particular test case with all possible permutations and combinations by combining the config parameters specified in global config and test case config. The config parser will first parse the global config file and create a global default dict. Now this can be updated/overwritten for each test case by reading test case specific config (testcase docstring). Now by reading `run_global_mode` value, we can finally create a dict for each test case. The test runner simply has to run a particular test case against each of these permutations and combinations of volume types and mount protocols. Now this is easier said than done. Because with our current implementation, we use unittest test runner by setting the TestClass with our test functions. But with this change, we will be having a single function (amd function address) for more than one combinations. If we set that function to unittest.TestClass with different name, we should somehow pass these config values to each test case. We should be able to pass the config details to the testcase using the decorator. But I have not done any POC on this and still do not know how good or efficient is this Idea.

But other way of doing the same is to create dict by reading default and test case config values which contains all test cases that can be run with a particular type of volume/mount combinations. With this option we will be creating a volume and running all the tests which should run against that volume/mount type. But even in this case we need a way for the test case to access its particular config values.

We can also try and write and come up with our own implementation of test runner and not use unittest testrunner. But again not sure how efficient this would be.

## Other minor improvements

1. Just using the brick mountpoint when it is specified in config file instead of brick devices
1. To support connecting to windows client for samba test automation.
1. Integrating with Jenkins to run set of gluster tests after each commit, nightly etc.
1. Injecting a test case specifier to each gluster logs before and after the test. This is to help identify the issues caused by a particular test


# Benefit to GlusterFS

A good user friendly test framework is necessary to automate the tests and find the bugs very early in the cycle.
