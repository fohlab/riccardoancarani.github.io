---
layout: post
title: Hunting for Suspicious LDAP Activity with SilkETW and Yara
subtitle: Detecting Active Directory Enumeration
tags: [threat-hunting]
comments: true
---

### Intro
This is another post to document my journey of learning Threat Hunting. In today's post we're going to perform threat hunting activities with the aim of hunting for AD domain enumeration.

We're going to heavily rely on FireEye's SilkETW and we'll search for suspicious LDAP queries generated by our endpoints.

[SilkETW](https://github.com/fireeye/SilkETW) is a handy wrapper for Event Tracing for Windows (ETW) that will allow us to perform searches and ship to centralised logging platforms like ElasticSearch. ETW is defined by Microsoft as:
```
Event Tracing for Windows (ETW) is an efficient kernel-level tracing facility that lets you log kernel or application-defined events to a log file. You can consume the events in real time or from a log file and use them to debug an application or to determine where performance issues are occurring in the application.
```
SilkETW provides support for Yara rules, this means that we can define malicious patterns for events. For example, in FireEye's [SilkETW: Because Free Telemetry is … Free!](https://www.fireeye.com/blog/threat-research/2019/03/silketw-because-free-telemetry-is-free.html), [@b33f](https://twitter.com/fuzzysec?lang=en) shows us how to create Yara rules for detecting cobaltstrike's `execute-assembly` in memory. Crazy.

In this example we're going to use LDAP ETW provider, which will give us events for all the LDAP queries generated by our endpoints. I executed SilkETW with the following command:

```
SilkETW.exe -t user -pn Microsoft-Windows-LDAP-Client -ot file -p C:\windows\temp\ldap.json -l verbose -y C:\Tools\SilkETW\v8\SilkETW\yara\ -yo All
```

The notable options are:

* `-pn Microsoft-Windows-LDAP-Client`, that defined the LDAP provider for ETW;
* `-y C:\Tools\SilkETW\v8\SilkETW\yara\`, that specifies the Yara rules to match, more on those later.

Back to hunting.

As always, we start with our hypothesis:

```
Adversaries may search for sensitive groups or users with specific attributes that would allow them to perform further attacks.
```

More specifically, the things that an attacker that already obtained a foothold within our internal network may do the following:

- Identify all the members of the `Domain Admins` or other sensitive groups;
- Search for all the user with `servicePrincipalName=*` for Kerberoasting attacks;
- Search for users with Kerberos preauthentication disabled;
- Search for users with `adminCount=1`.

In order to identify those behaviours we need to create Yara rules for each one of the aforementioned cases.

### Privileged Group Enumeration and AdminCount
Members of the groups defined below are considered to be sensitive. Attackers will often try to enumerate members of those groups in the situational awareness phase.

I also decided to add the adminCount option to the rule, since it's often used to identify privileged entities. The adminCount attribute is set to every user that was part of the AdminSDHolder sensitive group and it's a good indicator for attackers to identify privileged users:

```
rule PrivilegedGroupEnumeration
{
	strings:
		$s1 = "Domain Admins" ascii wide nocase
		$s2 = "Account Operators" ascii wide nocase
		$s3 = "Backup Operators" ascii wide nocase
		$s4 = "DnsAdmin" ascii wide nocase
		$s5 = "Enterprise Admins" ascii wide nocase
		$s6 = "Group Policy Creator Owners" ascii wide nocase
		$s7 = "admincount=1"
	condition:
		any of ($s*)
}
```

### SPN Scanning

Scanning for all the SPN within a domain can be seen as one of the first reconnaissance actions to see all the available services within a domain, but also default options of tools that perform Kerberoasting will do the same:

```
rule SPNScanning {
	strings:
		$s1 = /serviceprincipalname=\*/ ascii wide nocase
		$s2 = /serviceprincipalname=\*\/\*/ ascii wide nocase
	condition:
		any of ($s*)
}
```

### Kerberos Pre-authentication Disabled
Kerberos pre-auth is a UAC option that, if disabled, would allow adversaries to perform ASREP-Roasting attacks. The following Yara rule will identify attempts to enumerate users with Kerberos pre-auth disabled:

```
rule ASREPRoasting {
	strings:
		$s1 = "(userAccountControl:1.2.840.113556.1.4.803:=4194304)" ascii wide nocase

	condition:
		any of ($s*)
}
```

### Testing Time

In order to verify the rules we just created, we need to perform some form of enumeration. In the screenshot below I used PowerView's cmdlet for detecting users with kerberos pre-auth disabled and also the `setspn.exe` utility to find all the SPN within my domain:

![](/assets/2019-10-19-hunting-for-domain-enumeration/bd7b34fc7a8f0d4253ac23b36568a8e7.png)


As it is possible to see, we're correctly detecting enumeration activities!

![](/assets/2019-10-19-hunting-for-domain-enumeration/b7957a1c49c9e242e2d09ce8df231797.png)

### Final Thoughts

* This approach may cause false positives, in my small lab it works fine but in bigger environments it may not!
* For the moment I'm just outputting to a file, it's just for testing purposes since my HELK machine takes half of the RAM of my laptop. With SilkETW it's possible to output to ElasticSearch out-of-the-box.
* A huge thanks to Ruben who developed this amazing tool!
