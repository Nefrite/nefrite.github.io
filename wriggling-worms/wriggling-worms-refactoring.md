# Wriggling Worms Race – návrh refaktoringu před Fází 6

> Vychází z `wriggling-worms-race.html` (Fáze 5) a GDD v0.2. · verze 3
> **Stav:** ✅ kategorie A hotová (2026-10-07) · B, C čekají
> **Rámec:** prototyp slouží k ověření herní mechaniky. Finální hra se postaví na **výsledcích** (pravidla, naladěná čísla, poznatky z playtestů), ne na tomto kódu.
> **Omezení:** prototyp zůstává **jeden HTML soubor** bez build kroku a bez povinných závislostí, spustitelný dvojklikem.

## Kritérium priorit

Refaktoring se vyplatí, jen pokud:
1. **zajistí platnost výsledků** – čísla a pocity z prototypu musí jít přenést do finální hry,
2. **zrychlí iteraci** – ladění, opakování pokusu, porovnávání variant,
3. **odblokuje plánovanou mechaniku** (Fáze 6–8).

Čistota architektury, výkon „do zásoby“ a znovupoužitelnost kódu se **nepočítají**.

---

## Přehled kategorií

| Kat. | Význam | Položky |
|---|---|---|
| **A** ✅ | Nutné před Fází 6 – jinak jsou výsledky nepřenositelné nebo blokují další fáze | ✅ A1 časový krok · ✅ A2 `CONFIG` v jednotkách/s · ✅ A3 restart + seed · ✅ A4 vstup per hráč, kamera mimo červa |
| **B** | Nástroje pro iteraci – udělat hned po A, levné a vrací se každou session | B1 ladicí panel · B2 debug overlay · B3 záznam metrik z playtestu |
| **C** | Až to mechanika potřebuje – dělat v rámci té fáze, minimální variantou | C1 sežrané kořeny · C2 kolize sjednotit (Fáze 6) · C3 tělo s pevnými rozestupy (Fáze 8) · C4 výkon vzorkování terénu |
| **D** | V prototypu nedělat | rozdělení souborů / moduly, interpolace vykreslení, prostorový index, chunkování terénu, replay a testy determinismu, třídní hierarchie |

Pořadí: ~~A2 → A1 → A3 → A4~~ ✅ → **B1 → B2 → B3**, pak Fáze 6 (s C1, C2).

---

## A – Nutné před Fází 6 ✅ hotovo

**Co je v prototypu nově (shrnutí implementace):**
- Smyčka s fixním krokem 60 Hz (`frame()` → `step(SIM_DT)` → `render()`), ochrana `maxFrameTime` 0,25 s.
- `CONFIG` na začátku skriptu, vše v jednotkách za sekundu; HUD ukazuje rychlost v px/s (101 / 169 na startu).
- Seed z URL (`?seed=123`), jinak náhodný; zobrazen v HUD. Klávesy: **R** restart (stejná mapa), **N** nová mapa, **1 / 2** počet hráčů (restart se stejným seedem). URL parametr `?players=2`.
- Stejný seed = stejná mapa i při změně počtu hráčů (bezpečná zóna kamenů kryje starty pro 1 i 2 hráče).
- Režim 1 hráče: šipky i A/D; režim 2 hráčů: P1 šipky (červený), P2 A/D (modrý). Hráči spolu zatím nekolidují.
- Kamera `updateCamera(worms)` sleduje vedoucího červa; zadní zeď dostává červ jako parametr `minX`.
- Ověřeno: 60 kroků = 101,3 px v hlíně (Fáze 5: 1,6875 px × 60), stejný seed → identické kameny, šipka v MP otáčí jen P1, bez chyb v konzoli.

**Vědomé odchylky od Fáze 5:**
- Brouk během zatáčky už nemůže „přelosovat“ novou zatáčku (dřív umožňovala mrtvá proměnná `turnTimer`). Průměrná frekvence zatáčení je stejná, zatáčky jsou v průměru o něco delší.
- Brouk se odrazí od levého okraje světa (dřív mohl z mapy odejít).
- Kamera se posouvá po kroku všech červů → zadní zeď se uplatní s posunem o jeden krok (≈ 1,7 px, neznatelné).
- AI brouků (rozhodování o zatáčení) dál používá `Math.random()` – mapa je deterministická, průběh hry ne (plný determinismus je kat. D).

### A1. Časový krok místo framerate ✅

**Problém:** veškerá logika je „za frame“ (rychlosti, zatáčení, vlnění, růst, AI brouků, animace) a smyčka neměří čas. Na 120/144 Hz běží hra 2–2,4× rychleji. **Pro prototyp to je kritické:** naladěný pocit závisí na monitoru testera a čísla se nedají přenést do finální hry.

**Řešení:** fixní krok 60 Hz s akumulátorem (≈ 15 řádků, single-file bez problému):

```js
const SIM_DT = 1 / 60, MAX_FRAME = 0.25;
let acc = 0, last = performance.now();
function frame(now) {
  acc += Math.min((now - last) / 1000, MAX_FRAME); // ochrana po přepnutí záložky
  last = now;
  while (acc >= SIM_DT) { step(SIM_DT); acc -= SIM_DT; }
  render();
  requestAnimationFrame(frame);
}
requestAnimationFrame(frame);
```

- Fixní krok (ne `speed * dt` s proměnným dt): tunely bez mezer, žádné propadání kolizemi, pravděpodobnostní AI se nemusí přepočítávat.
- Při 60 Hz zůstává chování numericky stejné → **žádný rebalanc**.
- AI brouka přepsat na časovač (`state: 'walk'|'turn', timeLeft` v sekundách) – odstraní i mrtvou proměnnou `turnTimer`.
- Interpolaci vykreslení **nedělat** (kat. D); mírné cukání na 144 Hz výsledky neovlivní.

### A2. `CONFIG` v jednotkách za sekundu ✅

**Proč je to v A:** naladěná čísla **jsou** hlavní výstup prototypu. Musí být na jednom místě, pojmenovaná a v jednotkách nezávislých na snímkování – tak, aby šla přímo opsat do GDD a do finální hry.

```js
const CONFIG = {
  world:  { width: 16000, height: 900 },
  worm:   { radius: [9, 25], length: [75, 300], speed: { dirt: 101.25, tunnel: 168.75 },
            sizeSpeedBonus: 0.5, rootSlow: 0.25, turnRate: 3,
            wave: { freq: 12, amp: 0.15, turnDamp: 0.2 }, growth: { radius: 1.69, length: 16.9 } },
  bug:    { count: 30, radius: { min: 4.5, endFactor: 1.25 }, speed: { dirt: 100.8, tunnel: 168 },
            wakeDistance: 300, turn: { rate: 3.1, duration: [0.33, 1.33], speed: 3 } },
  stones: { count: 500, maxArea: 0.2, rx: [15, 45], ry: [10, 30], safeRadius: 150 },
  roots:  { systems: 60, radius: [20, 40], segments: [20, 50], shrink: 0.9 },
};
```

Přepočet dnešních hodnot (60 Hz):

| Hodnota | Za frame | Za sekundu |
|---|---|---|
| Červ – hlína / tunel | 1,6875 / 2,8125 px | 101,25 / 168,75 px/s |
| Červ – zatáčení | 0,05 rad | 3 rad/s |
| Červ – vlnění (fáze) | 0,2 rad | 12 rad/s (≈ 1,9 Hz) |
| Růst rádiusu / délky v kořeni | 0,028125 / 0,28125 | 1,69 / 16,9 px/s (max za ≈ 9,5 s) |
| Brouk – hlína / tunel | 1,68 / 2,8 px | 100,8 / 168 px/s |
| Brouk – šance začít zatáčet | 5 % | ≈ 3,1 /s |
| Brouk – zatáčka | 20–80 framů, ±0,05 rad | 0,33–1,33 s, ±3 rad/s |
| Brouk – animace nohou | 0,25 rad | 15 rad/s |

Bonus: HUD/GDD může zobrazovat „čas do maximální velikosti“ nebo „ztráta času na kořen“ – srozumitelnější než px/frame.

### A3. Restart + seed ✅

- **Restart (klávesa R)** – funkce `resetGame(seed)`: vyčistí terén, `rootsData`, kameny, brouky a červy a vygeneruje znovu. Dnes se svět vytvoří jednou při načtení a `rootsData` se nikdy nemaže. Fáze 7 (cíl → nová hra) bez toho nejde a každý pokus = reload stránky.
- **Seed** – PRNG `mulberry32` (5 řádků) místo `Math.random()` při generování světa. Seed v HUD a v URL (`?seed=1234`).
  Proč pro prototyp: **porovnávat varianty pravidel na stejné trati** (jinak je rozdíl ve výsledku jen šum mapy) a zopakovat konkrétní situaci z playtestu.
- AI brouků může klidně dál používat `Math.random()` – plný determinismus/replay je kat. D.

### A4. Vstup per hráč, kamera mimo červa ✅

**Proč teď:** Fáze 8 (lokální MP) je hlavní testovaná vlastnost hry a dnešní struktura ji blokuje:
- šipky i A/D ovládají **tentýž** `keys` → druhý hráč nemá klávesy,
- `Worm.update()` čte globální `keys` a **sám posouvá globální `cameraX`** a řeší zadní zeď.

Minimální změna:
- `keydown/keyup` plní `Set` stisknutých `e.code` (nezávislé na CZ rozložení a CapsLocku),
- `CONTROLS = [{ left: 'ArrowLeft', right: 'ArrowRight' }, { left: 'KeyA', right: 'KeyD' }]`,
- `worm.update(dt, turn)` dostane směr zatáčení jako parametr,
- `worms = []` místo `playerWorm` (zatím s jedním prvkem),
- `updateCamera(worms)` jako samostatná funkce po kroku červů (sdílená kamera vs. split-screen pak bude změna jen tady),
- `initStones()` nesmí sahat na `playerWorm` – bezpečná zóna se počítá ze startovní pozice.

---

## B – Nástroje pro iteraci

### B1. Ladicí panel
Živá změna hodnot z `CONFIG` bez reloadu (rychlosti, zpomalení v kořeni, růst, počet a velikost brouků). Dvě varianty, obě single-file:
- **lil-gui z CDN** (1 řádek `<script>`, ~10 řádků napojení) – nejrychlejší; bez internetu panel prostě chybí, hra běží,
- vlastní `<details>` s `<input type="range">` generovanými z `CONFIG` (~40 řádků, bez závislosti).

Přidat tlačítko **„Kopírovat CONFIG“** (JSON do schránky) → naladěné hodnoty jdou rovnou do GDD.
Volitelně: **presety** (např. `?preset=fastRoots`) pro A/B porovnání na stejném seedu.

### B2. Debug overlay (klávesa F1)
Vzorkovací body před hlavou (barva podle výsledku), kolizní kruhy, FPS / počet simulačních kroků, rádius brouků vs. červa (zelená = sežeru, červená = sežere mě – pro Fázi 6). Ukáže, proč se mechanika chová, jak se chová.

### B3. Záznam metrik z playtestu
Protože výstupem jsou poznatky, vyplatí se je měřit, ne jen odhadovat:
- čas závodu, čas strávený v kořenech, čas v tunelu vs. v hlíně,
- velikost v čase (křivka růstu), počet kolizí s kameny,
- od Fáze 6: sežraní / ztracení brouci.

Na konci kola zobrazit souhrn a tlačítko „Kopírovat výsledky“ (JSON/CSV do schránky, se seedem a aktuálním `CONFIG`). Odpoví to na otázky z GDD typu „vyplatí se zastavit u kořene?“ daty.

---

## C – Až to mechanika potřebuje

### C1. Sežrané kořeny (s Fází 6)
Kořeny zůstávají pro brouky překážkou a obrys se dál kreslí (GDD §5 bod 3). **Minimální řešení:** při kopání označit segmenty kořene, které hlava překryla, jako `eaten` (nebo jim zmenšit rádius) a vynechat je z kolize brouků a z overlaye.
Plnou logickou mřížku terénu dělat až, pokud bude mechanika potřebovat množství jídla v kořeni nebo typy půdy.

### C2. Sjednotit kolize (Fáze 6)
Dnes existují dvě různé `checkStoneCollision` (odraz u červa, klouzání u brouka). Při přidání kolize červ ↔ brouk vytáhnout společnou geometrii `collideEllipse(stone, x, y, r) → normála` a pravidlo velikosti jako jednu funkci `canEat(a, b)` (tolerance v `CONFIG`). Víc neabstrahovat.

### C3. Tělo s pevnými rozestupy (Fáze 8)
Tělo je historie pozic hlavy – počet bodů závisí na rychlosti. Po A1 to už nezávisí na monitoru, takže stačí řešit, až budou kolize červ ↔ červ (přidávat bod, až když je hlava dál než `spacing`).

### C4. Výkon vzorkování terénu
`getImageData` po pixelech (~95×/frame) je pomalé, ale dnes stačí. Řešit, až se při 2 hráčích nebo více broucích objeví propady FPS (B2 to ukáže). Nejlevnější cesta: logická mřížka `Uint8Array` paralelně s canvasem.

---

## D – V prototypu nedělat

| Co | Proč ne |
|---|---|
| Rozdělení do souborů / ES moduly | porušuje single-HTML; moduly navíc nejdou z `file://` |
| Interpolace vykreslení | kosmetika, neovlivní výsledky |
| Prostorový index (grid/bucket) | 500 kamenů zatím nevadí; viz C4 |
| Chunkování terénu | jen pro výrazně delší mapy |
| Replay, testy determinismu | seed + metriky (A3, B3) stačí |
| Hierarchie tříd / ECS | kód se zahodí; stačí funkce a jednoduché objekty |

---

## Organizace jednoho souboru

Aby se single HTML dal číst (i vkládat do chatu s AI), držet v `<script>` pevné pořadí sekcí s nadpisy v komentářích:

```
// ===== 1. CONFIG =====
// ===== 2. UTIL (rng, math) =====
// ===== 3. INPUT =====
// ===== 4. WORLD (terén, kameny, kořeny, generování) =====
// ===== 5. ENTITIES (Worm, Bug) =====
// ===== 6. GAME (reset, step, kamera, pravidla, metriky) =====
// ===== 7. RENDER + HUD + DEBUG =====
// ===== 8. LOOP + BOOT =====
```

✅ Drobnosti (provedeno v rámci A): odstranit Tailwind CDN (používá se jen na nadpis, hra má pak styly i offline), brouk bez ošetření levého okraje světa, `y` kamene s okrajem `rx` místo `ry`, mrtvé `Bug.speed` v konstruktoru, první volání `gameLoop()` bez `requestAnimationFrame`.

---

## Kontrola po kategorii A ✅

1. ✅ Na 60 Hz se hra chová stejně jako Fáze 5 (ověřeno: 101,3 px/s v hlíně; růst = původní hodnota × 60).
2. ⏳ Na 144 Hz / s omezením FPS v DevTools je rychlost stejná – plyne z konstrukce smyčky, ověřit ručně na monitoru s vyšší frekvencí.
3. ✅ Stejný seed → stejná mapa; R vygeneruje čistý svět.
4. ✅ Druhý `CONTROLS` záznam jde zapnout (klávesa 2 / `?players=2`) a ovládá vlastního červa (zatím bez dalších pravidel).
5. ✅ Soubor se otevře dvojklikem a funguje bez internetu (Tailwind CDN odstraněn). Na `file://` se URL se seedem nemusí sama aktualizovat – seed je vždy v HUD.
