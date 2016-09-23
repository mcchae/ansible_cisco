# ansible_cisco
cisco communication using ansible
Ansible 최신 버전 (2016.9.23 현재)인 2.2로 테스트한 Cisco Catalyst 테스트 결과

## 준비작업

### ansible 설치
[Ansible 소개](https://github.com/mcchae/ansible_doc_ko/blob/master/Asible%20소개.md) 에 나와 있는 것처럼

```sh
$ sudo pip install git+git://github.com/ansible/ansible.git@devel
```

VirtualEnv 환경에서도 설치하여 보았으나 모듈을 복사하고 수행하면서 디폴트 파이썬을 찾기 때문에 시스템 디폴트 파이썬에 설치합니다.

필요한 paramiko 등과 같은 모듈은 모두 같이 설치됩니다.

#### Ubuntu 14.04

/usr/bin/python (2.7.x)에 위와 같은 방법으로 잘 설치됩니다.

#### macOS Sierra

`homebrew` 로 설치한 파이썬이 있어 **/usr/local/bin/python** 을 실행시키고 있었으나 시스템 디폴트 파이썬인 **/usr/bin/python** 을 찾기 때문에 PATH를 /usr/bin 우선으로 두어 시스템 디폴트 파이썬에 작업함

> *주의:*
> 맥의 디폴트 파이썬인 경우 pip 가 설치되어 있지 않습니다. 

```sh
$ curl -O https://bootstrap.pypa.io/get-pip.py
$ sudo python get-pip.py
```
위의 방법으로 pip 를 설치하고 마찬가지 방법으로

```sh
$ sudo pip install git+git://github.com/ansible/ansible.git@devel
```
설치하면 됩니다.


## 작업 환경

ansible 디폴트 환경설정 파일은 /etc/ansible/ansible.cfg 를 이용하지만 macOS 등에서는 디폴트로 /etc/ansible이 생성되지 않기 때문에, 또 특정 프로젝트에 맞게 작업하는 데는 해당 폴더에 맞는 작업을 필요하므로 본 작업에 따른 작업은 해당 폴더에 국한하여 작업된다고 가정합니다.

## 설정 및 Inventory 준비

### 설정

해당 폴더의 ansible.cfg 파일에 다음과 같이

```ini
[defaults]
transport=paramiko
hostfile = ./myhosts
host_key_checking=False
timeout = 5
nocows = 1
```

와 같이 지정해 줍니다.

> **노트:** `nocows=1` 은 mac 에서 돌릴 때 디폴트로 결과가 cowsay 로 나오는 것을 방지하지 위함입니다.

### Inventory (관리 대상 목록)

myhosts 라는 파일에

```ini
[cisco-devices]
R1
#R2
```

> **노트:** 위에서 테스트로 R1 만 돌리고 R2는 막아놓았습니다.

R1 이라는 것이 /etc/hosts 파일에 존재해도 되지만 이것이 호스트 변수처럼 지정하게 할 수도 있는데 바로 `host_vars` 폴더에 해당 이름 (예 *R1*) 으로 파일을 만드는 방법입니다.

예를 들어 `host_vars/R1` 이라는 파일에는

```yml
---
ansible_ssh_host: 10.31.1.254
ansible_ssh_user: admin
ansible_ssh_pass: future_01
```

> **노트:** 사용자는 ssh 접속시 사용한 사용자ID 이고 암호는 ssh 암호입니다.

> *참고:* 시스코 카탈리스트 스위치 설정은 [3560G 스위치 설정](http://mcchae.egloos.com/5202403) 을 참고하였습니다.

## 애드-혹 테스트

ansible 플레이북이 아니라 ansible 만으로 모듈만 테스트 하는 것을 애드-혹 이라고 합니다.

### raw 모듈 테스트

우선 ansible의 모든 모듈은 agent-less 방식으로 동작하는 대신 해당 작업 모듈이 원격 관리 대상 머신에 복사되고 그 모듈을 돌리게 됩니다. 하지만 raw 모듈은 이 방식을 따르지 않고 바로 ssh 접속 명령을 내리는 방식으로 동작합니다.

```sh
$ ansible cisco-devices -u admin -m raw -a "show clock" -vvvv
```

라고 동작을 내렸지만 

```sh
Loading callback plugin minimal of type stdout, v2.0 from /Users/mcchae/anaconda2/lib/python2.7/site-packages/ansible/plugins/callback/__init__.pyc
<10.31.1.254> ESTABLISH CONNECTION FOR USER: admin on PORT 22 TO 10.31.1.254
R1 | UNREACHABLE! => {
    "changed": false, 
    "msg": "Authentication failed.", 
    "unreachable": true
}
```

라고 오류가 나오는 것이었습니다.

이것 저것 조사를 해 보았지만 결국 raw 모듈 자체에 대한 해결 방법은, 키가 아니라 암호로 접속하는 경우에는 원 ansible 플러그인 소스를 수정해 주는 방법이 있더군요. (참고로...)

그러기 위하여 우선 설치된 ansible 모듈의 위치를 찾은 다음

```python
$ python
Python 2.7.11 (default, Jan 22 2016, 08:29:18) 
[GCC 4.2.1 Compatible Apple LLVM 7.0.2 (clang-700.1.81)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>> import ansible
>>> ansible.__file__
'/usr/local/lib/python2.7/dist-packages/ansible/__init__.pyc'
```

```sh
$ vi /usr/local/lib/python2.7/dist-packages/ansible/plugins/connection/paramiko_ssh.py
```
라고 열어

```python
226             ssh.connect(
227                 self._play_context.remote_addr,
228                 username=self._play_context.remote_user,
229                 allow_agent=allow_agent,
230                 #look_for_keys=True,
231                 look_for_keys=False,
232                 key_filename=key_filename,
233                 password=self._play_context.password,
234                 timeout=self._play_context.timeout,
235                 port=port,
236                 **sock_kwarg
237             )
```
라고 look_for_keys 를 False로 해 놓으면

```sh
$ ansible cisco-devices -u admin -m raw -a "show clock" -vvv
Using /media/psf/Home/hostFS/tdd_ansible/ansible.cfg as config file
<10.31.1.254> ESTABLISH CONNECTION FOR USER: admin on PORT 22 TO 10.31.1.254
<10.31.1.254> EXEC show clock
R1 | SUCCESS | rc=0 >>

*01:08:22.230 UTC Mon Mar 1 1993
```

라고 결과가 나옵니다.

그런데 raw 모듈을 직접 사용할 경우는 거의 없으므로 원 소스까지 수정할 필요는 없을 것 같네요.

### ios_command 테스트

여러 시행착으로 거쳤습니다만, 결국

```sh
$ ansible cisco-devices -c local -m ios_command -a "host={{ansible_ssh_host}} username=admin password=_____01 commands='show version'"
```

와 같이 명령을 내리면,

```json
R1 | SUCCESS => {
    "changed": false, 
    "stdout": [
        "Cisco IOS Software, C3560 Software (C3560-IPBASEK9-M), Version 12.2(50)SE5, RELEASE SOFTWARE (fc1)\nTechnical Support: http://www.cisco.com/techsupport\nCopyright (c) 1986-2010 by Cisco Systems, Inc.\nCompiled Tue 28-Sep-10 13:21 by prod_rel_team\nImage text-base: 0x01000000, data-base: 0x02900000\n\nROM: Bootstrap program is C3560 boot loader\nBOOTLDR: C3560 Boot Loader (C3560-HBOOT-M) Version 12.2(53r)SEY1, RELEASE SOFTWARE (fc1)\n\nSwitch uptime is 1 hour, 14 minutes\nSystem returned to ROM by power-on\nSystem image file is \"flash:/c3560-ipbasek9-mz.122-50.SE5/c3560-ipbasek9-mz.122-50.SE5.bin\"\n\n\nThis product contains cryptographic features and is subject to United\nStates and local country laws governing import, export, transfer and\nuse. Delivery of Cisco cryptographic products does not imply\nthird-party authority to import, export, distribute or use encryption.\nImporters, exporters, distributors and users are responsible for\ncompliance with U.S. and local country laws. By using this product you\nagree to comply with applicable laws and regulations. If you are unable\nto comply with U.S. and local laws, return this product immediately.\n\nA summary of U.S. laws governing Cisco cryptographic products may be found at:\nhttp://www.cisco.com/wwl/export/crypto/tool/stqrg.html\n\nIf you require further assistance please contact us by sending email to\nexport@cisco.com.\n\ncisco WS-C3560V2-24PS (PowerPC405) processor (revision J0) with 131072K bytes of memory.\nProcessor board ID FDO1436V1F7\nLast reset from power-on\n2 Virtual Ethernet interfaces\n24 FastEthernet interfaces\n2 Gigabit Ethernet interfaces\nThe password-recovery mechanism is enabled.\n\n512K bytes of flash-simulated non-volatile configuration memory.\nBase ethernet MAC Address       : E8:40:40:87:A7:80\nMotherboard assembly number     : 73-12634-01\nPower supply part number        : 341-0266-03\nMotherboard serial number       : FDO1512035H\nPower supply serial number      : DCA1504HF3B\nModel revision number           : J0\nMotherboard revision number     : A0\nModel number                    : WS-C3560V2-24PS-S\nSystem serial number            : FDO1436V1F7\nTop Assembly Part Number        : 800-33159-01\nTop Assembly Revision Number    : C0\nVersion ID                      : V05\nCLEI Code Number                : COMP500ARB\nHardware Board Revision Number  : 0x02\n\n\nSwitch Ports Model              SW Version            SW Image                 \n------ ----- -----              ----------            ----------               \n*    1 26    WS-C3560V2-24PS    12.2(50)SE5           C3560-IPBASEK9-M         \n\n\nConfiguration register is 0xF\n"
    ], 
    "stdout_lines": [
        [
            "Cisco IOS Software, C3560 Software (C3560-IPBASEK9-M), Version 12.2(50)SE5, RELEASE SOFTWARE (fc1)", 
            "Technical Support: http://www.cisco.com/techsupport", 
            "Copyright (c) 1986-2010 by Cisco Systems, Inc.", 
            "Compiled Tue 28-Sep-10 13:21 by prod_rel_team", 
            "Image text-base: 0x01000000, data-base: 0x02900000", 
            "", 
            "ROM: Bootstrap program is C3560 boot loader", 
            "BOOTLDR: C3560 Boot Loader (C3560-HBOOT-M) Version 12.2(53r)SEY1, RELEASE SOFTWARE (fc1)", 
            "", 
            "Switch uptime is 1 hour, 14 minutes", 
            "System returned to ROM by power-on", 
            "System image file is \"flash:/c3560-ipbasek9-mz.122-50.SE5/c3560-ipbasek9-mz.122-50.SE5.bin\"", 
            "", 
            "", 
            "This product contains cryptographic features and is subject to United", 
            "States and local country laws governing import, export, transfer and", 
            "use. Delivery of Cisco cryptographic products does not imply", 
            "third-party authority to import, export, distribute or use encryption.", 
            "Importers, exporters, distributors and users are responsible for", 
            "compliance with U.S. and local country laws. By using this product you", 
            "agree to comply with applicable laws and regulations. If you are unable", 
            "to comply with U.S. and local laws, return this product immediately.", 
            "", 
            "A summary of U.S. laws governing Cisco cryptographic products may be found at:", 
            "http://www.cisco.com/wwl/export/crypto/tool/stqrg.html", 
            "", 
            "If you require further assistance please contact us by sending email to", 
            "export@cisco.com.", 
            "", 
            "cisco WS-C3560V2-24PS (PowerPC405) processor (revision J0) with 131072K bytes of memory.", 
            "Processor board ID FDO1436V1F7", 
            "Last reset from power-on", 
            "2 Virtual Ethernet interfaces", 
            "24 FastEthernet interfaces", 
            "2 Gigabit Ethernet interfaces", 
            "The password-recovery mechanism is enabled.", 
            "", 
            "512K bytes of flash-simulated non-volatile configuration memory.", 
            "Base ethernet MAC Address       : E8:40:40:87:A7:80", 
            "Motherboard assembly number     : 73-12634-01", 
            "Power supply part number        : 341-0266-03", 
            "Motherboard serial number       : FDO1512035H", 
            "Power supply serial number      : DCA1504HF3B", 
            "Model revision number           : J0", 
            "Motherboard revision number     : A0", 
            "Model number                    : WS-C3560V2-24PS-S", 
            "System serial number            : FDO1436V1F7", 
            "Top Assembly Part Number        : 800-33159-01", 
            "Top Assembly Revision Number    : C0", 
            "Version ID                      : V05", 
            "CLEI Code Number                : COMP500ARB", 
            "Hardware Board Revision Number  : 0x02", 
            "", 
            "", 
            "Switch Ports Model              SW Version            SW Image                 ", 
            "------ ----- -----              ----------            ----------               ", 
            "*    1 26    WS-C3560V2-24PS    12.2(50)SE5           C3560-IPBASEK9-M         ", 
            "", 
            "", 
            "Configuration register is 0xF", 
            ""
        ]
    ], 
    "warnings": []
}
```
와 같이 결과가 잘 나왔습니다.

확인한 결과 host에 `host={{ansible_ssh_host}}` 대신 `host={{inventory_hostname}}` 라고  한 것이 화근이었습니다. 해당 IP로 접속 *ansible\_ssh_host* 해야 하는데 `R1` 이라는 *inventory_hostname* 로 접속한 것이 문제였었습니다. 

또한 `-c local` 옵션이 있는데 스위치와 같은 하드웨어는 해당 모듈을 원격으로 보내 수행할 수 없으므로 이 모듈을 돌리는 머신의 로컬에서 해당 모듈을 돌리게 됩니다. 

만약 이 옵션을 빼고 명령을 수행하면,

```
Loading callback plugin minimal of type stdout, v2.0 from /usr/local/lib/python2.7/dist-packages/ansible/plugins/callback/__init__.pyc
Using module file /usr/local/lib/python2.7/dist-packages/ansible/modules/core/network/ios/ios_command.py
<10.31.1.254> ESTABLISH CONNECTION FOR USER: admin on PORT 22 TO 10.31.1.254
R1 | UNREACHABLE! => {
    "changed": false, 
    "msg": "Authentication failed.", 
    "unreachable": true
}
```
와 같이 실패 메시지가 보이게 됩니다.


시스코 스위치는 `enable` 명령을 통하여 권한 상승 (***became***)을 한 다음 명령을 내리는 것들이 있습니다.

예를 들어 스위치에 원격에서 ssh를 통해 접속하면

```sh
Switch>
```
라는 프람프트가 뜨는데 

```sh
Switch> enable
Password: *******
Switch#
```
와 같이 권한 상승을 시키고 

```sh
Switch#show running-config 
Building configuration...

Current configuration : 3498 bytes
!
version 12.2
no service pad
service timestamps debug datetime msec
service timestamps log datetime msec
no service password-encryption
!
...
```
와 같이 결과가 나옵니다. 이것을 ios_command 를 이용해 보면,

```sh
$ ansible cisco-devices -c local -m ios_command -a "host={{ansible_ssh_host}} username=admin password=password_01 auth_pass=password__01 authorize=yes commands='show run'"

R1 | SUCCESS => {
    "changed": false, 
    "stdout": [
...
    ], 
    "warnings": []
}
```

와 같이 원하는 결과가 잘 나옵니다.

## 플레이북을 이용한 테스트

`show_version.yml` 플레이북을 다음과 같이 작성하고

```yml
---
- name: Run show clock commands
  hosts: cisco-devices
  gather_facts: false
  connection: local

  tasks:
    - name: RUN 'SHOW VERSION'
      ios_command:
        host: "{{ ansible_ssh_host }}"
        username: "admin"
        password: "password_01"
        commands:
          - show version
```

```sh
$ ansible-playbook show_version.yml 

PLAY [Run show clock commands] *************************************************

TASK [RUN 'SHOW VERSION'] ******************************************************
ok: [R1]

PLAY RECAP *********************************************************************
R1                         : ok=1    changed=0    unreachable=0    failed=0   
```

돌리면 잘 수행됨을 알 수 있습니다.

그 결과를 확인하려면 `-vvvv` 와 같이 디버깅 옵션을 강화하거나 `debug` 모듈을 플레이북에 주는 방법이 있습니다.


앞으로 더 하면서 추가된 사항은 추가하도록 하겠습니다.

어느분께는 도움이 되셨기를 ....
