# ThinkPad T480 ujjlenyomat-olvasó Fedora 44 alatt

**Az olvasó felismerte az ujjat, de a meghajtó elutasította a találatot.** Ezt a hibát sikerült azonosítani és helyben javítani egy Synaptics/Validity `06cb:009a` olvasóval felszerelt ThinkPad T480-on.

Ez egyetlen gépen kipróbált esetleírás, 2026. október 22-i állapottal. Nem általános telepítő, nem hivatalos Fedora-csomag és nem biztonsági audit.

## A környezet

Fedora 44 KDE, Plasma 6.7.5, fprintd 1.94.5, kernel 7.2.11-200.fc44.x86_64. A készülék Windowsban és python-validityvel mentett mintákat tartalmazott.

A sikeres megoldás alapja a [libfprint !626 fejlesztési javaslat](https://gitlab.freedesktop.org/libfprint/libfprint/-/merge_requests/626), `1cf22b89c335b1b63efcccb1113234be9ba2e9d0` alapcommittal és helyi módosításokkal. Nem állítjuk, hogy a kiadott libfprint alapból támogatja ezt az olvasót.

## Az út a hibáig

A python-validity/open-fprintd párossal sikerült mintát rögzíteni, de a felismerés többnyire sikertelen volt. USB-megszakadást és szolgáltatás-újraindulást is láttunk. Az ismételt rögzítés, újrakalibrálás és az USB-elaltatás kikapcsolása nem hozott megbízható megoldást. Nem bizonyítottuk, hogy minden korábbi tünet ugyanabból a hibából eredt.

Egy Rust nyelvű kísérleti meghajtó sikeresen felépítette a titkosított kapcsolatot, de a felismerés ebben a próbában nem működött megbízhatóan.

A natív fejlesztési meghajtó részletes naplója mutatta meg a döntő különbséget:

```
Identify: chip matched user_dbid=<record> subtype=245,
which is no print in the gallery -- no match
NOT IDENTIFIED!
```

A készülék tehát már talált egyezést; a szoftveres hozzárendelés utasította el.

## A két kódprobléma

1. A régi bináris felhasználóazonosítókat szövegként kezelte a meghajtó. A Windows SID-azonosítók beágyazott nullabájtjai miatt nem maradt meg a teljes azonosságvizsgálathoz szükséges tartalom és hossz.
2. A meghajtó az 1–10 közötti anatómiai ujjazonosítókat ismerte, a készülék viszont 245–248-as régi mintákat is tartalmazott. Ezek eredeti értékét külön meg kellett őrizni.

A [helyi patch](patches/legacy-validity-identities.patch) a bináris azonosítót és az eredeti ujjazonosítót együtt őrzi meg. Az ellenőrzés továbbra is a pontos felhasználórekordot, azonosítóbájtokat, hosszt, ujjazonosítót és a rekordban lévő megfelelő mintát követeli meg. Nem fogadunk el automatikusan tetszőleges készülékbeli találatot.

## Mi működött a végén?

- A Validity tesztcsomag mind a 202 tesztje sikerült: 201 meglévő és egy új regressziós teszt.
- A külön példaprogramban ugyanazzal az ujjal hat egymást követő felismerés sikerült, két különböző régi rekordhoz rendelve.
- Mindkét mutatóujj és a jobb hüvelykujj elsőre működött a példaprogram próbájában.
- A hivatalos Fedora fprintd szolgáltatás külön könyvtárból betöltött javított libfprinttel és helyileg importált referenciákkal működik.
- A felhasználó újraindítás után is igazolta az ujjlenyomatos sudo és a KDE-képernyőzár működését.
- Egy későbbi szolgáltatásos próba 4-ből 3 sikeres olvasást adott. Ez nem reprezentatív találatiarány-mérés.
- A végső Linux-módosítások után a Windows működését nem teszteltük újra.

## Könnyen félrevezető részletek

A hivatalos kliensnél az összes helyi referencia vizsgálatához:

```bash
fprintd-verify -f any
```

Paraméter nélkül az első tárolt mintát választotta. Másik rekordhoz tartozó helyes felismerés ilyenkor sikertelen ellenőrzésnek tűnhetett.

A régi Windows-minták felületi ujjnevei nem feltétlenül felelnek meg a tényleges ujjaknak; több rekord ugyanahhoz az ujjhoz is tartozhat. A saját importálónk gépspecifikus, korábban igazolt rekordokra épült. Más gépen a rekordszámok mást jelentenek, ezért ezt nem publikáljuk általános telepítőként.

A végső javításhoz nem kellett gyári visszaállítás vagy a meglévő készülékbeli minták törlése. A próbához külön letiltottuk az automatikus újrapárosítást; ez a védelem nem része a csak azonosítókezelést tartalmazó mellékelt patchnek.

## Kezdeti bejelentkezés és KDE Wallet

A lemeztitkosítás feloldása és a kezdeti grafikus bejelentkezés jelszavas maradt. Így a Wallet automatikusan feloldódik; az ujjlenyomatot sudo és képernyőzár használja.

Az induló bejelentkezés elé beszúrt kísérleti PAM-szabályt visszavontuk, mert a felület csak a jelszómező elküldése után indította az olvasót, és a Wallet külön jelszót kért. Más disztribúció PAM-fájljait ne másold át ellenőrzés nélkül.

## Korlátok és továbblépés

A javítás helyi prototípus; a tartós referenciaformátumhoz és a migrációhoz fejlesztői felülvizsgálat szükséges. Frissítésekkel való hosszú távú kompatibilitását még nem igazoltuk. A kívánatos végállapot karbantartott upstream és disztribúciós csomag.

Az [angol jelentés](README.md) és a [reprodukálási jegyzet](REPRODUCING.md) fejlesztőknek ad további részleteket. A munka AI-segítséggel készült, a hardverpróbákat a laptop tulajdonosa végezte.

A nyers naplókat, ujjlenyomatképeket és sablonokat, privát kulcsokat, valódi SID-eket, sorozatszámokat és személyes fájlútvonalakat nem tesszük közzé.
