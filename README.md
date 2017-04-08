# **Návrh MSSQL databáze pro evidenci rezervací pokojů**
---
#### S využitím MS `Management Studia 2014`

## Autor 
---
Jiří Jindra

## Verze
---
```
1.3.1
```
## Cíl
---
>Cílem tohoto projektu `( MSSQL evidenčního databázového systému)` je umožnit obsluze evidovat informace o rezervacích pokojů v hotelu. Tyto informace zahrnují od jakého** data** a jakým **zákázníkem** byla rezervace sjednána, který **zaměstnanec** hotelu ji zadal do systému, informace o rezervovaném pokoji a celková **délka pobytu**.

## Funkční požadavky
---
V databázovém systému jsou evidovány **veškeré** informace o dané rezervaci, jako je datum zahájení pobytu, délka samotného pobytu. Nutností také je udržovat informace o rezervovaném **pokoji**, jako je jeho **číslo**, **umístění** (ve smyslu pater), **typ** pokoje a od typu odvíjející se **cena za jednu noc**. 

Velice důležité informace představují záznamy o stávajících zákaznících, kteří již v hotelu objednávku provedli. V případě potřeby může obsluha nového zákazníka zaevidovat.

V neposlední řadě databáze shraňuje též **přihlašovací údaje zaměstnanců**, kteří vyřizují rezervace s návštěvníky. Lze tak **spětně dohledat**, který zaměstnanec uzavíral které rezervace.

 
## ER model databázového systému
---
Databáze je vytvořená pro **MSSQL Server**. V případě otevření scriptu pro vytvoření databáze ve vývojovém prostředí jiného typu databáze je **nutn**é kód pro vytvoření **mírně upravit** podle pravidel daného typu serveru

Databáze je vytvořena pro **MSSQL Server**. V případě otevření scriptu pro vytvoření databáze ve vývojovém prostředí jiného typu databáze je **nutné** kód pro tyto účely neprve **mírně upravit** podle pravidel daného typu databázového systému.

![ER model](jindra_diagram.png)
```
ER model popisující vnitřní strukturu evidenčního systému.
```

## Entitní integrita
---
Není-li uvedeno jinak, každá tabulke má `primární klíč`, který je pro každý řádek unikátní a není `NULL`.
## Doménová integirta
---
**Každá** hodnota tabulek má přiřazený `typ, popř. rozsah či masku`.
Pokud není uvedeno **jinak**, atributy jsou bez masky. Typ, povinnost a rozsah je zřejmý z **[ER modelu](#er-model-databazove-systemu)**

## Pohledy
---
Databázový systém pro evidenci rezervací pokojů v hotelovém zařízení neobsahuje **žádné pohledy*, jelikož **nebyly** ani obsluhou, ani zadavatelem **sjednány**, tudíž nejsou nevyhnutelně potřebné.

V případě **jakýkoliv budoucích potřeb** není překážka žádané pohledy do databázového systému přidat, a to na míru **požadavkům**.

## Indexy
---
V okamžik vytvoření primárních klíčů tabulek se k těmto klíčům automaticky vytvořily odpovídající **indexy**. Vytvoření indexů **urychlí** vyhledávání a tím pádem i **získání výsledku**. Jedná se o tyto primární klíče: 

**Tabulka** |  **Název**
------------|--------------
POKOJ       | pokoj_cislo
TYP         | nazev
REZERVACE   | `Composite PK z fieldů` datum_od,pokojFK, zakFK
ZAK         |  id
ZAM         | zam_login


> Následně byly pro tento databázový systém vytvořeny **explicitní indexy** pro takové atributy, které jsou ve klientu **hojně vyhledávané či vypisované**, jelikož se tím zrychlí jejich vyhledávání. 
 
 #### Indexy jsou vytvořené nad těmito atributy: 

 * **zak_jmeno + zak_prijmeni** tabulky `zak`
 ```
složený index ze jména a příjmení zákazníka, velice často vyhledávané informace
```

* **pocet_dni** tabulky `rezervace`
```
index nad tímto sloupcem urychlí vyhledávání informací o 
rezervacích hledaných podle délky pobytu
```

## Triggery
---
Triggery a procedury **usnadňují** psaní kódu, zpřehledňují ho a **umožňují** vytvoření objektů, které plní funkci, kterou žádáme či potřebujeme často. **Šetří nám tímto čas**, jelikož daný kód není nutné psát vícekrát. V případě uložené pocedury ji stačí **zavolat**, v případě triggeru ho volat **netřeba**, jelikož se provádí **automaticky** při námi určené **činnosti**.


V databázovém systému nejsou vytvořené **žádné triggery**, jelikož **zadavatelem nebyly požadovány** a designér databáze **neshledal potřebné** pro tuto situaci nutně triggery použít. V případě, že se **situace změní**, či zadavatel mírně upraví své požadavky, lze do databázového systému **dodatečně triggery přidat**, a to na míru daných požadavků

## Uložené procedury
---
V databázovém systému jsou vytvořené celkem **3 uložených procedur**, které usnadňují výpis rezervací, a to v různých zadných podmínkách.

##### Seznam vytvořených procedur:
 * **roomOccupancy**
```
Jako vstup zadáváme číslo pokoje, procedura vrátí pomocí selectu všechny informace o 
rezervacích, které byly v minulosti i přítomnosti uzavřeny pro tento pokoj
```
* **rezervaceByYear**
 ```
 Do vstupu této procedury posíláme rok v číselné podobě. Procedura vrací počet celkových 
 rezervací uzavřených pro všechny pokoje v daném roce, který je poslán jako parametr.
```

* **rezervaceByDays**
```
Procedura vrací všechny rezervace, jejichž délka pobytu (v délce celých dnů) odpovídá 
parametru, který jsme při volání této procedur předali.
```

## Ukázkové selecty
>Následující výčet selectů ukazuje, na přibližně **jaké principu** klient pracuje a jaká data a jakým způsobem zpracovává. Dané selecty jsou **statické**, což znamená, že vracejí** konkrétní záznamy**. V klientu samozřejmě statické nejsou, ale hledané data si uživatel **sám** doplní podle toho, co ho **v danou chvíli  nejvíce zajímá**.

##### a) Všechny 14 denní rezervace (klient,pokoj,typ) seřazené podle data_od (sestupně)
```
select
rezervace.datum_od, rezervace.pocet_dni ,zak.zak_jmeno, zak.zak_prijmeni, zak.zak_email, 
pokoj.pokoj_cislo, pokoj.patro,typ.nazev, typ.cena_noc 
from 
rezervace 
inner join pokoj on rezervace.pokojFK = pokoj.pokoj_cislo 
inner join typ on pokoj.typFK = typ.nazev 
inner join zak on zak.id=rezervace.zakFK 
where pocet_dni = 14 
order by rezervace.datum_od desc;
```

##### b) Počet rezervací za rok 2015 a za rok 2016
```
select count(*) 'Počet rezervací v zadaném roce' from rezervace 
where datepart(year,datum_od)=2015;
```
```
select count(*) 'Počet rezervací v zadaném roce' from rezervace 
where datepart(year,datum_od)=2016;
```

##### c) Obsazení pokoje 6 (kdy,kdo,jak dlouho,celkovou cenu)
```
select 
rezervace.datum_od, rezervace.pocet_dni ,zak.zak_jmeno, zak.zak_prijmeni, zak.zak_email, 
pokoj.pokoj_cislo, pokoj.patro,typ.nazev, 
typ.cena_noc * rezervace.pocet_dni 'Cena celkem' 
from 
rezervace 
inner join pokoj on rezervace.pokojFK = pokoj.pokoj_cislo 
inner join typ on pokoj.typFK = typ.nazev 
inner join zak on zak.id=rezervace.zakFK 
where pokoj_cislo = 6;
```

## Postup a požadavky na import databázové části
---
##### Požadované programové vybavení a ostatní:

•	Microsoft Management Studio 2014 a vyšší
•	Microsoft Poznámkový blok (Notepad)
•	Funkční vysokorychlostní internetové připojení
•	Silné nervy při výskytu chyb


##### Postup při importu vyexporotvaného scriptu *(.txt)* na stávající server (v bodech) s použitím Management Studia 2014:
1. Přihlaste se na svůj server pod Vašimi přihlašovacími údaji
2. Následně vytvořte nové příkazové okno tak zvané **Querry Window**, a to kiknutím na tlačítko `New Querry`.
3. Otevře se nové `SQLQUerry` pro zadávání příkazů. Pro import této databáze se předpokládá existence **alespoň jedné** (prázdné) databáze na Vašem serveru
4. Otevřete si textový soubor s exportem databáze, zkopírujte celý text a ten vložte do nového `QuerryWindow`
5. Po zkopírování celého kódu tentýž kód označte a v Management studiu stiskněte tlačítko `Execute`, čímž se kód provede a změny se projeví na databázovém serveru

## Požadavky na spuštění klienta
---
Jako každý program má I klient k tomuto databázovému systému své požadavky a podmínky, za kterých může být uživatelem úspěšně spuštěn.

##### Mezi tyto podmínky patří zejména:

* Nainstalovaný **Java virtual machine** pro spuštění. jar souboru klienta
*	Funkční internetové připojení pro přístup k databázovému server s databázovým systémem
*	Minimální volné místo na disku : alespoň **6 MB**.
*	Aplikace v neupravené formě lze spustit na operačních systémech Windows/ MacOS / Linux
*	Procesor minimálně **Pentium II 266 Mhz**
*	Aplikace podporuje `32-bitové` i `64-bitové` systémy
            Postup spuštení programu:
*	Stáhnout si `.jar` soubor klienta vytvořeného pro tento databázový system
*	Umístit ho na plochu či do Vámi zvoleného adresáře
*	Otevřít **dvojitým** poklepáním myši
*	Obsluha zadá uživatelské **jmeno a heslo** daného pracovníka podniku – udaje jsou předem vytvořené administrátorem databáze v jiném nástroji, tudíž **srkze klienta** není možno přidávat další zaměstnance. Klient je slouží čistě pro ukládání a správu rezervací a zákazníků.
*	Posledním krokem je stisknutí tlačítka **LogIn**, které v případě, že zadané přihlašovací údaje obsluhy korespondují s některými s uložených, dojde k přihlášení a umožnění vytváření zákazníků a rezervací


## Závěr
---
Výsledný databázový systém by se dal s možnými **dílčími úpravami** použít v menším restauračním zařízení. Obsahuje **vešeré důležité** aspekty a splňuje veškeré normy a legislativu ČR. V případě **odkupu** zahraniční firmou by má osoba, či pověření osoba, musel databázi **zkontrolovat** a případně **upravit** pro legislativu dané země. 

 Rozsah databáze samozřejmě **není konečný** a v případě potřeby může být **rozšířen** o určité aspekty, které kupující bude v budoucnu požadovat, pokud to **nebude v rozporu** se zákonem.




