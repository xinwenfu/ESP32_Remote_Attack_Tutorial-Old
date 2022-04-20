# ESP32 Remote Attack Tutorial

## Prerequisites

Install ESP-IDF version 4.4

Get VM

Set VM to Bridged Mode, then restart VM if needed.

Connect ESP32 to VM

Install mosquitto

```
sudo apt update && sudo apt install mosquitto mosquitto-clients
```

Confirm mosquitto is installed and running:

```
systemctl status mosquitto
```

```
netstat -tlpn | grep 1883
```

Finally, get the IP address of your VM:

```
ifconfig
```

This will list the network interfaces on your machine. You are interested in the Ethernet interface, typically named something like "eth0" or "ens33". The IP address of the interface is the address next to "inet".

## Build, Flash, and Test Projet

### Configure Project

Run menuconfig

```
idf.py menuconfig
```

Go to Example Configuration. This asks you for the URL of the MQTT broker to connect to. We want to connect to the mosquitto broker running on our VM, so set the value to your IP address. The structure of the field is mqtt://<IP address>. For example, in my case, it is mqtt://192.168.1.186.

Now go to Example Connection Configuration. This asks you to supply your WiFi connection information. Enter your SSID and password. There is no need to change any other settings. You can now exit and save the configuration.

### Build, Flash, and Monitor

Now build, flash, and monitor:

```
idf.py build flash monitor
```

If successful, you will see the ESP32 connect to your WiFi access point and subscribe to 2 MQTT topics: "/topic/qos0" and "/topic/qos1".

### Test Project

Without closing the ESP32 monitor, open a new termianl. In that new terminal, run the following command, which uses mosquitto_pub (a software for publishing MQTT messages) to publish a simple "Hello" message to the topic "/topic/qos0":

```
mosquitto_pub -h 0.0.0.0 -t /topic/qos0 -m "Hello"
```

## Inject Malicious String into Program
  
Now we will see how to inject data (in the form of a string) into a program through a well-known attack called **Format String Attack**. A format string attack can occur when certain functions do not format user input correctly. For example, `printf("%s, input)` is safe, while `printf(input)` is vulnerable to the attack. More details on format string attacks (for x86 architecture) are available here: https://web.ecs.syr.edu/~wedu/Teaching/cis643/LectureNotes_New/Format_String.pdf
  
This MQTT application is vulnerable to the format string attack. See line 98 of the main file: https://github.com/PBearson/ESP32_Remote_Attack_Tutorial/blob/86b7400b642051719e6b2161f4da7cff6dcaf6b0/main/app_main.c#L98

  To perform the attack, there are two shell scripts in this project. The first script, `inject_payload.sh`, will inject the string "You are hacked!" into memory address 0x3ff80101. The address range 0x3ff80000 - 0x3ff81fff is reserved for RTC FAST memory, which is memory that persists through the ESP32's deep sleep mode. Since our application does not use deep sleep, this memory region is empty. The second script, `print_payload.sh`, will print the injected string.

  To execute the first script, run the following command in your second terminal:
  
  ```
  sh inject_payload.sh
  ```
  
  This command uses mosquitto_pub to publish the following payload (in binary) to the ESP32:
  
  ```
 0101f83f0201f83f0301f83f0401f83f0501f83f0601f83f0701f83f0801
f83f2001f83f2001f83f0b01f83f0c01f83f0d01f83f0e01f83f0f01f83f
1001f83f2532357825362468686e2532327825372468686e253236327825
382468686e253137317825392468686e25333231782531302468686e2531
37782531312468686e25323433782531322468686e253138377825313324
68686e25333238782531342468686e25323439782531352468686e253235
38782531362468686e2538782531372468686e2532353078253138246868
6e25323535782531392468686e25313839782532302468686e2532323378
2532312468686e0a
  ```
  
  You will see the ESP32 appear to print random memory addresses, like so:
  
  ![image](https://user-images.githubusercontent.com/11084018/164243528-33a45a36-f475-48de-b3e4-a814d7a23289.png)
  
  In reality, this is a side effect of carefully selecting which bytes to inject into memory. It will be explained later.
  
  To execute the second script, run the following command:
  
  ```
  sh print_payload.sh
  ```
  
  This command will use mosquitto_pub to publish the following payload (in binary) to the ESP32:
  
  ```
  0101f83f200d25362473200d0a
  ```
  
  In the ESP32 monitor, you should see the injected string print out to the console:
  
  ![image](https://user-images.githubusercontent.com/11084018/164245035-4f3e36c7-c9dc-4a2f-bf62-eeefe79d9b87.png)
  
  If you do not see the string, then simply run the script again. The RTC FAST region is accessed exclusively by the ESP32's PRO_CPU core, while the normal application is executed by the APP_CPU core. This means that executing `printf` and reading the string happen on different CPU cores, and sometimes the `printf` function will return before the string has been retrieved from memory.

  As a side note, if you are interested in displaying precisely what data is sent to the ESP32, you can convert either payload back to its binary form with the following commands:
  
  ```
  echo <payload> | xxd -p -r
  ```
  
  ![image](https://user-images.githubusercontent.com/11084018/164247098-3ddc4f81-67eb-419e-8ae8-5eb262e5e5b9.png)
  
  ![image](https://user-images.githubusercontent.com/11084018/164247413-8305880f-aa21-44e3-87d0-e59b6b85fdc4.png)


  
  ### Explanation of `inject_payload.sh`
  
  The `inject_payload.sh` script essentially has the following structure:
  
  ```
  <address 1>, i.e., 0x3ff80101
  <address 2>, i.e., 0x3ff80102
  ...
  <address 16>, i.e., 0x3ff80110
  <format string to inject 'Y' into address 1>
  <format string to inject 'o' into address 2>
  <format string to inject 'u' into address 3>
   ...
  <format string to inject '!' into address 15>
  <format string to inject '\0' into address 16>
  ```
  
  By placing the addresses at the beginning of the buffer, the format string attack will actually place those addresses onto the stack. The `<format string to inject...>` lines will reference the stack at different offsets, so by preemptively placing the addresses onto the stack, we can write to them. For example, `%6$hhn` will write data to the first address on the stack, `%7$hhn` will write to the second address, and so forth. There are methods to carefully control what data is written, which will be explained shortly. Now the rest of this section will explain the `inject_payload.sh` script line by line.

  Line 3 sets the mosquitto_pub command prefix, which prepares our script to send a message to the "/topic/qos0" topic:
  
  https://github.com/PBearson/ESP32_Remote_Attack_Tutorial/blob/605baf042e11948bfcca83b64565830069b863e3/inject_payload.sh#L3
  
  Lines 5 - 20 prepare the memory addresses we will need for the attack. We create addresses 0x3ff80101 - 0x3ff80110. Each address will correspond to a byte from our malicious string, including the null terminator. The address bytes are reversed since the ESP32 uses little endian addressing.
  
  https://github.com/PBearson/ESP32_Remote_Attack_Tutorial/blob/605baf042e11948bfcca83b64565830069b863e3/inject_payload.sh#L5-L20
  
  Line 23 combines all addresses together into a single variable:
  
  https://github.com/PBearson/ESP32_Remote_Attack_Tutorial/blob/605baf042e11948bfcca83b64565830069b863e3/inject_payload.sh#L23
  
  Lines 25 through 40 write the bytes into the addresses. To control what data is written to each address, we use the "%Nx" parameter (where N is an integer) to increase the number of characters written by `printf` by N bytes. Then when the "%M$hhn" parameter (where M is another integer) is executed, the value written to the address is the number of characters written so far. Further explanation is given below.
    
 For the first address, 0x3ff80101, we want to write the character 'Y'. This has hex value 0x59, or decimal value 89. Therefore, we need to first get `printf` to write 89 characters. Since we use the "%hhn" parameter, which specifies a one-byte write width, it is also okay to write any number of characters where the least significant byte is 0x59, e.g., 0x159, 0x259, 0x359, etc., and it will only write 0x59. Since our buffer begins with 16 addresses, each of which is 4 bytes long, `printf` will have already printed 64 characters. To write the value 89, we can use the "%25x" parameter to print 25 more characers. This also has the effect of printing out register A11, which is traditionally the second argument passed to `printf`. Then we use the "%6\$hhn" parameter to write the value 89 to address 0x3ff80101. Note that we need to escape the $ symbol, since this symbol is typically used in shell scripts to reference variables. Also note that using the "%M$hhn" format string does not contribute to the number of bytes written by `printf`.
    
 For the second address, 0x3ff80102, we want to write the character 'o'. This has hex value 0x6f, or hex value 111. Therefore, we need to print out 22 more characters. Using the parameter "%22x%7\$hhn", we print out 22 more characters (111 total) and write this value to address 0x3ff80102. It also has the effect of printing out register A12, which is traditionally the third argument passed to `printf`.
    
For the third address, 0x3ff80103, we want to write the character 'u'. This has hex value 0x75, or 117. At first glance, it may seem that we need to print out exactly 6 more characters. However, if we try to use the parameter "%6x", this will print out the contents of register A13, which may be a 32-bit value. For example, it may contain an address such as `3f4036d8`. Although the address itself is only 4 bytes, `printf` will print it as a string, which is 8 bytes. Therefore, the "%6x" parameter could potentially print out an additional 8 bytes instead of the desired 6 bytes. To avoid this issue, instead of trying to print 0x75, we can print 0x175, or 373. Thus, we can print an additional 262 bytes. The full parameter is "%262x%8\$hhn", which increases the number of characters printed to 373 and writes the least significant byte (0x75) to address 0x3ff80103.
    
 For addresses 0x3ff80104 through 0x3ff80110, we repeat this process, using the "%Nx" paramater to increase the number of bytes printed by `printf` and writing that value to each corresponding address. It will also print out the contents of register A14 (fifth `printf` argument), A15 (sixth `printf` argument), the stack value pointed to by the stack pointer (seventh argument), SP + 4 (eighth argument), SP + 8, and so forth. This is why we see some of the addresses we injected onto the stack when we observe the console output.
  
  https://github.com/PBearson/ESP32_Remote_Attack_Tutorial/blob/605baf042e11948bfcca83b64565830069b863e3/inject_payload.sh#L25-L40
    
Line 43 combines lines 25 through 40 into a single variable:
    
https://github.com/PBearson/ESP32_Remote_Attack_Tutorial/blob/605baf042e11948bfcca83b64565830069b863e3/inject_payload.sh#L43
    
Line 46 crafts the final payload by combining the addresses with the format string parameters:
    
https://github.com/PBearson/ESP32_Remote_Attack_Tutorial/blob/605baf042e11948bfcca83b64565830069b863e3/inject_payload.sh#L46
    
Optionally, lines 49 through 52 print a hexdump of the payload if the user supplies an argument to the script, for example, `sh inject_payload.sh 1`:
     
https://github.com/PBearson/ESP32_Remote_Attack_Tutorial/blob/605baf042e11948bfcca83b64565830069b863e3/inject_payload.sh#L49-L52
    
Finally, line 54 combines the mosquitto_pub command with the payload. The payload becomes the message that is published to the "/topic/qos0" topic.
    
https://github.com/PBearson/ESP32_Remote_Attack_Tutorial/blob/605baf042e11948bfcca83b64565830069b863e3/inject_payload.sh#L54
