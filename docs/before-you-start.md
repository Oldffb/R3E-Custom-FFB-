# R3E Custom FFB — Before you start

> Gerado por `tests/scripts/export-textos.mjs` a partir de `labels.json`.
> Instruções obrigatórias da 1.ª execução (FfbHub.NoticeVersion = 1).
> Não editar à mão: as alterações fazem-se no labels.json e volta-se a correr o script.

---

## English (official)

### Before you start

Before using this application you must turn Force Feedback OFF in RaceRoom, under Controls → Force Feedback, as shown in the images below. Only then is your device free for R3E Custom FFB to use: the application reads the data directly from the shared-memory telemetry, so there is no conflict between the two modes driving the same device.

Once you have made the change, confirm using the button below.

![RaceRoom — Force Feedback ON (must be changed)](../img/raceroom-ffb-on.png)
*✕ RaceRoom — Force Feedback ON (must be changed)*

![RaceRoom — Force Feedback OFF (correct)](../img/raceroom-ffb-off.png)
*✓ RaceRoom — Force Feedback OFF (correct)*

#### The Windows Firewall prompt

The first time you run it, Windows may ask whether to allow the application on
the network. What it actually opens is **one port, 5123/TCP, on loopback only**
(`127.0.0.1` and `::1`) — that is how the main window and the overlay load their
interface, which is a local web page. It never reaches the internet.

Loopback traffic is not filtered by Windows Firewall, so **denying the prompt
should not stop the application from working**. If port 5123 is already taken by
another program, pass a different one as a command-line argument.

More detail in [What it touches](what-it-touches.md) and
[Troubleshooting](troubleshooting.md).

---

## Português (oficial)

### Antes de começar

Antes de utilizar esta aplicação é fundamental desativar o Force Feedback do simulador RaceRoom, em Controls → Force Feedback, como mostrado nas imagens abaixo. Só desta forma o dispositivo fica livre para a aplicação R3E Custom FFB: a aplicação lê os dados diretamente da telemetria na área de memória partilhada, e assim não há conflito entre os dois modos a usar o mesmo dispositivo.

Depois de fazer o ajuste, confirme no botão abaixo.

![RaceRoom — Force Feedback LIGADO (tem de ser alterado)](../img/raceroom-ffb-on.png)
*✕ RaceRoom — Force Feedback LIGADO (tem de ser alterado)*

![RaceRoom — Force Feedback DESLIGADO (correto)](../img/raceroom-ffb-off.png)
*✓ RaceRoom — Force Feedback DESLIGADO (correto)*

#### O aviso da Firewall do Windows

Na primeira execução o Windows pode perguntar se autoriza a aplicação na rede. O
que ela abre é **uma porta, 5123/TCP, apenas em loopback** (`127.0.0.1` e `::1`)
— é assim que a janela principal e o overlay carregam a interface, que é uma
página web local. Nunca chega à internet.

O tráfego de loopback não é filtrado pela Firewall do Windows, por isso **negar o
aviso não deve impedir a aplicação de funcionar**. Se a porta 5123 já estiver
ocupada por outro programa, passe outra como argumento de linha de comandos.

Mais detalhe em [What it touches](what-it-touches.md) e
[Troubleshooting](troubleshooting.md).

---

Wondering what the application touches on your system? See
[What it touches](what-it-touches.md) — every file read or written, and why no
RaceRoom file is ever modified.
