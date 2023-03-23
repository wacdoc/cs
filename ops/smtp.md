# Sestavte si svůj vlastní server pro odesílání pošty SMTP

## preambule

SMTP může přímo nakupovat služby od cloudových dodavatelů, jako jsou:

* [Amazon SES SMTP](https://docs.aws.amazon.com/ses/latest/dg/send-email-smtp.html)
* [Ali cloud e-mail push](https://www.alibabacloud.com/help/directmail/latest/three-mail-sending-methods)

Můžete si také vytvořit svůj vlastní poštovní server - neomezené odesílání, nízké celkové náklady.

Níže si ukážeme krok za krokem, jak vybudovat vlastní poštovní server.

## Výběr serveru

Server SMTP s vlastním hostitelem vyžaduje veřejnou IP s otevřenými porty 25, 456 a 587.

Běžně používané veřejné cloudy tyto porty ve výchozím nastavení blokují a je možné je otevřít zadáním pracovního příkazu, ale je to nakonec velmi problematické.

Doporučuji nákup od hostitele, který má tyto porty otevřené a podporuje nastavení reverzních doménových jmen.

Zde doporučuji [Contabo](https://contabo.com) .

Contabo je poskytovatel hostingu se sídlem v Mnichově, Německo, založený v roce 2003 s velmi konkurenčními cenami.

Pokud jako měnu nákupu zvolíte euro, cena bude levnější (server s 8GB pamětí a 4 CPU stojí asi 529 juanů ročně a počáteční instalační poplatek je na jeden rok zdarma).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/UoAQkwY.webp)

Při zadávání objednávky uveďte `prefer AMD` a server s CPU AMD bude mít lepší výkon.

V následujícím textu vezmu Contabo's VPS jako příklad, který demonstruje, jak vytvořit svůj vlastní poštovní server.

## Konfigurace systému Ubuntu

Operačním systémem je zde Ubuntu 22.04

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/smpIu1F.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/m7Mwjwr.webp)

Pokud server na ssh zobrazí `Welcome to TinyCore 13!` (jak je znázorněno na obrázku níže), znamená to, že systém ještě nebyl nainstalován. Odpojte ssh a počkejte několik minut, než se znovu přihlásíte.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/-qKACz9.webp)

Když se zobrazí `Welcome to Ubuntu 22.04.1 LTS` , inicializace je dokončena a můžete pokračovat následujícími kroky.

### [Volitelné] Inicializujte vývojové prostředí

Tento krok je volitelný.

Pro pohodlí jsem instalaci a konfiguraci systému softwaru ubuntu vložil do [github.com/wactax/ops.os/tree/main/ubuntu](https://github.com/wactax/ops.os/tree/main/ubuntu) .

Spusťte následující příkaz pro instalaci jedním kliknutím.

```
bash <(curl -s https://raw.githubusercontent.com/wactax/ops.os/main/ubuntu/boot.sh)
```

Čínští uživatelé, použijte místo toho následující příkaz a jazyk, časové pásmo atd. se nastaví automaticky.

```
CN=1 bash <(curl -s https://ghproxy.com/https://raw.githubusercontent.com/wactax/ops.os/main/ubuntu/boot.sh)
```

### Contabo umožňuje IPV6

Povolte IPV6, aby SMTP mohl také odesílat e-maily s adresami IPV6.

upravit `/etc/sysctl.conf`

Upravte nebo přidejte následující řádky

```
net.ipv6.conf.all.disable_ipv6 = 0
net.ipv6.conf.default.disable_ipv6 = 0
net.ipv6.conf.lo.disable_ipv6 = 0
```

Pokračujte [výukovým programem contabo: Přidání připojení IPv6 na váš server](https://contabo.com/blog/adding-ipv6-connectivity-to-your-server/)

Upravte `/etc/netplan/01-netcfg.yaml` , přidejte několik řádků, jak je znázorněno na obrázku níže (výchozí konfigurační soubor Contabo VPS tyto řádky již obsahuje, stačí je odkomentovat).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/5MEi41I.webp)

Poté `netplan apply` , aby se změněná konfigurace projevila.

Po úspěšné konfiguraci můžete použít `curl 6.ipw.cn` k zobrazení adresy ipv6 vaší externí sítě.

## Klonujte operační úložiště konfigurace

```
git clone https://github.com/wactax/ops.soft.git
```

## Vygenerujte si bezplatný certifikát SSL pro název své domény

Odesílání pošty vyžaduje certifikát SSL pro šifrování a podepisování.

Ke generování certifikátů používáme [acme.sh.](https://github.com/acmesh-official/acme.sh)

acme.sh je open source automatický nástroj pro podepisování certifikátů,

Vstupte do konfiguračního skladu ops.soft, spusťte `./ssl.sh` a v **horním adresáři** se vytvoří složka `conf` .

Najděte svého poskytovatele DNS z [acme.sh dnsapi](https://github.com/acmesh-official/acme.sh/wiki/dnsapi) , upravte `conf/conf.sh` .

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/Qjq1C1i.webp)

Poté spusťte `./ssl.sh 123.com` a vygenerujte certifikáty `123.com` a `*.123.com` pro název vaší domény.

První spuštění automaticky nainstaluje [acme.sh](https://github.com/acmesh-official/acme.sh) a přidá naplánovanou úlohu pro automatické obnovení. Můžete vidět `crontab -l` , je tam takový řádek jako následující.

```
52 0 * * * "/mnt/www/.acme.sh"/acme.sh --cron --home "/mnt/www/.acme.sh" > /dev/null
```

Cesta k vygenerovanému certifikátu je něco jako `/mnt/www/.acme.sh/123.com_ecc。`

Obnova certifikátu zavolá skript `conf/reload/123.com.sh` , upravte tento skript, můžete přidat příkazy jako `nginx -s reload` pro obnovení mezipaměti certifikátů souvisejících aplikací.

## Sestavte SMTP server s chasquidem

[chasquid](https://github.com/albertito/chasquid) je open source SMTP server napsaný v jazyce Go.

Jako náhrada za staré programy pro poštovní servery, jako jsou Postfix a Sendmail, je chasquid jednodušší a snadněji použitelný a je také jednodušší pro sekundární vývoj.

Spustit `./chasquid/init.sh 123.com` se nainstaluje automaticky jedním kliknutím (nahraďte 123.com názvem vaší odesílací domény).

## Nakonfigurujte DKIM pro podpis e-mailu

DKIM se používá k odesílání e-mailových podpisů, aby se zabránilo tomu, že dopisy budou považovány za spam.

Po úspěšném spuštění příkazu budete vyzváni k nastavení záznamu DKIM (jak je uvedeno níže).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/LJWGsmI.webp)

Stačí přidat záznam TXT do vašeho DNS (jak je uvedeno níže).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/0szKWqV.webp)

## Zobrazení stavu služby a protokolů

 `systemctl status chasquid` Zobrazení stavu služby.

Stav normálního provozu je znázorněn na obrázku níže

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/CAwyY4E.webp)

 `grep chasquid /var/log/syslog` nebo `journalctl -xeu chasquid` může zobrazit protokol chyb.

## Reverzní konfigurace názvu domény

Reverzní název domény má umožnit překlad IP adresy na odpovídající název domény.

Nastavení reverzního názvu domény může zabránit tomu, aby byly e-maily identifikovány jako spam.

Když je pošta přijata, přijímající server provede reverzní analýzu názvu domény na IP adrese odesílajícího serveru, aby potvrdil, zda má odesílající server platný reverzní název domény.

Pokud odesílající server nemá reverzní název domény nebo pokud se reverzní název domény neshoduje s IP adresou odesílajícího serveru, může přijímající server rozpoznat e-mail jako spam nebo jej odmítnout.

Navštivte [https://my.contabo.com/rdns](https://my.contabo.com/rdns) a nakonfigurujte, jak je uvedeno níže

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/IIPdBk_.webp)

Po nastavení reverzního názvu domény nezapomeňte nakonfigurovat dopředné rozlišení názvu domény ipv4 a ipv6 na server.

## Upravte název hostitele chasquid.conf

Upravte `conf/chasquid/chasquid.conf` na hodnotu reverzního názvu domény.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/6Fw4SQi.webp)

Poté spusťte `systemctl restart chasquid` a restartujte službu.

## Zálohujte conf do úložiště git

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/Fier9uv.webp)

Například zálohuji složku conf do svého vlastního procesu github následovně

Nejprve vytvořte soukromý sklad

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/ZSCT1t1.webp)

Vstupte do adresáře conf a odešlete do skladu

```
git init
git add .
git commit -m "init"
git branch -M main
git remote add origin git@github.com:wac-tax-key/conf.git
git push -u origin main
```

## Přidat odesílatele

běh

```
chasquid-util user-add i@wac.tax
```

Lze přidat odesílatele

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/khHjLof.webp)

### Ověřte, zda je heslo nastaveno správně

```
chasquid-util authenticate i@wac.tax --password=xxxxxxx
```

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/e92JHXq.webp)

Po přidání uživatele bude `chasquid/domains/wac.tax/users` aktualizováno, nezapomeňte jej odeslat do skladu.

## DNS přidat SPF záznam

SPF (Sender Policy Framework) je technologie ověřování e-mailů, která se používá k prevenci podvodů s e-maily.

Ověřuje identitu odesílatele pošty tím, že kontroluje, zda IP adresa odesílatele odpovídá DNS záznamům názvu domény, za kterou se vydává, a zabraňuje podvodníkům v zasílání falešných e-mailů.

Přidáním SPF záznamů můžete maximálně zabránit tomu, aby byly e-maily identifikovány jako spam.

Pokud váš doménový server nepodporuje typ SPF, stačí přidat záznam typu TXT.

Například SPF `wac.tax` je následující

`v=spf1 a mx include:_spf.wac.tax include:_spf.google.com ~all`

SPF pro `_spf.wac.tax`

`v=spf1 a:smtp.wac.tax ~all`

Všimněte si, že zde mám `include:_spf.google.com` , protože později nakonfiguruji `i@wac.tax` jako odesílací adresu v poštovní schránce Google.

## Konfigurace DNS DMARC

DMARC je zkratka pro (Domain-based Message Authentication, Reporting & Conformance).

Používá se k zachycení nedoručených zpráv SPF (možná způsobených chybami konfigurace nebo se za vás někdo vydává, aby rozesílal spam).

Přidat záznam TXT `_dmarc` ,

Obsah je následující

```
v=DMARC1; p=quarantine; fo=1; ruf=mailto:ruf@wac.tax; rua=mailto:rua@wac.tax
```

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/k44P7O3.webp)

Význam každého parametru je následující

### p (zásady)

Označuje, jak zacházet s e-maily, které selžou při ověření SPF (Sender Policy Framework) nebo DKIM (DomainKeys Identified Mail). Parametr p lze nastavit na jednu ze tří hodnot:

* žádné: Neprovede se žádná akce, pouze výsledek ověření je odeslán zpět odesílateli prostřednictvím mechanismu e-mailových zpráv.
* Karanténa: Uložte poštu, která neprošla ověřením, do složky nevyžádané pošty, ale neodmítne ji přímo.
* odmítnout: Přímé odmítnutí e-mailů, které neprojdou ověřením.

### fo (možnosti selhání)

Určuje množství informací vrácených mechanismem hlášení. Lze jej nastavit na jednu z následujících hodnot:

* 0: Hlásit výsledky ověření pro všechny zprávy
* 1: Hlásit pouze zprávy, které neprošly ověřením
* d: Hlásit pouze selhání ověření názvu domény
* s: hlásit pouze selhání ověření SPF
* l: Hlásit pouze selhání ověření DKIM

### rua & ruf

* rua (URI hlášení pro souhrnné zprávy): E-mailová adresa pro příjem souhrnných zpráv
* ruf (URI hlášení pro forenzní zprávy): e-mailová adresa pro zasílání podrobných zpráv

## Přidejte záznamy MX pro přeposílání e-mailů do Google Mail

Protože jsem nemohl najít bezplatnou podnikovou schránku, která by podporovala univerzální adresy (Catch-All, může přijímat jakékoli e-maily odeslané na toto doménové jméno, bez omezení prefixů), použil jsem k přeposílání všech e-mailů do své schránky Gmailu chasquid.

**Pokud máte vlastní placenou firemní poštovní schránku, neupravujte MX a tento krok přeskočte.**

Upravit `conf/chasquid/domains/wac.tax/aliases` , nastavit poštovní schránku pro přeposílání

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/OBDl2gw.webp)

`*` označuje všechny e-maily, `i` je předpona e-mailové adresy odesílajícího uživatele vytvořená výše. Pro přeposílání pošty musí každý uživatel přidat řádek.

Poté přidejte záznam MX (zde ukazuji přímo na adresu reverzního názvu domény, jak je znázorněno na prvním řádku na obrázku níže).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/7__KrU8.webp)

Po dokončení konfigurace můžete použít jiné e-mailové adresy k odesílání e-mailů na `i@wac.tax` a `any123@wac.tax` , abyste zjistili, zda můžete přijímat e-maily v Gmailu.

Pokud ne, zkontrolujte protokol chasquid ( `grep chasquid /var/log/syslog` ).

## Pošlete e-mail na adresu i@wac.tax pomocí služby Google Mail

Poté, co Google Mail obdržel e-mail, přirozeně jsem doufal, že odpovím s `i@wac.tax` místo i.wac.tax@gmail.com.

Navštivte [https://mail.google.com/mail/u/1/#settings/accounts](https://mail.google.com/mail/u/1/#settings/accounts) a klikněte na „Přidat další e-mailovou adresu“.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/PAvyE3C.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/_OgLsPT.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/XIUf6Dc.webp)

Poté zadejte ověřovací kód přijatý e-mailem, na který byl přeposlán.

Nakonec ji lze nastavit jako výchozí adresu odesílatele (spolu s možností odpovědět stejnou adresou).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/a95dO60.webp)

Tímto jsme dokončili zřízení poštovního serveru SMTP a zároveň využíváme Google Mail k odesílání a přijímání e-mailů.

## Odesláním zkušebního e-mailu zkontrolujte, zda je konfigurace úspěšná

Zadejte `ops/chasquid`

Spustit `direnv allow` instalaci závislostí (direnv byl nainstalován v předchozím inicializačním procesu s jedním klíčem a do shellu byl přidán hák)

pak běž

```
user=i@wac.tax pass=xxxx to=iuser.link@gmail.com ./sendmail.coffee
```

Význam parametrů je následující

* uživatel: uživatelské jméno SMTP
* heslo: heslo SMTP
* komu: příjemce

Můžete poslat zkušební e-mail.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/ae1iWyM.webp)

Doporučuje se používat Gmail k přijímání testovacích e-mailů, abyste zkontrolovali, zda jsou konfigurace úspěšné.

### Standardní šifrování TLS

Jak je znázorněno na obrázku níže, je zde tento malý zámek, což znamená, že certifikát SSL byl úspěšně povolen.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/SrdbAwh.webp)

Poté klikněte na „Zobrazit původní e-mail“

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/qQQsdxg.webp)

### DKIM

Jak je znázorněno na obrázku níže, původní poštovní stránka Gmailu zobrazuje DKIM, což znamená, že konfigurace DKIM byla úspěšná.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/an6aXK6.webp)

Zaškrtněte políčko Received v hlavičce původního e-mailu a uvidíte, že adresa odesílatele je IPV6, což znamená, že IPV6 je také úspěšně nakonfigurován.
