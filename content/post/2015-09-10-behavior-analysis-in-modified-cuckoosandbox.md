---
title: "Behavior Analysis in Modified Cuckoo Sandbox "
date: 2015-09-10
categories:
  - Malware Analysis
tags:
  - Cuckoo Sandbox
toc: true  
---

The goal of this project is to build an automatic dynamic malware analysis system. We leverage the advantages of Cuckoo Sandbox as a dynamic behavior collection platform. Cuckoo Sandbox is developed over 4 years in the open source community . It is flexible to customize due to a well structured modular architecture.

# Installation

Without the vm clone, there are three parts in setting up Cuckoo Sandbox. [Install the packages in the host machine](http://docs.cuckoosandbox.org/en/latest/installation/host/requirements/), [install cuckoo](http://docs.cuckoosandbox.org/en/latest/installation/host/installation/), [install virtual machine](http://docs.cuckoosandbox.org/en/latest/installation/guest/). With the virtual machine clone, we have all these components setup and ready to use. 

To use the virtual machine clone, create a new Ubuntu 64 bit virtual machine in virtualbox, in the step to create the hard drive, choose the option "Use an existing virtual hard drive file" and select the ```.vdi``` file from the directory of the vm clone to import. Finish the default configuration of the vm. You'll have a virtual machine with Cuckoo Sandbox installed and ready to use.

The username and password for Ubuntu host:
```
root password is 123456

username  rui
password: 123456
```
The Windows guest don't have a password

# Configure Cuckoo

Before running Cuckoo. We need to make sure Cuckoo is configured correctly by modifying the configuration files in the ```conf``` directory . We only need the following configuration files.

	1. cuckoo.conf
	2. virtualbox.conf
	3. processing.conf
	4. reporting.conf 
	5. auxiliary.conf

We have them fully configured in our copy. In case you need any change, please refer to the detailed comments in the individual configuration file.

# How to run Cuckoo sandbox
There are two steps:

1. To run the python script ```cuckoo.py``` in the application root directory. This script start up the Cuckoo Sandbox. ```cuckoo.py --debug``` will output detailed debug information upon startup. See the [official manual](http://docs.cuckoosandbox.org/en/latest/usage/start/) for other commandline options. 

	>Note: Before the first time running Cockoo, start the virtualbox and Windows 7 guest machine manually. Otherwise, Cuckoo will incur error during startup, this is due to the reason that it cannot bind the virtual network card we installed in virtualbox without boot the guest. See the appendix for more details about the network setup between the guest and host.

2. To submit the sample to Cuckoo by running the python script ```utils/submit.py /path/to/sample```. This will start the analysis of the submitted sample. There are also other options you can provide during submission, such as ```--max```, which will specify the maximum samples will add for analysis. See the [official manual](http://docs.cuckoosandbox.org/en/latest/usage/submit/) for other options.

# Quick start commands:
```bash
#1. Start Cuckoo
rui@rui-VirtualBox:~/cuckoo$./cuckoo.py 
# with debug level of logs dump to stdout.
rui@rui-VirtualBox:~/cuckoo$./cuckoo.py --debug

#2. Submit sample to Cuckoo
rui@rui-VirtualBox:~/cuckoo$./utils/submit.py /path/to/the/sample
# example of batch analysis of samples from a directory, 10 samples 
rui@rui-VirtualBox:~/cuckoo$./utils/submit.py --max 10 /path/to/the/sample/directory

#3. Check the statistics
rui@rui-VirtualBox:~/cuckoo$./utils/stats.py 

#4. Remove the analysis generated data (db, storage, and log) from Cuckoo directory
rui@rui-VirtualBox:~/cuckoo$./cuckoo.py --clean

#5. To combine the multiple json analysis results generated in vtfeature. the purpose of
#   this is to generate dataset as input to the existing machine learning algorithms.
rui@rui-VirtualBox:~/cuckoo$./utils/joinjson.py

#6. To enable and disable screenshot 
modify the code in the line cuckoo/analyzer/windows/modules/auxiliary/screenshots.py:27
	"self.do_run = Ture" to enable
	"self.do_run = False" to disable 
```

# Cuckoo reports and there locations
You can configure the generation of the report in the file ```processing.conf```
and ```reporting.conf```. We are interested in two kinds of reports, one resides in the directory ```storage/rawfeature/``` and another resides in ```storage/vtfeature/```. The rawfeature includes complete behavioral profiles while the vtfeature includes tailored profiles to match the information obtained from virustotal.com. These two kinds of reports for each analyzed sample are json files with its SHA256 as file name. 

There is a script ```utils/joinjson.py``` that join all the generated json files in the ```storage/vtfeature/```. The joined file is put in the ```storage/dataset/```. The file name is the string ```dataset-```, followed by a decimal number and the postfix ```.json```. For example, the file name ```dataset-10.json``` is tell us how many sample's profile in ```storage/vtfeature/``` is joined.

## Where to find the analysis results?
Following is the directory structure of the analysis output, see the comments by the side of the file or directory below.
```text
storage
├── analyses
│   ├── 1
│   │   ├── analysis.log	#Log from analysis guest
│   │   ├── binary 
│   │   ├── dump.pcap
│   │   ├── files			#malware created files uploaded here 
│   │   │   ├── 1260809960
│   │   │   │   └── WebBrowser_embedded.exe
...
│   │   │   └── 9504481298
│   │   │       └── Failed.htm
│   │   ├── logs			#logs of the hooked APIs 
│   │   │   └── 1364.bson
│   │   ├── reports			#report generated by vanila cuckoo
│   │   │   ├── report.html
│   │   │   └── report.json
│   │   └── shots			#screenshot of the analysis machine
│   ├── 2
│   │   ├── analysis.log
... #similar to structure as above
│   │   └── shots
│   └── latest -> /home/rui/cuckoo/storage/analyses/2
├── binaries
│   ├── 0008e49c2d25161a78e8062ee59d6d03fdad59fd014906d2fdc0092988dc413b
│   └── 33120cda1dd67cc13e73e83db6d7a33eea3ef2474653e90e5e153b6b473b3f14
├── dataset					#generated by manually running ./utils/joinjson.py
│   ├── dataset-2.json		#number after the dash sign is total json joined
├── rawfeature				#json files contain comprehensive behavior reports 
│   ├── 0008e49c2d25161a78e8062ee59d6d03fdad59fd014906d2fdc0092988dc413b.json
│   └── 33120cda1dd67cc13e73e83db6d7a33eea3ef2474653e90e5e153b6b473b3f14.json
└── vtfeature				#json files contain tailored reports match virustotal
    ├── 0008e49c2d25161a78e8062ee59d6d03fdad59fd014906d2fdc0092988dc413b.json
    └── 33120cda1dd67cc13e73e83db6d7a33eea3ef2474653e90e5e153b6b473b3f14.json
```

# The added processing module and report module

Although reports generated by Cuckoo Sandbox is both informative and configurable, 
the contents are too detailed to read either by human being or by software, because 
it is lack of behavioral categorization and order of importance of each. In our project, 
we require complete and structed behavior information in order to build the learning and
detection components, which is mainly developed by my teammate Chi.

## Processing module ```comprehensive.py```

This module generate the complete behavior profile from the logged binary json file. 
It can be found under directory: ```cuckoo/modules/processing/```. It adds much 
more semantic interpretation of Windows APIs in extracting acurate dynamic behavioral 
information. The behavior information we generated can be divided in to 9 groups as following, there are 31 operations:

| Groups | System Objects |  Operations  |
|--------|----------------|--------------|
|   1    | File        |<ul><li>open</li><li>read</li><li>delete</li><li>modify</li><li>move</li></ul>|
|   2    | Registry    |<ul><li>open</li><li>create</li><li>delete</li><li>enum</li><li>modify</li><li>move</li><li>query</li><li>close</li></ul>|
|   3    | Service     | <ul><li>openscmanager</li><li>open</li><li>start</li><li>create</li><li>delete</li><li>modify</li></ul>|
|   4    | Mutex       |<ul><li>open</li><li>create</li></ul>|
|   5    | Processes   |<ul><li>started</li><li>terminated</li></ul>|
|   6    | Runtime DLLs|<ul><li>.dlls files</li></ul>|
|   7    | Network     |<ul><li>TCP</li><li>UDP</li><li>DNS</li><li>HTTP</li></ul>| 
|   8    | Hooks	   |<ul><li>hooks</li><li>unhooks</li></ul> |
|   9    | Windows     |<ul><li>searchedwindows</li></ul> |

## Reporting module ```comprehensivedump.py```
This module dump the json object into a file in the directory of ```cuckoo/vtfeature```. The name is the sha256 value.  Folowing is a template the created by the ```comprehensivedump.py``` reporting module.
```javascript
{
  "metadata": {
    "name":"",
    "type":"PE32 executable (GUI) Intel 80386, for MS Windows",
    "size":"",
    "sha256": "",
    "md5":"",
    "machine": {
      "analysisid": 55, 
      "duration": 171, 
      "guest": "Win7-32", 
      "manager": "VirtualBox", 
      "shutdown": "2015-08-04 01:05:59", 
      "started": "2015-08-04 01:03:08", 
      "version": "1.3-dev"
    },
    "virustotal": {
        "date": "2015-07-16 18:22:44", 
        "permalink": "https://www.virustotal.com/file/0a69bfbdefa4ae6595fb09c2de70948f0818fd18f0998419917fb8f9efd162b4/analysis/1437070964/", 
        "positives": 45, 
        "total": 56
        "scans": {
            "ALYac": {
                "detected": true, 
                "result": "Generic.Malware.IN!p2p!dld.F635BCD0", 
                "update": "20150716", 
                "version": "1.0.1.4"
            }, 
			... 
        }, 
     }
  },
  "file": {
    "open":[],
    "read":[],
    "create":[],
    "delete":[],
    "modify":[],
    "move":[]
  },
  "registry": {
    "open":[],
    "read":[],
    "create":[],
    "delete":[],
    "modify":[]
  },
  "service": {
    "open":[],
    "create":[],
    "delete":[],
    "enum":[],
    "modify":[],
    "query":[],
    "close":[]
  },
  "mutex": {
    "open":[],
    "create":[]
  },
  "network": {
    "domain":[],
    "tcp":[],
    "udp":[],
    "http":[]
  },
  "runtimeDLL": [],
  "process": [],
  "hook": [],
  "unhook":[],
  "searchedwindows":[]
}
```

## Reporting module ```vtfeature.py```

This is a seperate reporting module that designed to generate behavioral profile
that match exactly to the profile collected from virus total by Chi. It generate
a string that could be used as malware classification label. The ```vtfeature.py```
file is compact and I will present it below.
```python
# Copyright (C) 2010-2015 Cuckoo Foundation.
# This file is part of Cuckoo Sandbox - http://www.cuckoosandbox.org
# See the file 'docs/LICENSE' for copying permission.

import os
import json
import codecs
import operator
import re

from lib.cuckoo.common.abstracts import Report
from lib.cuckoo.common.exceptions import CuckooReportError
from collections import OrderedDict

class VtFeatureDump(Report):
    """Saves analysis results in JSON format."""

    def run(self, results):
        """Writes report.
        @param results: Cuckoo results dict.
        @raise CuckooReportError: if fails to write report.
        """
        indent = self.options.get("indent", 4)
        encoding = self.options.get("encoding", "utf-8")

        vtfeature = OrderedDict()
        details = OrderedDict()
        categories = "" 

        compreobj = results["comprehensive"]

        wordcount = {}
        
        for k, v in compreobj["virustotal"]["scans"].iteritems():
            if v["result"]:
                words = re.split('\.|/| |:|!', v["result"])
                for word in words:
                    w = word.lower()
                    if w not in wordcount:
                        wordcount[w] = 1 
                    else:
                        wordcount[w] += 1

        sorted_wc = sorted(wordcount.iteritems(), key=operator.itemgetter(1))
        sorted_wc.reverse()
        i = 10
        for kk, vv in sorted_wc:
            if i:
                categories += "({}={})".format(kk, vv)
                i -= 1

        details = {
                "id":compreobj["sha256"],
                "categories": categories,
                "permurl":compreobj["virustotal"]["permalink"],
                "scorestr":"{0}/{1}".format(compreobj["virustotal"]["positives"],compreobj["virustotal"]["total"]),
                }

        vtfeature = { 
                "Additional details": details,
                "Read files": compreobj["file"]["read"],
                "TCP connections":compreobj["TCP connections"],
                "Hooking activity":compreobj["hooks"],
                "DNS requests":compreobj["DNS requests"],
                "HTTP requests":compreobj["HTTP requests"],
                "Opened services":compreobj["service"]["open"],
                "Written files": compreobj["file"]["modify"],
                "Deleted files": compreobj["file"]["delete"],
                "Created mutexes":compreobj["mutex"]["create"],
                "Searched windows":compreobj["searchedwindow"],
                "Opened files":compreobj["file"]["open"],
                "Replaced files":compreobj["file"]["create"],
                "Created processes":compreobj["processes"],
                "Opened mutexes":compreobj["mutex"]["open"],
                "UDP communications":compreobj["UDP connections"],
                "Runtime DLLs":compreobj["runtimedll"]
                } 

        try:
            reportname = compreobj["name"]+".json"
            path = os.path.join(self.vtfeature_path, reportname)
            with codecs.open(path, "w", "utf-8") as report:
                json.dump(vtfeature, report, sort_keys=True,
                          indent=int(indent), encoding=encoding)
        except (UnicodeError, TypeError, IOError) as e:
            raise CuckooReportError("Failed to generate JSON report: %s" % e)
```

The behavior report it generated is as following:
```javascript
{
    "Additional details": {
        "categories": "(trojan=22)(win32=15)(temr=10)(gen=8)(zusy=8)(138428=7)(variant=7)(backdoor=3)(w32=2)(genericr-dmc=2)", 
        "id": "3a3a3adbf4b6cc962d603c3f453f546dd0afdf9a5a11cb7161821523c0d733e9", 
        "permurl": "https://www.virustotal.com/file/3a3a3adbf4b6cc962d603c3f453f546dd0afdf9a5a11cb7161821523c0d733e9/analysis/1437087697/", 
        "scorestr": "42/56"
    }, 
    "Created mutexes": [], 
    "Created processes": [], 
    "DNS requests": [], 
    "Deleted files": [], 
    "HTTP requests": [], 
    "Hooking activity": [], 
    "Opened files": [], 
    "Opened mutexes": [], 
    "Opened services": [], 
    "Read files": [], 
    "Replaced files": [], 
    "Runtime DLLs": [], 
    "Searched windows": [], 
    "TCP connections": [], 
    "UDP communications": [], 
    "Written files": []
}
```

# Code modification statistics
This is the code modification statistics from original Cuckoo source:
```
rui@rui-VirtualBox:~/cuckoo$ git diff --stat b91ebffe 430f6434
 .gitignore                                        |   13 +
 analyzer/windows/modules/auxiliary/screenshots.py |    2 +-
 conf/cuckoo.conf                                  |    7 +-
 conf/processing.conf                              |   10 +-
 conf/reporting.conf                               |   10 +
 lib/cuckoo/common/abstracts.py                    |    3 +
 lib/cuckoo/core/scheduler.py                      |    3 +-
 lib/cuckoo/core/startup.py                        |    5 +-
 modules/processing/comprehensive.py               | 1354 +++++++++++++++++++++
 modules/reporting/comprehensivedump.py            |   32 +
 modules/reporting/mongodb.py                      |    3 +-
 modules/reporting/vtfeaturedump.py                |   86 ++
 utils/api.py                                      |   66 +-
 utils/joinjson.py                                 |   31 +
 utils/setup.sh                                    |    2 +-
 15 files changed, 1586 insertions(+), 41 deletions(-)
```

# Appendix: Network Setup between Host and Guest 
> Note the term "vitualbox" mentioned in this section is refer to the guest Virtualbox(the inner virtualbox in our setting), becasue our installed "Cuckoo host" itself is a virtual machine. Our setting is a nested virtualization environment. 

After installing the Windows guest virtualbox, we do the following steps to configure the network. This is important, otherwise, Cuckoo will not run.  

1. Turn off windows auto-update and firewall, enable some other Windows (guest) settings by run the following command
```	
reg add "hklm\software\Microsoft\Windows NT\CurrentVersion\WinLogon" /v AutoAdminLogon /d 1 /t REG_SZ /f
reg add "hklm\system\CurrentControlSet\Control\TerminalServer" /v AllowRemoteRPC /d 0x01 /t REG_DWORD /f
reg add "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System" /v LocalAccountTokenFilterPolicy /d 0x01 /t REG_DWORD /f
```
make sure you didn't set up username and password to login. The above setting make sure  you are enable auto-logon (Allows for the agent to start upon reboot) and Remote RPC (Allows for Cuckoo to reboot the sandbox using RPC) 

2. Setting up the network in VirtualBox
	1. We should use two network adapter, the first one is host-only network adapter (Create one such adapter (vboxnet0) for the vm from the virtualbox menu ```File->Preference->Network```), this adapter will enable host guest communication and RPC mechanism Cuckoo requires. After creating the host-only network adapter(i.e. vboxnet0), select it in the vm's own setting, using default settings.[192.168.56/24]
	2. Configure the host-only network adapter as following(all default values). This is important for Cuckoo to run correctly. The default value is match the default configuration in the virtualbox.conf file.
		![create](https://lh3.googleusercontent.com/-scIosvMflog/VYxAs24zHKI/AAAAAAAACdo/LRAeGTRNMzw/s800/create-the-host-only-adapter.png)
		![create](https://lh3.googleusercontent.com/-Nw5PKZqBt4I/VYxAcVxAoiI/AAAAAAAACdo/5N2I10ucMdE/s800/create-host-only-adapter-dhcp-server-setting.png)
	3. The other adapter we should use is NAT. To set this, just enable the second adapter from the virtual machine's network configuration, and select the NAT in the dropdown menu.

3. Using the `vboxmanage` command line tool to create snapshot of vms.
	```
	vboxmanage snapshot "[VM Name]" take "[Snapshot Name]" --pause
	vboxmanage controlvm "[VM Name]" poweroff
	vboxmanage snapshot "[VM Name]" restorecurrent
	```
4. Configure
	1. cuckoo.conf
		* Make sure the following configuration is set up correctly:
			```
			machinary = virtualbox
			[resultserver] 
			ip = 192.168.56.1
			```
	2. virtualbox.conf
		* Make sure the following is set up correctly:
			```
			machine = Win7-32
			platform = windows
			ip = 192.168.56.101
			snapshot = cuckoo-test
			```

# Reference
1. [Cuckoo Sandbox Blackhat USA 2013 White Paper](https://docs.google.com/document/d/1uSt8A3V-fBs4kv6dhNmaNdX3LX9moC3VTH70baL0XAo/edit#)
2. [Cuckoo Sandbox Blackhat USA 2013 Slides](http://downloads.cuckoosandbox.org/slides/blackhat.pdf)
3. [recon2013 Slide by Jurriaan](http://recon.cx/2013/slides/recon2013-Jurriaan%20Bremer-Haow%20do%20I%20sandbox.pdf)
