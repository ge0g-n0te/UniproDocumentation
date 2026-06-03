# Налаштування власного VPN серверу за домогою клієнта OpenVPN
Відкрити термінал `cmd` та, за допогомою стандартної утиліти `ssh`, підключитись до серверу:
```powershell
ssh root@<ip_серверу>
```
З'явиться текст із підтвердженням, де необхідно вказати `yes`:

```powershell
The authenticity of host '***.***.***.*** (***.***.***.***)' can't be established.
****** key fingerprint is **************************************************.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
```
І далі буде пропозиція введення паролю
```powershell
root@***.***.***.***s password:
```
2.
