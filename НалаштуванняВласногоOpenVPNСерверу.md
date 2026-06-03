# Налаштування власного VPN серверу за домогою клієнта OpenVPN
Відкрити термінал `cmd` та, за допогомою стандартної утиліти `ssh`, підключитись до серверу від імені користувача `root`:
```powershell
ssh root@<ip_серверу>
```
З'явиться текст із підтвердженням, де необхідно вказати `yes`:
```
The authenticity of host '***.***.***.*** (***.***.***.***)' can't be established.
****** key fingerprint is **************************************************.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
```
І далі буде пропозиція введення паролю, вказати відповідний пароль за користувачем `root`:
```
root@***.***.***.***s password:
```
Після успішного підключення повинно з'явитись подібне повідомлення:
```
Linux ****************** *.*.*-**-**** #1 SMP PREEMPT_DYNAMIC ****** *.*.***-* (****-**-**) x**_**
The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.
Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: **** ****  * **:**:** **** from ***.***.***.***
root@dedicate:~#
```
(example)[example.jpg]
2.
