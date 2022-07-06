# A1Z26

Can you find the flag?

## Assuntos relacionados ao desafio

- Criptografia
- Esteganografia

## Solução

O desafio disponibiliza dois arquivos para download, [images.zip](images.zip) e
[text.txt](text.txt). O arquivo zip é protegido por senha e o arquivo de texto
parece ser a senha do arquivo zip, pois contém apenas a palavra "zup" repetida
várias vezes e algumas estão incompletas:

```
zupzu
zupzupzupzupzupzupzup
z
zupzupzupzupz
zupzupzupzupzup
z
zupzupzupzupzupzupzupzupzu
zupzupzupzupzupzupzup
zupzupzupzupzupz
```

Depois de analizar o conteúdo dos arquivos, procurei o que seria `A1Z26`, o
título do desafio. A primeira resposta foi o site
https://www.dcode.fr/letter-number-cipher que é uma ferramenta para criptografar
e descriptografar a cifra A1Z26. Mas como funciona a cifra A1Z26? A própria
página da ferramenta contém a explicação:

> A Cifra Letra-para-Número (ou Cifra Número-para-Letra ou alfabeto numerado)
consiste em substituir cada letra pela sua **posição no alfabeto**, por exemplo
A=1, B=2, Z=26, daí seu sobrenome **A1Z26**.

Beleza, então a gente precisa de uma sequência de números para usar a cifra
A1Z26 e usar como senha no arquivo zip, mas onde estão esses números? Cada linha
tem um determinado número de letras e ai está a resposta, trocando cada linha
pela quantidade de letras fica:

```
5
21
1
13
15
1
26
21
16
```

Utilizando a sequência `5 21 1 13 15 1 26 21 16` no site conseguimos o seguinte
resultado:

```
euamoazup
```

Utilizando como senha no arquivo zip, extrai as imagens:

```
$ unzip -d images -P 'euamoazup' images.zip

Archive:  images.zip
  inflating: images/JPG/LogoCMYK_Vertical.jpg  
  inflating: images/JPG/LogoSimbolo_.jpg  
  inflating: images/JPG/LogoCMYK.jpg
```

O parâmetro `-d` serve para informar em qual pasta extrair os arquivos e o
parâmetro `-P` serve para informar a senha do arquivo zip. Abrindo as imagens
normalmente, são variações do logo da Zup, temos que procurar em outro lugar.
Olhando os metadados das imagens com a ferramenta [ExifTool][exiftool] não foi
possível encontrar nada, apenas metadados normais. Usando a ferramenta
[Stegsolve][stegsolve] para variar as propriedades da imagem, também não foi
possível encontrar nada. O que nos resta é olhar o binário da imagem, olhando
cada imagem, foi possível encontrar a flag na seguinte imagem:

```
$ strings images/JPG/LogoCMYK_Vertical.jpg | grep -i ZUP-CTF

  ZUP-CTF{zup1nh4r0x}
```

[exiftool]: https://exiftool.org
[stegsolve]: http://www.caesum.com/handbook/stego.htm
