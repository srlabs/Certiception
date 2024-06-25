# Certiception

**Certiception is a honeypot for Active Directory Certificate Services (ADCS)**, enabling you to catch hackers with a trap that is too juicy to ignore. 

Developed by the [SRLabs](https://srlabs.de) red team, it creates a vulnerable-looking certificate template in your ADCS environment, sets up restrictions to prevent exploitation, and supports in setting up effective alerting.

Originally released at [Troopers24](https://troopers.de/troopers24/talks/8fjh87/), Certiception comes with a strategic guide to effective deception:
* **Presentation deck**: [The Red Teamers' guide to deception](./documentation/The_Red_Teamers_Guide_To_Deception.pdf)
* **Tool release blog post**: (to be added)

**tl;dr here is how attackers see it. Looks vulnerable in the tools, attempted exploitation fails.**

![How attackers see is](./documentation/images/how_attackers_see_it.png)

## Background
As a red team, we often see insufficient detection and response for the lateral movement and privilege escalation stages of our attacks. We believe internal honeypots (aka. canaries, aka. deception tech) are an effective way for defenders to catch threats that make it through initial defenses.

Internal honeypots are intentional traps for attackers placed in your network. They look vulnerable but trigger an alert on exploitation. Here's why we think deception has great potential: 
* **Low effort and cost** – they can be set up with existing tools like a SIEM 
* **Low noise** – they can be designed to only trigger on malicious use and have low false positives
* **High relevance alerts** – a triggered honeypot indicates a significant threat, catching cases that really should be investigated

At the same time we faced many ineffective deception setups during our past engagements. To support defenders in building effective honeypots, we provide our extensive deception strategy and how to get started in our [slide deck](./documentation/The_Red_Teamers_Guide_To_Deception.pdf).

However, after collecting our favorite honeyport for Active Directory, we realized there was still a technology left for which we always wanted to have a honeypot. Active Directory Certificate Services (ADCS) is the perfect location for a trap:
1. It can be accessed by all domain users --> easy to discover for attackers
2. Vulnerabilities usually enable full domain compromise --> attractive to exploit
3. Exploitation tooling is widely known --> low barrier for exploitation
4. It's often vulnerable in real-world configurations --> authentic and not suspicious
5. Under-monitored in many networks --> attackers feel safe to exploit

 This is why we built Certiception.
## Concept
Certiception sets up a new CA in your environment and configures an [ESC1](https://posts.specterops.io/certified-pre-owned-d95910965cd2) honeypot on it.

It is implemented as an Ansible playbook calling multiple roles. Overall, the following steps are executed:
* Set up a new CA, add a “vulnerable” ESC1 template and enable it only on the new CA
* Install and configure the [TameMyCerts](https://github.com/Sleepw4lker/TameMyCerts) policy module to prevent issuance if certificate signing requests contain a SAN
* Enable the extended audit log on the CA to include template names in the event logs
* Print a SIGMA rule to set up alerting in your SIEM
* Set up continuous checks with [Certify](https://github.com/GhostPack/Certify) to catch any other CA enabling the vulnerable template

Parameters like the CA or template name can be customized to make it harder to recognize them as a honeypot. Here is an overview how the setup works:

![Architecture and flow](./documentation/images/architecture_and_flow.png)

Support for other types of ESC vulnerabilities and adding honey templates to existing CAs is planned for the future.
### Alerting
Currently we use built-in Windows events from the CA and events of the TameMyCerts policy module. To get the built-in CA events with the required information, Certiception enables the extended audit log on the honey CA server.

Here are some of the key events you can use.

| Event source         | Event ID                                | Alert                                                                    |
| -------------------- | --------------------------------------- | ------------------------------------------------------------------------ |
| Windows Security Log | 4886 – Certificate enrollment requested | **MEDIUM** - Honey template was used                                     | 
| Windows Security Log | 4887 – Certificate issued               | Not used, 4886 has more coverage                                         |
| Windows Security Log | 4888 – Certificate request denied       | Not used, TameMyCerts 6 is more precise when issuance fails non-malicious|
| TameMyCerts          | 6 – CSR denied due to policy violation  | **CRITICAL** - attempted exploitation via SAN |

Certiception prints SIGMA rules to use for the two different alerts. You need to ensure the respective event IDs are onboarded into your SIEM and then setup alerting with the SIGMA rules.

In the future we might switch to events generated by [TameMyCerts](https://github.com/Sleepw4lker/TameMyCerts).
## Usage
Setting up your own ADCS honeypot.
### Prerequisites
* **Domain-joined Windows server** to install the CA on
* **Machine with Ansible and WinRM connectivity to server** to clone and execute this repository
* **Local admin account for the server** to install the CA and exploitation protections
* **Enterprise Admin account** to register the new CA
* **Basic Domain account** without any privileges for executing the continuous Certify checks
### Installing Certiception
1. Configure Ansible to access your CA host via `inventory.yml`
2. Choose unique parameters for your honeypot in `host_vars/honeypotCA.yml`
3. Create EDR exception for future Certify location (used for monitoring if any non-honey CAs enable the vulnerable template, not required yet as Certify-based monitoring it not yet published)
4. Execute Certiception via Ansible
```bash
ansible-playbook -i inventory.yml playbooks/certiception.yml
```
4. Onboard the servers' event logs to your SIEM and configure alerts with the printed SIGMA rules
5. Verify and manually test your setup

## Safety and security considerations
This tooling is provided without any warranty or guarantees. It combines existing software, automating installation and configuration. You are responsible for all installation and configuration steps performed by Certiception.

If you use this tool, we strongly recommend you read the source code to understand what you configure and verify your setup after installation.

Additionally, we recommend to consider the following:
1. At the time of release the tool was not scrutinized by the community yet - expect that its safety will still be improved over time.
2. An ADCS honeypot only make sense if the PKI team takes up the ownership for it. When performing configuration changes, the implications on the honeypot need to be considered. E.g. when migrating the honey CA to a new server without also migrating the policy module, the honey template becomes exploitable.
3. Besides the honey template, your honey CA should be secured, hardened and managed like any other ADCS CA in your network
4. The CA set up by this tool is a plain CA with the CA certificate stored on disk and no HSM
5. Certiception sets up basic checking for real vulnerable templates utilizing [Certify](https://github.com/GhostPack/Certify) on the CA server. For production environments we would instead recommend to have a separate machine in the network perform these continuous checks. They should not only be based on "find" commands for template identification but also attempt exploitation of the honey template (SIEM allow-listed) to catch configuration changes that make it exploitable on the honey CA.

## Future work
* Support placing honey templates on existing CAs
* Implement support for more ESC misconfigurations (e.g. ESC3 and ESC8)
* Investigate additional hardening options against exploitation
* Use lower priv. accounts instead of enterprise admin
* Let the community scrutinize safety of the honeypot
* Add less suspicious error message on denied CSRs
* Investigate and mitigate ways of fingerprinting
* Strengthen continuous monitoring intended to catch and mitigate insecure configurations 

## Acknowledgements
* [Uwe Gradenegger](https://github.com/Sleepw4lker) for his great [PKI and ADCS blog](https://www.gradenegger.eu/en/) and as the developer of the [TameMyCerts](https://github.com/Sleepw4lker/TameMyCerts) policy module
* [@GoateePFE](https://twitter.com/goateepfe) with his [ADCSTemplate](https://github.com/GoateePFE/ADCSTemplate) scripts we use for creating up the honey template
* [@harmj0y](https://twitter.com/harmj0y) and [@tifkin_](https://twitter.com/tifkin_) for their [ADCS research](https://posts.specterops.io/certified-pre-owned-d95910965cd2) and corresponding [Certify tool](https://github.com/GhostPack/Certify)
* [@ly4k_](https://twitter.com/ly4k_) who found [ESC9 and ESC10](https://research.ifcr.dk/certipy-4-0-esc9-esc10-bloodhound-gui-new-authentication-and-request-methods-and-more-7237d88061f7) (and develops [Certipy](https://github.com/ly4k/Certipy))
* [@sploutchy](https://infosec.exchange/@sploutchy) for [ESC11](https://blog.compass-security.com/2022/11/relaying-to-ad-certificate-services-over-rpc/)
* [Hans-Joachim Knobloch](https://www.linkedin.com/in/hans-joachim-knobloch-165527267/) for [ESC12](https://pkiblog.knobloch.info/esc12-shell-access-to-adcs-ca-with-yubihsm)
* [@Jonas_B_K](https://twitter.com/jonas_b_k) and [@_wald0](https://x.com/_wald0) auditing [ADCS with Bloodhound](https://posts.specterops.io/adcs-attack-paths-in-bloodhound-part-1-799f3d3b03cf) and [ESC13](https://posts.specterops.io/adcs-esc13-abuse-technique-fda4272fbd53)
* [@PyroTek3](https://twitter.com/PyroTek3) for his [previous work](https://www.hub.trimarcsecurity.com/post/the-art-of-the-honeypot-account-making-the-unusual-look-normal) on Active Directory honeypots
* [@gentilkiwi](https://twitter.com/gentilkiwi) for inspiration like [this one](https://twitter.com/gentilkiwi/status/706515035422629890) 
* All the friends and colleagues who provided input and feedback for our talk and development

Peace out :)

![Footer](./documentation/images/footer.png)