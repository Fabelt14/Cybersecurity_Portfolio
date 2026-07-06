# Configure Initial Switch Settings

## Objective

Configure two Cisco switches from factory defaults to a secured baseline. This lab covers hostname assignment, console and privileged EXEC access control, password encryption, MOTD banner configuration, and persisting configuration to NVRAM so settings survive a power cycle.

---

## Topology



![Packet Tracer topology showing S1 and S2 switches each connected to one PC](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/Networking/Screenshots/04_01%20Packet%20Tracer%20topology%20showing%20S1%20and%20S2%20switches%20each%20connected%20to%20one%20PC.jpg)



The network consists of two switches (S1 and S2) each connected to one PC. Both switches required identical baseline security configuration applied independently.

---

## Devices Used

- S1 (Cisco Switch, 12 Fast Ethernet interfaces, 2 Gigabit Ethernet interfaces)
- S2 (Cisco Switch, same hardware profile)
- PC1 (connected to S1)
- PC2 (connected to S2)

**Default hardware profile confirmed on S1:**

| Interface Type | Count |
|---|---|
| Fast Ethernet | 12 |
| Gigabit Ethernet | 2 |
| VTY lines | 0-15 (16 simultaneous remote sessions) |

---

## Configuration

### Part 1: Verify Default Switch State

Before touching any configuration, I ran `show startup-config` to check what is stored in NVRAM.

The switch responded with "startup-config is not present." This is expected behavior on a factory-fresh device. The switch only has a running configuration in RAM at this point. Nothing has been saved to NVRAM yet, so a power cycle would wipe all changes.

This matters because an attacker who gains physical access to an unconfigured switch and power-cycles it gets a completely open device with zero access controls. The first task in any switch deployment is saving a secured configuration to NVRAM before anything else.

---

### Part 2: Basic Switch Configuration (S1)

**Step 1: Assign hostname**

```
Switch#configure terminal
Switch(config)#hostname S1
S1(config)#exit
```



![Hostname changed from Switch to S1](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/Networking/Screenshots/04_02%20Hostname%20changed%20from%20Switch%20to%20S1.jpg)


Changing the hostname from the default "Switch" is the first step in asset identification. Every device on a network needs a unique, meaningful name. If logs show suspicious activity from "Switch", identifying which physical device generated it is nearly impossible. "S1" maps directly to a documented asset.

---

**Step 2 & 3: Secure Console Access**

```
S1(config)#line console 0
S1(config-line)#password letmein
S1(config-line)#login
S1(config-line)#exit
```



![Console line configured with password and login enforcement](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/Networking/Screenshots/04_03%20Console%20line%20configured%20with%20password%20and%20login%20enforcement.jpg)

The `login` command is the critical piece here. Setting a password alone does nothing without `login`. The `login` command tells IOS to actually prompt for that password when someone connects to the console port. Without it, the password line exists in the config but authentication is never enforced.

**Verification:** After exiting and reconnecting to the console, the switch prompted for a password before allowing access.

![Console prompt requesting password after reconnection](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/Networking/Screenshots/04_04%20Console%20prompt%20requesting%20password%20after%20reconnection.jpg)

---

**Step 4 & 5: Privileged EXEC Access and the Plaintext Problem**

```
S1(config)#enable password c1$c0
```


![enable password c1$c0 configured](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/Networking/Screenshots/04_05%20enable%20password%20c1%24c0%20configured.jpg)


Running `show running-config` immediately after setting this password revealed the problem:

![Running config showing enable password c1$c0 in plaintext](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/Networking/Screenshots/04_06%20Running%20config%20showing%20enable%20password%20c1%24c0%20in%20plaintext.jpg)



The password sits in the configuration file in plaintext. Anyone with read access to the running config or a backup of it sees the password immediately. This is a known weakness of the `enable password` command.

---

**Step 6 & 7: Fix the Plaintext Problem with Enable Secret**

```
S1(config)#enable secret itsasecret
```



![enable secret configured](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/Networking/Screenshots/04_07%20enable%20secret%20configured.jpg)



Running `show run` now shows a different result:



![Running config showing enable secret hashed with MD5](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/Networking/Screenshots/04_08%20Running%20config%20showing%20enable%20secret%20hashed%20with%20MD5.jpg)



The `enable secret` value stores as an MD5 hash (`$1$mERr$ILwg/b7kc.7X/ejA4Aosn0`). Even with full access to the configuration file, an attacker cannot reverse this to recover the original password without a brute-force attack. When both `enable password` and `enable secret` are configured, IOS always uses `enable secret` and ignores `enable password`. The old plaintext entry becomes irrelevant.

---

**Step 8: Encrypt All Remaining Plaintext Passwords**

```
S1(config)#service password-encryption
```

![service password-encryption enabled](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/Networking/Screenshots/04_09%20service%20password-encryption%20enabled%20%2B%20Running%20config%20showing%20all%20passwords%20now%20encrypted.jpg)


This command applies Cisco Type 7 encoding to all currently configured plaintext passwords and all future ones. Running `show run` confirmed that the console password (previously "letmein") now appears as `7 08221D0A0A49`.

- **An important distinction:** `enable secret` uses MD5 (Type 5), which is a one-way hash. Type 7 encryption used by `service password-encryption` is a reversible obfuscation algorithm, not true encryption. Freely available tools online can decode Type 7 passwords in seconds. It stops casual observers from reading credentials at a glance, but it is not a security control against a determined attacker. The lesson is that `enable secret` provides real protection while `service password-encryption` provides only a thin layer of obfuscation.

---

### Part 3: MOTD Banner

```
S1(config)#banner motd "This is a secure system. Authorized Access Only!"
```



![MOTD banner configured on S1](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/Networking/Screenshots/04_10%20MOTD%20banner%20configured%20on%20S1.jpg)



**When it displays:** The banner appears before the login prompt, before anyone authenticates.

**Why it matters legally:** Without a warning banner, prosecuting unauthorized access in many jurisdictions is difficult. An attacker can claim they did not know the system was private. A banner stating "authorized access only" removes that defense. It establishes explicit notice before any interaction with the device. Every managed switch and router in a production network needs one.

---

### Part 4: Save Configuration to NVRAM

```
S1#copy running-config startup-config
```

![copy running-config startup-config saving to NVRAM](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/Networking/Screenshots/04_11%20copy%20running-config%20startup-config%20saving%20to%20NVRAM.jpg)

The shortest abbreviation IOS accepts: `copy run start`

Until this command runs, every configuration change lives only in RAM. A power failure or reload wipes it completely. Saving to NVRAM ensures the device boots with the secured configuration every time.

- **Verification:** Running `show startup-config` after saving confirmed all settings were recorded correctly, including the banner, hostname, encrypted passwords, and enable secret hash.

---

### Part 5: Configure S2

Applied identical security settings to S2 independently to practice the sequence without reference.

**Hostname:**
```
Switch(config)#hostname S2
```

![S2 hostname set](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/Networking/Screenshots/04_12%20S2%20hostname%20set.jpg)


**Console protection:**
```
S2(config)#line console 0
S2(config-line)#password letmein
S2(config-line)#login
```

![S2 console line configured with password](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/Networking/Screenshots/04_13%20S2%20console%20line%20configured%20with%20password.jpg)


**Enable password and enable secret:**
```
S2(config)#enable password c1$c0
S2(config)#enable secret itsasecret
```

![S2 privileged EXEC passwords configured](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/Networking/Screenshots/04_14%20S2%20privileged%20EXEC%20passwords%20configured.jpg)


**MOTD banner:**

```
S2(config)#banner motd $This is a secure system. Authorized Access Only!$
```

![S2 MOTD banner configured successfully](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/Networking/Screenshots/04_15%20S2%20MOTD%20banner%20configured%20successfully.jpg)


**Encrypt all passwords and save:**
```
S2(config)#service password-encryption
S2#copy running-config startup-config
```


![NVRAM save completed](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/Networking/Screenshots/04_16%20S2%20NVRAM%20save%20completed.jpg)



---

## Verification

Final assessment confirmed all 16 configuration items correct on both switches.



![Packet Tracer assessment showing 72/72 score with all items marked Correct](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/Networking/Screenshots/04_17%20Packet%20Tracer%20assessment%20showing%2072%20score%20with%20all%20items%20marked%20Correct.jpg)



**Score: 72/72**

Verified items across both switches:
- Banner MOTD
- Console login enforcement
- Console password
- Enable password
- Enable secret
- Service password encryption
- Hostname
- Startup config saved

---

## Key Concepts Learned

- **Running-config vs startup-config:** Running-config lives in RAM and reflects the current state. Startup-config lives in NVRAM and loads on boot. An unconfigured switch has no startup-config. Any changes not saved with `copy run start` disappear on power cycle.

- **The login command is not optional:** A password without `login` is never enforced. Both must be present for console authentication to actually work.

- **Enable password is a legacy trap:** It stores credentials in plaintext. Anyone who reads the config file gets the password. Always use `enable secret` instead. When both exist, IOS uses `enable secret` and ignores `enable password`.

- **Type 7 is not encryption:** `service password-encryption` applies a reversible algorithm that looks like encryption but is not. It stops casual observation, not deliberate attacks. Real protection comes from `enable secret` and its MD5 hashing.

- **IOS error markers are precise:** The `^` character in error output points to the exact position where the parser failed. Reading the marker saves time that would otherwise be spent guessing which part of a command was wrong.

- **MOTD banners create legal standing:** A warning displayed before authentication establishes explicit notice of restricted access. Without it, unauthorized access is harder to prosecute.

---

## Challenges Faced

- **S2 MOTD banner syntax error:** The first banner command used double quotes as delimiters, but the banner text was wrapped in the same character, causing a parser conflict. IOS flagged the error with a `^` marker pointing to the conflicting character. Switching to `$` as the delimiter resolved it immediately.

---

## Lessons Learned

- A switch running factory defaults is an open door. No console password means anyone with physical access to the console port has full control. No enable secret means anyone who reaches user EXEC mode can escalate to privileged mode without credentials. No startup-config means a power cycle resets everything.

- The sequence matters: configure credentials first, verify they work before exiting, then save. Saving a broken configuration locks you out of your own device.

- The difference between `enable password` and `enable secret` is not cosmetic. One stores plaintext, the other stores a hash. In any environment where configuration backups are stored, shared, or transmitted, that distinction determines whether credentials are exposed.

---

## Disclaimer

This lab was performed in Cisco Packet Tracer, a simulated network environment used for educational purposes. No physical network infrastructure was modified.
