# Wriggling Worms Race – Game Design Document

> **Stav:** pracovní draft v0.4 · **Prototyp:** `wriggling-worms-race.html` (Fáze 5 + refaktoring A)
> Související: [`wriggling-worms-refactoring.md`](wriggling-worms-refactoring.md) – technický plán refaktoringu
> Všechna čísla jsou v jednotkách **za sekundu** (simulace běží fixně 60 Hz) a odpovídají objektu `CONFIG` v prototypu.
> Legenda: ✅ = implementováno v prototypu · 🟡 = částečně · ❓ = otevřená otázka / k rozhodnutí

---

## 1. High concept

❓ *Jednovětý pitch – doplníme společně.*
Pracovní verze: **Lokální multiplayerový 2D závod žížal podzemím – prohrabávej se hlínou, žer kořeny a menší brouky, ať rosteš a zrychluješ, vyhni se kamenům a větším broukům a buď první v cíli.**

| | |
|---|---|
| Žánr | Arkádový závod (lokální multiplayer), snake-like růst, agar.io princip „větší žere menší“ |
| Platforma | Web (HTML5 Canvas), ovládání klávesnicí |
| Pohled | 2D z boku (řez půdou), horizontální scroll zleva doprava |
| Hráči | Lokální multiplayer na jedné klávesnici – prototyp 1–2 hráči (víc ❓) |
| Cílová skupina | ❓ |
| Délka session | ❓ |

## 2. Design pillars (návrh)

1. **Kopání mění svět** – každý pohyb trvale zanechává tunel, který ovlivňuje rychlost (tebe i ostatních).
2. **Růst = síla i riziko** – větší červ je rychlejší, ale delší tělo a větší hlava ztěžují manévrování.
3. **Organické, „vrtivé“ ovládání** – žížala se nevede přesně, vlní se; ovládání je jen zatáčení.
4. **Sdílený tunelový systém** – soupeři se navzájem ovlivňují cestami, které vykopou (následovat soupeře = rychlejší, ale „jeho“ kořeny už jsou sežrané).
5. **Velikost rozhoduje** – kdo je větší, žere; kdo je menší, utíká (platí pro brouky, ❓ i mezi hráči?).

## 3. Core loop

✅ Plazit se vpřed → hledat kořeny → žrát (zpomalení, ale růst) → být rychlejší → postupovat dál.
🟡 Žrát menší brouky / utíkat před většími (brouci zatím bez interakce).
🟡 **Cíl:** první hráč na cílové čáře na konci mapy vyhrává (cílová čára zatím neimplementována).

**Klíčové dilema:** zastavit se u kořene a růst (−75 % rychlosti) vs. pokračovat a nechat soupeře utéct. Větší brouci dál v mapě tlačí hráče k růstu, jinak se z kořisti stane potrava.

---

## 4. Současný stav prototypu (Fáze 5 + refaktoring A)

### 4.0 Spuštění a ovládání prototypu
| Akce | Klávesa / parametr |
|---|---|
| Zatáčení – 1 hráč | `←/→` nebo `A/D` |
| Zatáčení – 2 hráči | P1 (červený) `←/→` · P2 (modrý) `A/D` |
| Restart (stejná mapa) | `R` |
| Nová mapa (nový seed) | `N` |
| Počet hráčů (restart se stejným seedem) | `1` / `2` |
| URL parametry | `?seed=123&players=2` |

- Klávesy se čtou podle fyzické pozice (`e.code`) → nezávislé na CZ rozložení a CapsLocku.
- Soubor se spouští dvojklikem a běží offline (bez závislostí).

### 4.1 Svět a generování
| Parametr | Hodnota | Stav |
|---|---|---|
| Rozměr světa | 16 000 × 900 px (10 obrazovek) | ✅ |
| Viewport | 1 600 × 900 px | ✅ |
| Terén | Offscreen canvas; hlína `#3E2723`, vykopaná cesta `#D7CCC8` | ✅ |
| Generování | **Seedované** (PRNG `mulberry32`) – stejný seed = stejná mapa (kameny, kořeny, rozmístění a velikosti brouků) | ✅ |
| Nezávislost na počtu hráčů | Stejný seed dá stejnou mapu pro 1 i 2 hráče (bezpečná zóna kamenů kryje všechny starty) | ✅ |
| Determinismus průběhu | ❌ AI brouků používá `Math.random()` → mapa se opakuje, průběh hry ne (záměrně) | — |

### 4.2 Simulace
- ✅ Fixní simulační krok **60 Hz** s akumulátorem; vykreslování tak často, jak dovolí monitor → hra běží stejně rychle na 60 i 144 Hz.
- ✅ Ochrana po přepnutí záložky: max. 0,25 s simulace najednou.

### 4.3 Kamera
- ✅ Sdílená kamera **sleduje vedoucího červa** jen dopředu – posune se, když vedoucí přejde 75 % šířky obrazovky.
- ✅ Kamera nikdy necouvá; levý okraj obrazovky je pro **všechny žížaly** pevná zeď (odraz). **Brouky zadní zeď neomezuje** – okraj obrazovky volně přecházejí (viz 4.7).
- 🟡 **Důsledek pro MP:** zaostávajícího hráče tlačí zadní zeď → vzniká přirozený „rubber band“ v opačném směru (trest za zaostání, ne pomoc). ❓ Je to žádoucí?
- ❓ Alternativy: split-screen, kamera na střed mezi hráči, vyřazení hráče, který vypadne z obrazu.
- ❓ Možný budoucí režim s auto-scrollem (tlak času).

### 4.4 Hráč – žížala
**Ovládání:** zatáčení **3 rad/s**. Pohyb vpřed je automatický, nelze zastavit ani couvat.

**Pohyb:**
- ✅ Sinusové vlnění směru: 12 rad/s (≈ 1,9 Hz), amplituda 0,15 rad, při zatáčení stlumeno na 20 %.
- ✅ Tělo = historie pozic hlavy, oříznutá na cílovou délku; segmenty se ke konci zužují (50–100 % rádiusu).
- ✅ Kopání: hlava za sebou kreslí tunel o poloměru `radius + 2`.

**Rychlost (px/s):**
| Stav | Hodnota | Vliv |
|---|---|---|
| Hlína | 101 | – |
| Vykopaný tunel | až 169 | lineárně podle podílu „cesty“ v 5 vzorcích před hlavou (vějíř ±45°, 5 px před hlavou) |
| Kořen | 25 % základní (≈ 25 px/s) | když prostřední vzorek trefí kořen |
| Bonus za velikost | ×1 až ×1,5 | obě rychlosti škálují s rádiusem (na max. velikosti 152 / 253 px/s) |

**Růst (jen při žraní kořene):**
| Parametr | Start | Max | Přírůstek |
|---|---|---|---|
| Rádius | 9 | 25 | +1,69 px/s |
| Délka těla | 75 | 300 | +16,9 px/s |

→ Z nuly na maximum ≈ **9,5 s čistého žraní** kořenů (při rychlosti ~25 px/s).

**Kolize:**
- ✅ Vlastní tělo – hlava se odrazí (ochrana „krku“ 3,5× rádius, zásah pod 1,8× rádius).
- ✅ Kameny – odraz po normále elipsy.
- ✅ Horní/spodní okraj světa, zadní zeď kamery – odraz.
- ❌ Brouci – žádná interakce (červ jimi prochází).
- ❌ Druhý hráč – žádná kolize, červi se navzájem procházejí; ovlivňují se jen přes sdílené tunely.

### 4.5 Více hráčů 🟡
- ✅ 1–2 hráči, start na x = 100, ve 2 hráčích nad sebou s rozestupem 240 px.
- ✅ Barvy: P1 červený, P2 modrý.
- ✅ Sdílený terén – tunel jednoho hráče zrychluje druhého.
- ❌ Žádná pravidla mezi hráči (kolize, žraní, pořadí, cíl).

### 4.6 Prostředí
**Kameny** ✅
- Až 500 elips (rx 15–45, ry 10–30, náhodná rotace), max 20 % plochy světa, ne blíž než 150 px od startovních pozic.
- Neprůchozí pro červa i brouky. Nelze je prokopat.

**Kořeny** ✅
- 60 systémů, rostou ze stropu nebo ze dna (50/50), 20–50 segmentů, rádius 20–40 → zmenšuje se o 10 % na segment (konec pod 3 px).
- Zóna x ∈ 400 … 15 600.
- Pro červa: zpomalení + růst (= „jídlo“). Červ přes ně kreslí tunel, tj. kořen v terénu fakticky sežere.
- Pro brouky: neprůchozí překážka (klouzání podél).

### 4.7 NPC – brouci ✅
- 30 brouků, náhodně v x ∈ 500 … 16 000.
- **Velikost roste s postupem mapou:** rádius 4,5 → až 31,25 (= 125 % max. rádiusu červa) na konci světa.
- **Probuzení:** aktivují se, až když jsou < 300 px za pravým okrajem obrazovky.
- **AI:** náhodná chůze řízená časovačem – v průměru ≈ 3,1× za sekundu šance začít zatáčku (≈ 5 % na krok), zatáčka trvá 0,33–1,33 s rychlostí až ±3 rad/s. Během zatáčky nová nezačne.
- **Rychlost:** 101 px/s v hlíně → 168 px/s v tunelu (stejný princip vzorkování jako u červa, 3 vzorky).
- Také kopou tunely (rádius + 1) → mění terén pro hráče.
- **Pohyb a okraje:**
  - *Okraj obrazovky (kamery)* brouka neomezuje – volně ho přechází doleva i doprava a může zůstat za kamerou (zadní zeď platí jen pro žížaly).
  - *Levý a pravý okraj mapy* (x = 0 a x = 16 000) – odraz.
  - *Horní a dolní okraj mapy* – stočí se vodorovně a klouže podél.
  - Kameny a kořeny objíždí (klouzání po tečně).
- Animace: 3 páry nohou (15 rad/s), kusadla, oči.

### 4.8 UI / HUD
- ✅ Pro každého hráče: rádius, aktuální / max. rychlost v px/s.
- ✅ Seed, počet hráčů a nápověda kláves (vpravo nahoře).
- ❓ Ukazatel postupu (vzdálenost do cíle), pořadí, čas, minimapa?

### 4.9 Ladicí parametry (`CONFIG`)
Všechna herní čísla jsou na jednom místě na začátku skriptu, pojmenovaná a v jednotkách za sekundu – naladěné hodnoty z prototypu jdou přímo opsat do GDD a do finální hry. Struktura: `sim`, `view`, `world`, `colors`, `players`, `worm`, `bug`, `stones`, `roots`.
Plánováno (refaktoring B): živý ladicí panel s „Kopírovat CONFIG“, debug overlay (F1), záznam metrik z playtestu.

---

## 5. Pozorování z kódu (k diskusi)

1. **Hra nemá cíl ani konec** – na x = 16 000 se červ jen odrazí. Závod zatím nemá cíl, pořadí ani časomíru.
2. **Brouci nemají vliv na hráče** – rostou do velikosti větší než červ; role (menší = kořist, větší = hrozba) je rozhodnutá, ale neimplementovaná (Fáze 6).
3. **Sežrané kořeny zůstávají pro brouky překážkou** – kolize brouků používá `rootsData`, které se při žraní nemění; obrys kořene se také dál kreslí. → řeší refaktoring C1 s Fází 6.
4. **Rychlost červa a brouků je téměř shodná** (101/169 vs. 101/168 px/s) – záměr, nebo náhoda? U brouka-predátora to rozhoduje, zda se mu dá ujet.
5. **Zpomalení v kořeni je silné (25 %)** – dilema „růst vs. tempo“ dostane smysl s druhým hráčem a cílem; B3 (metriky) ho umožní změřit.
6. **Zadní zeď v MP trestá zaostávajícího** (viz 4.3) – nutno rozhodnout spolu s kamerou ve Fázi 8.
7. **Brouci za kamerou žijí dál** a nikdy nezmizí (drobnost výkonu). Díky volnému přechodu okraje obrazovky se brouk zpoza kamery může vrátit do obrazu – i zezadu za hráčem.
8. ✅ ~~Logika vázaná na framy~~, ✅ ~~bez seedu~~, ✅ ~~bez restartu~~, ✅ ~~sdílené ovládání~~ – vyřešeno refaktoringem A. Zbývající technický dluh (vzorkování `getImageData`, duplicitní kolize) je v plánu refaktoringu C.

---

## 6. Otevřené designové otázky

- ✅ ~~Soupeři v závodě~~ → **lokální multiplayer** (další režimy – auto-scroll, časovka – možná později).
- ✅ ~~Role brouků~~ → **podle velikosti:** menší = kořist (růst), větší = hrozba.
- ✅ ~~Struktura trati~~ → v prototypu **jedna mapa s cílovou čárou**; dlouhodobě zatím otevřené.
- ✅ ~~Počet hráčů a ovládání~~ → prototyp **2 hráči**: P1 šipky, P2 A/D. (Víc hráčů ❓)
- ✅ ~~Okraje pro brouky~~ → okraj obrazovky volně, okraj mapy vlevo/vpravo odraz, nahoře/dole klouzání.
- ❓ **Lose podmínka / trest:** co se stane, když tě větší brouk dostane? (smrt + respawn, zmenšení, zkrácení těla, omráčení?)
- ❓ **Kamera v MP:** sdílená s tlačící zadní zdí (současný stav), split-screen, nebo vyřazení zaostávajícího?
- ❓ **Interakce mezi hráči:** kolize těl, žraní menšího hráče, blokování tunelem?
- ❓ **Pravidlo „menší/větší“:** prosté porovnání rádiusu, nebo s tolerancí (např. musí být o 10 % větší)?
- ❓ **Catch-up mechanika:** pomoc pro zaostávajícího (tunely vedoucího ho zrychlují už teď – vyváží to tlačící zeď?).
- ❓ **Ekonomika růstu:** jen kořeny, nebo i jiné jídlo? Lze se zmenšit (ztráta při zásahu)?
- ❓ **Struktura obsahu:** jedna dlouhá trať, levely, procedurální endless?
- ❓ **Vizuální styl a tón:** humorný / roztomilý / realistický?

---

## 7. Roadmap prototypu

| Fáze | Obsah | Stav |
|---|---|---|
| 1 | Základní pohyb žížaly | ✅ |
| 2 | Kameny (překážky) | ✅ |
| 3 | Scrollující svět | ✅ |
| 4 | Kořeny – překážka, žraní a růst | ✅ |
| 5 | NPC brouci (pohyb, chování); oprava zrychlení v tunelech | ✅ |
| Refaktoring A | Fixní krok 60 Hz, `CONFIG` v jednotkách/s, seed + restart, vstup per hráč, kamera mimo červa | ✅ |
| Refaktoring B | Ladicí panel, debug overlay, záznam metrik | ⏳ další |
| 6 | Interakce brouk ↔ červ (žraní / hrozba podle velikosti) + C1 sežrané kořeny, C2 sjednocené kolize | |
| 7 | Cílová čára + win condition | |
| 8 | Lokální MP – pravidla mezi hráči, rozhodnutí o kameře (+ C3 tělo s pevnými rozestupy) | 🟡 základ hotový (2 hráči, vlastní ovládání) |

---

## 8. Changelog dokumentu
- **v0.1** (2026-10-07) – první draft sestavený z kódu prototypu Fáze 5.
- **v0.2** (2026-10-07) – doplněna vize: lokální multiplayer, brouci podle velikosti, mapa s cílovou čárou.
- **v0.3** (2026-10-07) – aktualizace po refaktoringu A: čísla v jednotkách/s, seed + restart, ovládání a režim 2 hráčů, sdílená kamera sledující vedoucího, změny AI brouků; roadmap doplněna o historii fází 1–5 a refaktoring.
- **v0.4** (2026-10-07) – upřesněno chování brouků na okrajích: okraj obrazovky volně (zadní zeď platí jen pro žížaly), levý/pravý okraj mapy odraz, horní/dolní klouzání.
