The Script:
    description: the bash script that perform uni/bi CommsMatrix tests and relys on the conf file as input to determine the behavior
    logic :
       - localhost execution :
            - Read The Conf file and do validations
            - generate the scripts
            - execute Listeners and Testers Scripts via ssh&at on listeners
            - create a backgtound task on localhost  to:
                - check every 10 minutes if testers finished execution via checking the flagfile at  /tmp/CommsMatrix-<ConfFileName>-<BlockName>-<Date>-done
                - if flagfile exist the task will copy the reports to the localhost on ~<user>/CommsMatrix/<Date>/<ConfFileName>/<BlockName>/Reports/<TesterIP>
            - script exit
        - listerners execution:
            - make sure nmap-ncat is installed
            - if firewalld running will be stopped during Listenduration
            - if target ports already open will not be touched
            - if targeted ports closed will be listening during the ListenDuration
            - logs will be saved ~<user>/CommsMatrix/<Date>/<ConfFileName>/Logs/Listening/
        - Testers execution:
            - make sure nmap-ncat is installed
            - will not start scan a port range before making sure the range endport is listening
            - reports will be saved on ~<user>/CommsMatrix/<Date>/<ConfFileName>/Reports
            - logs will be saved ~<user>/CommsMatrix/<Date>/<ConfFileName>/Logs/Listening/
            - after the tester script finished execution will create the flag file at ~<user>/CommsMatrix/<Date>/<ConfFileName>/<BlockName>/Reports/<TesterIP>
    Dependencies:
        execution-user : user who run the script must be either authenticate with public-key or with saved passwrod to the remote machine
        remote-user    : have to be sudo with no passwd
        packages : 
            - nmap-ncat  will be installed on the remote system if not installed
            - at should be installed on the remote system
        services:
            - firewalld service will be stopped if running
            - atd should be running 
    Files Generated:
        description: files get generated by the script for troubleshooting/reporting/logging 
        path:   ${HOME}/CommsMatrixData/<ConfFileName>/<Date>/
        location:
           localhost:
                configuration-file:
                    description: the conf file used when the script executed
                    path: <ConfFileName>.conf
                scripts:
                    description: the scripts generated by the CommsMatrix Script to be executed on remote listeners and testers
                    path: 
                        ListenerScripts: Scripts/<BlockName>/Listeners/<ListenerIP>-<tcp/udp>.sh
                        TesterScripts:  Scripts/<BlockName>/Testers/<TesterIP>/<TesterIP>-<ListenerIP>-<tcp/udp>.sh                    
                reports:
                    description:  the reports gathered from remote testers
                    path:               Reports/<BlockName>/<TesterIP>/<tcp/udp>/<TesterIP>-<ListenerIP>-<tcp/udp>.txt
                logs:
                    description:  the logs gathered from remote machines describe the execution progess
                    path: 
                        ListenersLogs:  Logs/<BlockName>/Listeners/<ListenerIP>/<ListenerIP>-<tcp/udp>.log
                        TesterLogs:     Logs/<BlockName>/Testers/<TesterIP>/<TesterIP>-<ListenerIP>-<tcp/udp>.log                                  
           TesterIP:                
                reports:
                    description:  the reports on testers
                    path: Reports/<tcp/udp>/<TesterIP>-<ListenerIP>.txt
                logs:
                    description:  the logs on TesterIP describe the execution progess
                    path: Logs/Testing/<TesterIP>-<ListenerIP>-<tcp/udp>.log
                flags:
                    description:   empty files created as a flag for the completion of all scan on specific block listeners ips 
                    path       :     Reports/done
            ListenerIP:
                logs:
                    description:  the logs on Listener describe the execution progess
                    path: Logs/Listening/<ListenerIP>-<tcp/udp>.log
                other:
                    description:   listen to udp port performed by python script
                    path       :   /tmp/UDP-Listener.py

The Configuration File:
    description: hold multiple [block] to determine the script behavior
    blocks:
        types:
           Default:   hold default attributes "Mode,User/s,ListenDuration" other attributes are now allowed
           <Others>:   hold specific attributes for the block including "Mode,User/s,ListenDuration,IPs,Ports," , block name is one string [a-zA-Z]
        attributes:
            mode independant : can be used within [default] or any other [block]
                Mode: avaliable values
                    uni :   one  way communication matrix test
                    bi  :   two way communication matrix test
                User:          the remtote user
                UDPPorts:      mixed ranges/indivudal ports comma separated
                TCPPorts:      mixed ranges/indivudal ports comma separated
                ListentDurationInMinutes: port listen duration applied on tcp and udp ports on listeners
            mode dependant :
                            IPs: the remote ips , bi mode only 
                            TestersIPs:      the remote testerip, uni mode only
                            ListenersIPs:    the remote listenerip , uni mode only
    Notes:
        - block name is one string [a-zA-Z]
        - hashed and empty lines in Configuration File ignored
        - spaces not allowd in Configuration File Name and Blocks Names
        - IPs/TesterIPs/ListenerIPs:
                - hole the value within the {}. 
                - the targeted ips can be mixed ranges/indivudal ips space or newline separated
                - the ip values are validated
                - IPS attribute is not allowd in uni mode
                - TesterIPs/ListenersIPS attribute are not allowd in bi mode
    example: |
        [Default]
        #default Mode
        Mode: uni
        #default remote user
        User: root
        #Default ListenDuration
        ListentDurationInMinutes: 10

        [Block1]

        ListentDurationInMinutes: 150
        User: ansible
        PortsUDP:     1000-1100,200,2000-2100
        PortsTCP: 1-100,110,300-400
        TestersIPs:
        {
        1.1.1.1
        2.2.2.2-2.2.2.10
        #3.3.3.3
        }
        ListenersIPs:
        {
        3.3.3.3
        4.4.4.4
        }


        [Block2]
        Mode:bi
        PortsUDP:     1000-1100,200,2000-2100
        IPs:
        {
        7.7.7.7
        8.8.8.8
        }
        