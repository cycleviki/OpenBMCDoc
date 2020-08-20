# SSH based SOL 
SSH based SOL function is achieve by two part, 
"obmc-console-server" daemon connect to specific uart establish channel to transfer/receive data. 
"obmc-console-client" daemon provide client to connect to channel which obmc-console-server establish, will store host console log in path */var/log/obmc-console.log*
openbmc 2.8 change to [multiple consoles structure](https://github.com/openbmc/obmc-console). ^[Though structure is expaned to multiple console , default service and related socket only support one console.
Default configuration is set to ttyVUART0. ]

## Configure 
 | Object    | Memo | Memo|
 | -------------------  | -------------------- | ------- | 
 | upstream-tty         | Device check setting | Only impact when device not set in command line|
 | local-tty            | Actual host console uart setting | |
 | local-tty-baud       | Baud rate setting, only for real UART device ||
 | lpc-address          | lpc-address settin for VUART devices ||
 | sirq                 | SIRQ setting for VUART device ||
 | logsize              | Set obmc-console.log size | Default is 16kb |
 | socket-id            | Define sock-id for multiple console server | Default socket-id is "host" |

 log size setting rule
 * No suffix : byte    e.g  logsize = 1048576        
 * "k" : kilobyte = 1024byte  , e.g logsize = 1024 k
 * "M" : megabytes = 1024Kbyte, e.g logsize = 1 M

## HS9216 implement step
HS9216 use UART3 (ttyS2) for host console uart. Use following step to enable this feature.
1. Establish server uart config  /etc/obmc-console/server.ttyS2.conf
   ```
   upstream-tty = ttyS2
   local-tty = ttyS2
   local-tty-baud = 115200
   ```
2.  command line to start obmc-console-server 
    ```
     /usr/sbin/obmc-console-server --config /etc/obmc-console/server.ttyS2.conf ttyS2
    ```
3. Remote to use ssh based SOL ^[If you're already logged into an OpenBMC machine, you can start a console session directly, using command: *obmc-console-clent* . see [host console support](https://github.com/openbmc/docs/blob/master/console.md) ]  
   ```
   ssh -p 2200 [user]@[bmc-hostname]
   ```
