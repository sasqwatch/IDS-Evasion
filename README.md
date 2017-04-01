# **IDS-Evasion**

## Index

- [Index](#index)

- [Attacks Snort could identify](#attacks-snort-could-identify)
  - [ElasticSearch Dynamic Script Arbitrary Java Execution (CVE-2014-3120)](#elasticsearch-dynamic-script-arbitrary-java-execution-cve-2014-3120)
  - [FTP Authentication Scanner (CVE-1999-0502)](#ftp-authentication-scanner-cve-1999-0502)
  - [OpenSSH MaxAuthTries Limit Bypass Vulnerability (CVE-2015-5600)](#openssh-maxauthtries-limit-bypass-vulnerability-cve-2015-5600)
  - [Jenkins-CI Script-Console Java Execution](#jenkins-ci-script-console-java-execution)
  - [Apache Struts REST Plugin With Dynamic Method Invocation Remote Code Execution (CVE-2016-3087)](#apache-struts-rest-plugin-with-dynamic-method-invocation-remote-code-execution-cve-2016-3087)
  - [ManageEngine Desktop Central 9 FileUploadServlet ConnectionId Vulnerability (CVE-2015-8249)](#manageengine-desktop-central-9-fileuploadservlet-connectionid-vulnerability-cve-2015-8249)

- [Attacks Snort could not identify](#attacks-snort-could-not-identify)
  - [Jenkins-CI Script-Console Java Execution](#jenkins-ci-script-console-java-execution-1)
  - [MS15-034 HTTP Protocol Stack Request Handling Denial-of-Service (CVE-2015-1635)](#ms15-034-http-protocol-stack-request-handling-denial-of-service-cve-2015-1635)
  - [ElasticSearch Dynamic Script Arbitrary Java Execution (CVE-2014-3120)](#elasticsearch-dynamic-script-arbitrary-java-execution-cve-2014-3120-1)

- [Is it easier to fix the application than to detect attacks?](#so-is-it-easier-to-fix-the-application-than-to-detect-attacks)

- [References](#references)

---

## **Attacks Snort could identify**
### ElasticSearch Dynamic Script Arbitrary Java Execution ([CVE-2014-3120](https://www.cve.mitre.org/cgi-bin/cvename.cgi?name=2014-3120)):
Most of snort rules are *commented out* [by default](https://www.snort.org/faq/why-are-rules-commented-out-by-default). So we need to search for them either by product name (i.e. in our case "ElasticSearch") or even better by CVE (i.e. in our case "CVE-2014-3120") and *uncomment* them (i.e. remove the "#" character from the beginning of the line), in order to enable them. We can use the `Select-String` command (the "grep-like" command in PowerShell) for that purpose:

![powershell_search_cve](screenshots/ElasticSearch/powershell_search_cve.png)

Running snort:

![powershell_running_snort](screenshots/ElasticSearch/powershell_running_snort.png)

We'll use "exploit/multi/elasticsearch/script_mvel_rce" module to exploit this vulnerability (you can find this module using `search ElasticSearch` or `search CVE-2014-3120` inside Metasploit).

Setting module options, checking whether if the target is vulnerable or not and finally running the module:

![metasploit_set_and_exploit](screenshots/ElasticSearch/metasploit_set_and_exploit.png)

Checking Snort:

![powershell_snort_detecting_elastic_rce](screenshots/ElasticSearch/powershell_snort_detecting_elastic_rce.png)

As we see, snort identified the attack successfully.


### FTP Authentication Scanner ([CVE-1999-0502](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-1999-0502)):
We'll use "auxiliary/scanner/ftp/ftp_login" module to do a dictionary attack on the Target FTP service.
The search didn't show a result on "CVE-1999-0502" which is associated with the module (check the module info), so I couldn't find a rule to uncomment in those tons of rules (like trying to find a needle in a haystack!), so I'll write the rule myself.

![powershell_no_result](screenshots/FTP_login/powershell_no_result.png)

First, we need to know what is the response of the target FTP service to unsuccessful logins in order to write a suitable PCRE (i.e. **P**erl **C**ompatible **R**egular **E**xpression) to match the unsuccessful logins:

![ftp_reconnaissance](screenshots/FTP_login/ftp_reconnaissance.png)

I'll write two tailored rules, one for regular unsuccessful logins: 

![unsuccess_login_rule](screenshots/FTP_login/unsuccess_login_rule.png)

.. & the other for detecting brute-forcing attempt:

![bruteforce_rule](screenshots/FTP_login/bruteforce_rule.png)

After running the module we'll find only one successful trial:

![metasploit_set_and_exploit](screenshots/FTP_login/metasploit_set_and_exploit.png)

Given that we have 20 usernames in *PASS_FILE*:

![awk_no_of_lines](screenshots/FTP_login/awk_no_of_lines.png)

.. Then we will expect an alert from *rule #1* for every unsuccessful login attempt (i.e.19 alerts) and an alert from *rule #2* for every 5 unsuccessful login attempts (i.e. 3 alerts because INTEGER_DIVISION(19/5)=3) occur within the determined threshold (i.e. 5 minutes):

![powershell_snort_detect_ftp](screenshots/FTP_login/powershell_snort_detect_ftp.png)

Snort generates the same alerts if we used Hydra:

![hydra](screenshots/FTP_login/hydra.png)

![powershell_snort_detect_hydra](screenshots/FTP_login/powershell_snort_detect_hydra.png)

Notice that if we tried the same method with ssh_login module (i.e. write a rule to detect unsuccessful SSH login attempts), it will not work. The reason is that FTP sends the packets in *plain text* while the packets sent by SSH are *encrypted* (except for first few packets until the two parties agreed to the key as *Diffie–Hellman* key exchange algorithm).
Using Wireshark to examine packets sent from the target FTP service (using `ip.dst==192.168.1.14/32 and ip.src==192.168.1.143/32 and tcp.port eq 21` *filter* to narrow our search):

![ftp](screenshots/FTP_login/ftp.png)
![ftp_331](screenshots/FTP_login/ftp_331.png)
![ftp_530](screenshots/FTP_login/ftp_530.png)

Note that all the packets are in *clear text*.

Using Wireshark to examine packets sent from The target SSH service (using `ip.dst==192.168.1.14/32 and ip.src==192.168.1.143/32 and tcp.port eq 22` *filter* to narrow our search):

![ssh](screenshots/FTP_login/ssh.png)
![ssh_D.H.](screenshots/FTP_login/ssh_D.H.png)
![ssh_enc](screenshots/FTP_login/ssh_enc.png)

So unlike FTP, we can't write a PCRE to match the packets sent from SSH in the same way we did before. But we'll see how to detect it in the next section.


### OpenSSH MaxAuthTries Limit Bypass Vulnerability ([CVE-2015-5600](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2015-5600)):
We'll use "auxiliary/scanner/ssh/ssh_login" module to exploit this vulnerability. First we search for our rule:

![powershell_search_cve](screenshots/SSH_login/powershell_search_cve.png)

Now the important piece in our rule is `content:"SSH-"; depth:4;`.. here "*content*" keyword makes snort look for "SSH-" string among the packets.. the "*depth*" keyword is a modifier to the "*content*".. simply, it tells snort how far into a packet it should search for the "SSH-" string.. in our case we are looking for "SSH-" within the first 4 bytes of the packet:

![wireshark_ssh_packet](screenshots/SSH_login/wireshark_ssh_packet.png)

Setting module options & Run it.. it found one successful trial:

![metasploit_set&run](screenshots/SSH_login/metasploit_set&run.png)

But when we check snort there is no alert. After examining the issue, I found that snort configuration file (i.e. snort.conf) didn’t include the file which contains our rule (i.e. indicator-scan.rules). So we've to include it by putting `include $RULE_PATH\indicator-scan.rules` in snort configuration file.

Now if we run the module again, Snort can detect the attack successfully:

![snort_detect_ssh_bf](screenshots/SSH_login/snort_detect_ssh_bf.png)


### Jenkins-CI Script-Console Java Execution:
We'll use "exploit/multi/http/jenkins_script_console" module to exploit this vulnerability. This module uses the Jenkins-CI Groovy script console to execute OS commands using Java. 

![metasploit_set](screenshots/Jenkins/metasploit_set.png)

Searching for the suitable rule among snort rules.. the vulnerability has no *CVE identifier* so we may search by product name (i.e. Jenkins) or we may try searching by module name (i.e. Jenkins_script_console):

![powershell_search_cve](screenshots/Jenkins/powershell_search_cve.png)

.. and it looks like we found our desired rule.

After running the module we gained a meterpreter successfully:

![gain_meterpreter](screenshots/Jenkins/gain_meterpreter.png)

Again when we check Snort there is no alert.. so first we'll go back and check our rule.. we'll change the content value to "POST /script" (in  [jenkins_script_console.rb](https://github.com/rapid7/metasploit-framework/blob/master/modules/exploits/multi/http/jenkins_script_console.rb) module there is `"#{@uri.path}script"` that concatenate the URI & "script".. the uri in the rule defined as "/jenkins" which is not right in our case).
Then we'll go to snort.conf file and add our port (i.e. 8484) to *HTTP_PORTS* variable.
Now if we run the module again, Snort generates the alerts successfully:

![snort_detect_jenkins](screenshots/Jenkins/snort_detect_jenkins.png)


### Apache Struts REST Plugin With Dynamic Method Invocation Remote Code Execution ([CVE-2016-3087](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2016-3087)):
First we search for our rule:

![powershell_search](screenshots/Struts/powershell_search.png)

found two rules, we'll go and enable them (i.e. uncomment them).

To exploit this vulnerability, we'll use "exploit/multi/http/struts_dmi_rest_exec" module.
Setting module options & checking if our target is vulnerable:

![ms_set_options](screenshots/Struts/ms_set_options.png)

Run the module:

![ms_exploit](screenshots/Struts/ms_exploit.png)

Unfortunately, when we check Snort, there is no alert generated. Before trying to solve this issue, let's inspect the sent packets with Wireshark (using `ip.src==192.168.1.14/32 and ip.dst==192.168.1.143/32 and tcp.port eq 8282` filter to narrow our search):

![wireshark_inspect_memAcc_&_new](screenshots/Struts/wireshark_inspect_memAcc_&_new.png)

I managed to solve the issue in two different methods:

First method, I wrote a new rule to match the vulnerability based on the packet inspection I did in the previous step:

![wireshark_rule_side2side](screenshots/Struts/wireshark_rule_side2side.png)

Note that we are matching the raw (i.e. unnormalized) URI:

![normalized_and_raw_uri](screenshots/Struts/normalized_and_raw_uri.png)

.. so we are trying to match "%23" and "%20". For that we have "http_raw_uri" and "I" modifiers to restrict the search to the unnormalized URI.

Now if we tried to run the exploit again, snort will detect it successfully:

![snort_detect_custom_rule](screenshots/Struts/snort_detect_custom_rule.png)

Second method, I made the two default rule works.. I found that if a rule is dealing with HTTP normalization, then I have to put its port (i.e. 8282) in http_inspect_server preprocessor that resides in Snort configuration file (i.e. snort.conf).

(The "http_inspect" preprocesor operates on "http_inspect_server" port list. The "http_inspect" preprocessor only inspects traffic on the ports within the http_inspect_server port list. So if a rule has "http_*" keyword(s) (e.g. http_uri), then its port has to be in the http_inspect_server port list.)

![http_inspect_add_port](screenshots/Struts/http_inspect_add_port.png)

let's take a look on the two default rules:

![default_rules](screenshots/Struts/default_rules.png)

Unlike the previous custom rule, here we are trying to match the normalized URI (e.g. |23| is the Hexadecimal  for "#" ASCII character).. So here we have "http_uri" and "U" modifiers to restrict the search to the normalized URI.

And If we tried to run the exploit again, Snort will detect it successfully:

![snort_detect_default_rules](screenshots/Struts/snort_detect_default_rules.png)


### ManageEngine Desktop Central 9 FileUploadServlet ConnectionId Vulnerability ([CVE-2015-8249](http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2015-8249)):
We'll start with searching for the rules:

![powershell_search](screenshots/ManageEngine/powershell_search.png)

.. uncomment these three rules and add 8020 to http_inspect_server port list in snort.conf file.

Then, we'll use "exploit/windows/http/manageengine_connectionid_write" metasploit module to exploit this vulnerability:

![ms_set_check_run](screenshots/ManageEngine/ms_set_check_run.png)

.. and Snort detect the attack:

![snort_detect_ManageEngine](screenshots/ManageEngine/snort_detect_ManageEngine.png)

---

## **Attacks Snort could not identify**
### Jenkins-CI Script-Console Java Execution:
Yes, the same vulnerability again!.. but this time we won't get caught by snort. We'll use *Obfuscation* (i.e. manipulating data so that the IDS signature will not match the packet that is passed but the receiving device with still interpret it properly).

We know that these two commands are identical:

![identical_rev_trav](screenshots/Jenkins_2/identical_rev_trav.png)

As we see, we move down into the directory tree and then uses the `../` to get back to the original location.
If the command is long enough, the IDS may, in the interest of saving CPU cycles, not process the entire string and miss the exploit code at the end. We'll take advantage of this concept of "*relative directories*" to evade Snort.  

What is snort looking for?  Let's take a look at our rule: 

![snort_rule](screenshots/Jenkins_2/snort_rule.png)

We could manipulate `"POST /script"` to something like `"POST /down/downAgain/../../script"`.

![wireshark_normal](screenshots/Jenkins_2/wireshark_normal.png)

We'll use the same settings as we did before except for one thing:

![fake_rel_dir](screenshots/Jenkins_2/fake_rel_dir.png)

This option will insert fake relative directories into the URI.. let's run the module and do *packet inspection*:

![gain_meterpreter](screenshots/Jenkins_2/gain_meterpreter.png)

![wireshark_fake](screenshots/Jenkins_2/wireshark_fake.png)

As we see, the number of the move ups (i.e. `../`) have to be *equal* to the move downs (e.g. `/Directory`).. the final expression will evaluate to the same URI as before (i.e. `/script`) but the IDS will not notice the attack at all:

![snort_not_detecting](screenshots/Jenkins_2/snort_not_detecting.png)


### ElasticSearch Dynamic Script Arbitrary Java Execution ([CVE-2014-3120](https://www.cve.mitre.org/cgi-bin/cvename.cgi?name=2014-3120)):
The same vulnerability identified by snort before, but we'll execute the attack in a bit different way to evade snort. 

As usual, Let's take a look at our rule:

![rule](screenshots/ElasticSearch_2/rule.png)

what the default rule misses here is the "nocase" modifier to `content:"POST "`. you might argue that there is two "nocase" modifiers.. but those belongs to `content:"script"` & `content:"System."`. Here I'll quote from Snort documentaion:

> The nocase keyword allows the rule writer to specify that the Snort should look for the specific pattern, ignoring case. nocase modifies the previous content keyword in the rule.

So the "nocase" modifier affect the previous content keyword only. We'll take advantage of this mistake.. use "exploit/multi/elasticsearch/script_mvel_rce" metasploit module with the same settings -as we did before- execpt for this option:

![method_random_case](screenshots/ElasticSearch_2/method_random_case.png)

.. this option will use random casing for the HTTP method (e.g. gEt or GEt).

without this option our packet normally looks like this:

![normal_packet](screenshots/ElasticSearch_2/normal_packet.png)

with this option being set, our packet may looks like these three random permutations:

![3_packet_permutations_example](screenshots/ElasticSearch_2/3_packet_permutations_example.png)

Now if we run the module, Snort has a probability of 0.0625 to caught us (as our method is the four-charachters method POST, and the available permutaions is 2^4).

I tried to execute the module and luckily snort didn't complain about it:

![snort_no_det](screenshots/ElasticSearch_2/snort_no_det.png)

### MS15-034 HTTP Protocol Stack Request Handling Denial-of-Service ([CVE-2015-1635](http://www.cve.mitre.org/cgi-bin/cvename.cgi?name=cve-2015-1635)):
We'll use "auxiliary/dos/http/ms15_034_ulonglongadd" module to cause a *denial-of-service* to our target.

First, search for the rule and go enable it:

![powershell_search_cve](screenshots/DoS/powershell_search_cve.png)

We will run Snort a bit differently this time.. using *fast* alert mode instead of *console* alert mode.. so that in case of the target machine became down won't miss the generated alerts (if there are such alerts):

![powershell_run_snort](screenshots/DoS/powershell_run_snort.png)

Back to metasploit, set the *RHOST*.. we can check if the target is vulnerable in two ways.. first using the `check` command inside the module:

![module_set&check](screenshots/DoS/module_set&check.png)

.. second way usning *telnet*, and wait to see if the server responds with "Requested Header Range Not Satisfiable", then the target may be vulnerable:

![check_telnet](screenshots/DoS/check_telnet.png)

Run the module:

![ms_run](screenshots/DoS/ms_run.png)

.. well, metsploit says us that execution completed. Now let's go see what happens to our target machine from these five consecutive screenshots:

![ms3_before](screenshots/DoS/ms3_before.png)
![ms3_after_1](screenshots/DoS/ms3_after_1.png)
![ms3_after_2](screenshots/DoS/ms3_after_2.png)
![ms3_after_3](screenshots/DoS/ms3_after_3.png)
![ms3_after_4](screenshots/DoS/ms3_after_4.png)

After the target machine is up again after reboot, we'll go to "*\Snort\log*" directory and check the "*alert.ids*" file. I found nothing so we could guess that Snort didn't caught the attack.

I tried to add new rules: 

![additional_2_rules](screenshots/DoS/additional_2_rules.png)

.. but Snort still didn't detect the attack.
Although I managed to trigger the rules with *wget*(i.e. `wget --header "Range: bytes=1-18446744073709551615" http://192.168.1.143`) , *curl*(i.e. `curl -v 192.168.1.143/ -H "Host: test" -H "Range: bytes=0-18446744073709551615"`) and *telnet*(like how we check if the target is vulnerable before), Snort didn't identify the metasploit attack.

---

## **So is it easier to fix the application than to detect attacks?**

Fixing is better because "pattern matching" is awful, you've to be precise to avoid false positives and sometimes being precise means that the attackers can evade your rules. 
Also you can't be sure that IDS will detect all the novel attacks as the attackers may execute their attacks in a devious ways.. including, but not limited to obfuscation, flooding, encryption and fragmentation.

There are other cases when you deploy a product that doesn't belong to you. So if a vulnerability announced, sometimes product provider can't instantly create a patch for this vulnerability or guide you with workarounds to mitigate its consequences. In that case, Incident Response Engineer has to write an attack signature for this attack. 

Another issue to consider is Zero-Day exploits -as almost every organization is at risk for zero-day exploits-, here the vulnerability is undisclosed -you don't know what you don't know!- so we are somehow compelled to use IDSs. Protecting against these kind of exploits may require mixing signature-based technique with statistical-based and behavior-based techniques.

Not properly configured IDS leads to a lot of false-positives which make security team not taking the alerts seriously. Also even if IDS detects an attack, the odds are that these (i.e. packets) contain spoofed IP addresses and somehow reduce the possibility of finding the actual attackers.

So I think that Fixing for sure is better when possible.. it prevents you from the burdens of IDSs.

---

## *References*:

###### [C. Del Carlo, "Intrusion Detection Evasion: How an attacker get past the burglar alarm", SANS Institute InfoSec Reading Room, 2003](https://www.sans.org/reading-room/whitepapers/detection/intrusion-detection-evasion-attackers-burglar-alarm-1284).

###### [D. Hammarberg, “The Best Defenses against Zero-day Exploits for Various-sized Organizations”, SANS Institute InfoSec Reading Room, 2014](https://www.sans.org/reading-room/whitepapers/bestprac/defenses-zero-day-exploits-various-sized-organizations-35562).