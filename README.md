### Задание 1
Ответьте на следующие вопросы:  

-   Какие сетевые службы в ней разрешены?  
Сканирование nmap выявило ряд запущенных сервисов, полный список на скрине  
![nmap](https://github.com/RSafin12/13.1-Vulnerabilities-and-attacks/blob/main/nmap.png)  
Вывод nmap показывает, что запущены самые популярные сервисы и они работают на очевидных портах, например, FTP на 21 и SSH на 22. Это уже облегчает задачу для злоумышленyика. 
Далее я попробовал nmap с ключjм `-A` для более подробного вывода.   

-   Какие уязвимости были вами обнаружены? (список со ссылками: достаточно трёх уязвимостей)  
Агрессивное сканирование вывело больше информации,  начнем по порядку, а именно:   
1. FTP  
```
21/tcp   open  ftp         vsftpd 2.3.4
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
22/tcp   open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0)
```
На сайте нашел информацию про эту [уязвимость](https://www.exploit-db.com/exploits/49757)  

2. MYSQL  
```
3306/tcp open  mysql       MySQL 5.0.51a-3ubuntu5
| mysql-info: Protocol: 10
| Version: 5.0.51a-3ubuntu5
| Thread ID: 9
| Some Capabilities: Connect with DB, Compress, SSL, Transactions, Secure Connection
| Status: Autocommit
|_Salt: ?:E*DUaf]!'BUdvF:0~L
```
Если я правильно понял, то данная версия MySQL подвержена DoS атаками такого [типа](https://www.exploit-db.com/exploits/41954)  
3. OpenSSH версии 4.7  
```
22/tcp   open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0)
| ssh-hostkey: 1024 60:0f:cf:e1:c0:5f:6a:74:d6:90:24:fa:c4:d5:6c:cd (DSA)
|_2048 56:56:24:0f:21:1d:de:a7:2b:ae:61:b1:24:3d:e8:f3 (RSA)
```
Ну и тут их больше всего, под версию OpenSSH 4.7 я нашел [это](https://www.exploit-db.com/exploits/45210) и [это](https://www.exploit-db.com/exploits/45233)  

4. Ну и открытый telnet сам по себе дыра. Можно перехватить трафик Wireshark и например увидеть логин/пароль.   
![telnet](https://github.com/RSafin12/13.1-Vulnerabilities-and-attacks/blob/main/telnet.png)  

### Задание 2  

В начале стоит упомянуть о 6 состояниях портов. 
-  открыт (open). Это значит, что приложение принимает запросы на порт - прямой путь для атаки. Желательно защитить порт с помощью firewall  
-  закрыт (closed). Порт доступен, но не используется(пока) каким-либо приложением.  
-  фильтруется (filtered). Значит, что nmap не может определить, открыт ли порт, т.к. он фильтруется. Обычно фильтрация осуществляется firewall на целевом хосте или промежуточным роутером/аппаратным firewall. Чаще всего в этом случае не отправляются какие-либо ответные пакеты, что заставляет nmap снова отправлять несколько запросов, что замедляет сканирование.   
- не фильтруется (unfiltered). Это значит, что порт доступен, но nmap не может точно определить его состояние. SYN/FIN в данном случае помогут определить точнее состояние порта  
- открыт|фильтруется (open|filtered). Nmap не может определить, открыт порт или фильтруется. Такое случается, когда открытые порты не отвечают.   
- закрыт|фильтруется (closed|filtered). Nmap не может определить, закрыт порт или фильтруется.  

##### SYN
`Nmap done: 1 IP address (1 host up) scanned in 0.29 seconds`  
SYN-сканирование считается самым популярным способом. Быстро запускается и сканирует тысячи портов в секунду, его работе не препятствуют фаерволлы. SYN-сканирование не устанавливает полноценного соединения, поэтому он малозаметен. Он работает с любым TCP-стеком и не зависит от особенностей платформы как это происходит при сканировании FIN или Xmass. 
Отправляются очень много SYN-пакетов, типа пытаясь установить соединение. 
Ответы SYN/ACK говорят о том, что порт открыт, RST - сбрасывается, не прослушивается. А если после нескольких запросов ничего не приходит обратно, то порт отмечается как фильтруемый.   
```
> sudo nmap -sS 192.168.3.33
Starting Nmap 7.80 ( https://nmap.org ) at 2023-04-23 11:25 MSK
Nmap scan report for 192.168.3.33
Host is up (0.00015s latency).
Not shown: 977 closed ports
PORT     STATE SERVICE
21/tcp   open  ftp
22/tcp   open  ssh
23/tcp   open  telnet
25/tcp   open  smtp
```
![SYN](https://github.com/RSafin12/13.1-Vulnerabilities-and-attacks/blob/main/SYN.png)  

##### FIN/Xmass
`Nmap done: 1 IP address (1 host up) scanned in 1.52 seconds` - FIN  
`Nmap done: 1 IP address (1 host up) scanned in 1.49 seconds` - Xmass  
В данных случаях используется правила TCP RFC, которые гласят, что любой пакет не содержащий установленного бита SYN, RST или ACK, ответит RST пакетом, если порт закрыт. Схема работы одна и та же, разница только в установленных флагах.   
И как говорилось выше, если ответ RST - то порт закрыт, если ICMP ошибка о недостижимости, то порт фильтруется. Отсутствие ответа - открыт | фильтруется.  

Считается, что такое сканирование очень незаметное, позволяет обойти фаерволы и фильтрацию. Однако недостатком является то, что сканирование будет работать только в том случае, если атакуемый объем следует RFC, а так происходит не всегда. Например, некоторые Cisco просто отправят RST ответ, даже если порт открыт. 
К недостаткам относят также то, что сканирование менее конкретное, т.к. присутствует обобщение открыт | фильтруется.   
```
> sudo nmap -sF 192.168.3.33
Starting Nmap 7.80 ( https://nmap.org ) at 2023-04-23 11:26 MSK
Nmap scan report for 192.168.3.33
Host is up (0.00032s latency).
Not shown: 977 closed ports
PORT     STATE         SERVICE
21/tcp   open|filtered ftp
22/tcp   open|filtered ssh
23/tcp   open|filtered telnet
25/tcp   open|filtered smtp
```
FIN  
![FIN](https://github.com/RSafin12/13.1-Vulnerabilities-and-attacks/blob/main/FIN.png)  
Xmass  
![Xmas](https://github.com/RSafin12/13.1-Vulnerabilities-and-attacks/blob/main/Xmas.png)  

##### UDP
UDP-сканирование считается самым медленным, также они сильнее забивает канал.   
UDP-сканирование отправляет пустой UDP-заголовок на целевой порт. Если в ответ поступает ICMP ответ с ошибкой недостижимости порта, то значит порт закрыт. Другие ошибки могут указывать на то, что порт фильтруется. Если служба отвечает UDP-пакетом, то порт открыт.   
<details>
![UDP](https://github.com/RSafin12/13.1-Vulnerabilities-and-attacks/blob/main/UDP.png)
</details>


