# NoteTown — zadávací dokument (design doc pro implementaci LLM)

> **Obsidian plugin.** Žijící středověké ASCII město, které vyrůstá z dat vaultu.
> Hra je **pasivní** — nehraje se aktivně, „hraje se" optimalizací a hlavně **designem města**.
> Toto je zadání pro vývojový LLM/agenta. Čti celé. Prózu drž v češtině, identifikátory/typy/soubory anglicky.

---

## 0. Severka (north star) — VIZUÁL JE NEJDŮLEŽITĚJŠÍ

Nejdůležitějším aspektem celé hry je **design a vizuál**. Vše ostatní (ekonomika, eventy, výkon) existuje proto, aby vyniklo nádherně ručně vytvořené ASCII město. Laťka kvality:

- **Stone Story RPG** — animované ASCII jako gold standard (živost, čitelnost, atmosféra).
- **Cogmind** — čistota ASCII UI, barvy, efekty.
- **Caves of Qud / Dwarf Fortress** — dlaždicová sídla, autotiling, hustota detailu.

Pravidlo: kdyby si někdo otevřel plugin a neviděl žádná čísla, **už jen pohled na město musí být potěšení**. Každý milník má vizuální quality-gate (viz §17). Nikdy neodbývej sprite „placeholderem", který by se dostal do release.

Estetika: **Catppuccin Mocha**, signature akcent **Mauve (`#cba6f7`)**. Paleta v §10.

---

## 1. Pilíře a non-goals

**Pilíře**
1. **Design města je hra.** Hráč přesouvá budovy, klade cesty, dekoruje. To je hlavní smyčka.
2. **Pasivní & nenápadné.** Žádné nucené klikání, žádná ztráta při ignoraci. Žádné aktivní game-loop výzvy.
3. **Pevně provázané s vaultem.** Město = vizualizace struktury vaultu. Změna vaultu → město reaguje.
4. **Město jako prohlížeč vaultu.** Klik na budovu/člověka = skok do poznámky. Serendipita nad vlastními poznámkami.
5. **Škáluje 50 ↔ 5000+ poznámek** a funguje i bez tagů/složek/linků (degraduje elegantně).

**Non-goals**
- Žádná „power" progrese. Zlato kupuje **jen kosmetiku**. Sandbox bez konce a bez výhry.
- Žádné chátrání/úpadek budov v čase (stáří = jen kosmetická patina, ne penalizace).
- Žádný zápis herního stavu do poznámek. Vault zůstává čistý.
- Mobil zatím ne (desktop-first; kód ať mobil neblokuje do budoucna).

---

## 2. Klíčový princip: vault → město (feature-agnostic)

Aby to bylo stejně hratelné při 50 i 5000 poznámkách a u člověka, co nepoužívá tagy/složky/linky, **nesmí nic záviset na jednom signálu**. Mapování:

| Prvek vaultu | Prvek města | Pozn. |
|---|---|---|
| Poznámka | Občan (žije v domě) | základní jednotka |
| `notesPerPerson` poznámek (default 1) | 1 občan | konfigurovatelné |
| `peoplePerHouse` občanů (default 4, rozsah 2–6) | 1 dům | konfigurovatelné |
| Složka (`file.parent`) | Čtvrť / sektor | fallback při ploché struktuře → 1 čtvrť, organické shluky |
| Hloubka zanoření složek | Vnitřní vs. okrajové umístění čtvrti | volitelný vliv |
| Link / backlink (`resolvedLinks`) | Cesta mezi souvisejícími budovami; **víc spojení = silnější/širší cesta** | |
| Silně linkovaná poznámka (hub/MOC) | **Landmark** — věž, tvrz, cechovní dům; větší, prominentní | důležitost = velikost/level |
| Tag (`getAllTags`) | Cech/korouhev/kosmetická varianta; akcent čtvrti | jen flavor, ne typ budovy |
| Velikost souboru (`file.stat.size`) | Level/patra budovy | **čti `stat.size`, NEčti obsah** (perf+soukromí) |
| Stáří (`file.stat.mtime`) | Materiál/éra (dřevo → kámen) | jen kosmetika, ne úpadek |
| Osiřelá poznámka (0 linků) | Samota za hradbami / chatrč | |
| Rozbitý link / chybějící asset | **Nájezdník** (viz §14) | |
| Počet domů | Spouští infrastrukturu (taverna, dřevorubec…) | jen počet + špetka náhody |

**Normalizace velikosti:** typy budov a infrastruktura se odvozují z **počtu domů**, ne z absolutního počtu poznámek; mapa roste se `sqrt(budov)`. Malý vault = úhledná vesnička, velký = opevněné velkoměsto, oba čitelné.

**Výběr signálu pro čtvrti:** primárně složky; když nejsou → nejbohatší dostupný signál (tagy), jinak jediná organická čtvrť. Nikdy hru nezablokuje absence funkce.

---

## 3. Herní smyčky

1. **Pasivní ekonomika** (§8) — budovy pomalu generují zlato do per-budovového stropu. Sebereš, když chceš.
2. **Design / plánování** (§7) — *hlavní smyčka*: přesouvání budov, kladení cest, nákup a rozmisťování dekorací za zlato.
3. **Průzkum vaultu** (§12) — pan/zoom/minimapa, hover = náhled poznámky, klik = otevření, „toulání" za náhodnými poznámkami.

Žádná smyčka nevyžaduje pozornost v reálném čase. Vše snese týden ignorace bez ztráty.

---

## 4. Datový model a identita

- **Veškerý herní stav** v `data.json` pluginu přes `this.loadData()/saveData()`. Nic do poznámek.
- **Identita budovy ↔ poznámka:** klíč = **cesta souboru**. Mapu `path → buildingId` drž v paměti i v `data.json`.
- **Eventy Obsidianu** (registruj v `onload`):
  - `vault.on('create')` → nový občan.
  - `vault.on('delete')` → odchod občana / zánik domu.
  - `vault.on('rename', (file, oldPath))` → **pokrývá přesun mezi složkami** (změna čtvrti) i přejmenování; aktualizuj klíč `oldPath → newPath`.
  - `vault.on('modify')` → přepočítej level (z `stat.size`), případně linky.
  - `metadataCache.on('resolved' | 'changed')` → aktualizace link grafu, tagů, frontmatteru.
- **Link graf:** `app.metadataCache.resolvedLinks` (`source → {target: count}`); backlinky inverzí nebo `getBacklinksForFile`.
- **Inkrementálnost:** každý event mění **jen dotčenou budovu/cestu**, nikdy neregeneruje celé město. Plná regenerace jen na první spuštění nebo na explicitní „Regenerate".
- **Cache layoutu:** ulož vypočítané pozice, ať je město stabilní mezi reloady.
- **Vyloučení:** nastavitelné ignorované složky/tagy (templates, attachments, daily prázdné…) → nezakládají občany.

### Datové struktury (orientačně, TypeScript)

```ts
interface BuildingState {
  id: string;
  notePaths: string[];        // 1+ rezidentů (poznámek) v domě
  kind: 'home'|'biz'|'hall'|'well'|'landmark';
  spriteKey: string;          // odkaz do sprite katalogu
  level: number;              // z velikosti/důležitosti
  district: string;           // složka (nebo odvozený signál)
  pos: {x:number;y:number};   // v tile souřadnicích
  locked: boolean;            // hráč zamkl pozici (vault už nepřesouvá)
  pendingGold: number;
  goldFullAt: number;         // timestamp dosažení stropu
}
interface DecorationState { id:string; key:string; pos:{x:number;y:number}; rot?:number; }
interface RoadEdit { tiles:[number,number][]; decorative:boolean; }
interface SaveData {
  version: number;
  seed: number;
  buildings: Record<string,BuildingState>;
  pathIndex: Record<string,string>;   // notePath → buildingId
  decorations: DecorationState[];
  roadEdits: RoadEdit[];
  settings: Settings;
}
```

---

## 5. Generace města (organický růst)

**Organický generátor je už hotový a ověřený v prototypu `notetown-prototype.html`** — radiální hlavní třídy z náměstí, organické větvení uliček, frontage zástavba **zevnitř ven** (centrum husté, okraje řidší), čtvrti v úhlových sektorech, náměstí s radnicí a kašnou, hrubá hradba s branami. Síť je při všech velikostech **100% 4-souvislá** (chodci se dostanou všude). **Použij tento algoritmus jako základ** a vylepši:

1. **Landmarky:** nejvíc linkované poznámky umísti jako prominentní stavby blízko náměstí.
2. **Hradba & brány:** od ~16 domů; brány tam, kde ven vychází hlavní třída.
3. **Priorita umístění (override systém):**
   - `locked` budova → pevná pozice; změna složky aktualizuje jen `district` meta, **ne** pozici.
   - odemčená → auto-organická pozice; na změnu čtvrti reaguje **přestěhováním** (animace, §7).
   - nový občan → nejdřív doplní podkapacitní dům ve své čtvrti; když není → nový dům na frontage okraji čtvrti s **animací stavby**, nebo (dle nastavení) do „to-place" zásobníku pro ruční umístění.
   - dekorace a dekorativní cesty → čistě hráčské, uložené po souřadnicích; auto-placement se jim vyhýbá.
4. **Stabilita:** přidání poznámky nesmí přeskládat město — jen lokální dostavba.

---

## 6. Přesouvání budov & plánovací režim (jádro hry)

Dedikovaný **Plánovač** (toggle v liště view). Toto je hlavní herní obsah — musí být příjemný a hutný.

- **Přesun budov:** drag & drop na volné dlaždice; náhled validity (kolize s cestou/budovou/dekorací zvýrazněna). Po umístění se budova automaticky `locked`.
- **Cesty:** „malování" dekorativních cest/dlažby; auto-routing mezi dvěma body; mazání. (Vault-link cesty jsou auto a vizuálně odlišené od ručních.)
- **Dekorace:** umisťování z katalogu (§9), rotace (pokud sprite podporuje orientace), mazání.
- **Zámek/odemčení** jednotlivých budov (řízení vztahu k vaultu).
- **Bulldoze** dekorací; **undo/redo**; snapping na tile mřížku.
- **„To-place" zásobník** nově vzniklých domů (volitelný režim plné kontroly).

Vše se ukládá jako overrides v `data.json` (po `buildingId` resp. souřadnicích). Auto-layout slouží jako výchozí stav, který si hráč postupně přetváří.

---

## 7. Animace „živého" města (vault eventy → chování)

| Event vaultu | Animace ve městě |
|---|---|
| create poznámka | staveniště → postupná **výstavba** domu |
| delete poznámka | dům **hoří** → sutiny → zmizí (rezidenti odejdou) |
| přesun (jiná složka), odemčený dům | rezidenti **rozeberou dům** a odstěhují → karavana → **postaví v nové čtvrti** |
| modify (level nahoru/dolů) | budova **upgraduje/downgraduje** vizuální tier (cosmetic) |
| přidán/odebrán link | cesta mezi budovami **vyroste / vybledne** |

Tyto animace dělají město „živým" — drž je pomalé a klidné (nestrhávat pozornost), ale řemeslně skvělé.

---

## 8. Ekonomika zlata (POMALÁ, pasivní)

- Každá budova akumuluje zlato po **per-budovový strop**, dosažený za ~8 h reálného času (default; v nastavení). **Generace musí být pomalá** — jde o ambientní pocit, ne grind.
- Po dosažení stropu budova **přestane generovat** (žádný trest za nesebrání).
- Sazby: dům < byznys < landmark. Vše v nastavení; default tak, aby plné město naplnilo „za noc".
- **Sběr (řešení konfliktu klik vs. otevření poznámky):**
  - **Levý klik na tělo budovy = otevře poznámku** (průzkum má přednost, §12).
  - **Plná budova zobrazí drobný coin-marker** nad střechou; **klik na coin-marker = sebere** zlato té budovy.
  - **Tlačítko „Vybrat vše"** v HUD sebere z celého města naráz.
- **Útrata:** výhradně **dekorace a kosmetické upgrady** (§9). Ceny škálované tak, aby pomalé zlato dávalo příjemný, neuspěchaný sink.

> Otevřená volba: zda zlato po stropu drobně „přetéká" do společné pokladny (aby šlo hrát čistě přes „Vybrat vše" bez klikání na budovy). Doporučení: ano, malý overflow do treasury s vlastním stropem — zachová pasivitu.

---

## 9. Dekorace (hlavní gold-sink)

Bohatý, rozšiřitelný katalog. Vše **jen kosmetika**. Příklady kategorií:
- **Zeleň:** okrasné stromy (sezónní), keře, květinové záhony, sady, živé ploty, vinice.
- **Voda:** kašny, jezírka, studny, kanály, mostky.
- **Mobiliář:** sochy, památníky, lavičky, lampy/luceny (svítí v noci!), korouhve, vlajky, tržní stánky, hranice dřeva.
- **Ohrady:** ploty, zídky, brány, živé ploty.
- **Povrchy:** dlažba, štěrk, mozaiky náměstí, různé cesty.
- **Building upgrady:** hezčí varianty střech/fasád/oken pro existující typy.

Každá dekorace = sprite v katalogu (§10) + cena. Luceny/okna musí participovat na noční vrstvě (svítí).

---

## 10. Sprite systém & VIZUÁL (nejvyšší priorita)

Všechny sprity — domy, **vyšší levely budov**, landmarky, dekorace, lidé, cesty, hradby, voda — musí být **opravdu pečlivě nadesignované a vzájemně konzistentní**. Žádné odbyté glyphy.

### 10.1 Formát spritu
Sprite = vrstvený, multi-tile, deklarativní (data-driven, ať je katalog rozšiřitelný):

```ts
interface Sprite {
  key: string;
  w: number; h: number;                  // footprint v tiles
  layers: SpriteLayer[];                 // pořadí: ground → base → roof → overlay → windows
  variants?: { season?: ...; era?: ...; level?: ... };
  anim?: { fps:number; frames:CellGrid[] }; // kouř, vlajky, voda, chodci
}
interface SpriteLayer { cells: Cell[]; }   // Cell = {dx,dy,glyph,role,colorToken}
```
- **role** řídí dynamické barvení: `roof|wall|window|door|water|leaf|trunk|stone|flag|lamp|sign|citywall|cobble…`.
- `window`/`lamp` se účastní noční vrstvy (svítí žlutě/teple).
- `leaf` se barví dle sezóny. `era` mění materiál (dřevo→kámen) dle stáří.

### 10.2 Autotiling (kritické pro hezký vzhled)
Cesty, hradby, ploty, voda, kanály **musí autotilovat** — různé glyphy pro rovný / roh / T / kříž / konec dle sousedů, aby spoje byly plynulé a ne „kostičkované". Implementuj bitmask (8- nebo 4-sousedství) → glyph tabulka per materiál.

### 10.3 Levely budov
Každý typ má **více ručně vytvořených vizuálních tierů** odrážejících důležitost/velikost:
- domy: chatrč → dům → patrový dům → tvrz/manor;
- byznys: stánek → krám → cechovní dům;
- landmark: věž / katedrála / hradní palác.
Tiery musí být zřetelně odlišné a všechny krásné.

### 10.4 Lidé
Sprity občanů s **idle + chůze frame(y)**, varianty (vesničan, kupec, stráž, dítě, mnich, nájezdník), ideálně směrové. V klidu drobné mikroanimace.

### 10.5 Vysoké rozlišení detailu (LOD-aware glyphy)
Pro detail při přiblížení použij **hybrid**: blokové/box-drawing glyphy na strukturu + **Braille (U+2800–U+28FF)** na jemné stínování a malé postavy (2×4 sub-buňky → vyšší efektivní rozlišení). Sprite katalog drž v několika detailních úrovních (viz LOD §11).

### 10.6 Paleta (Catppuccin Mocha, signature Mauve)
```
base #1e1e2e  mantle #181825  crust #11111b
text #cdd6f4  subtext #a6adc8 overlay #6c7086
surface0 #313244 surface1 #45475a surface2 #585b70
MAUVE #cba6f7 (signature)  blue #89b4fa  lavender #b4befe
green #a6e3a1  yellow #f9e2af  peach #fab387
red #f38ba8  maroon #eba0ac  pink #f5c2e7  teal #94e2d5  sky #89dceb
```
Mauve = akcent UI, radnice, výběr v plánovači, landmarky, highlight aktivní poznámky. Sezónní a den/noc palety odvozuj z Mocha (viz §13). Barvy vždy z tokenů, nikdy hardcode mimo paletu.

---

## 11. Render pipeline (obří města + detail)

Města mohou být **opravdu velká** → bez chytrého rendereru to nepojede. Požadavky: plynulý **pan**, **zoom**, **vysoké rozlišení** pro detail budov i lidí.

- **Canvas 2D** (WebGL jen kdyby bylo nutné). Žádný DOM per-tile.
- **Glyph cache:** předrenderuj glyphy do offscreen bitmap keyed `(glyph,colorToken,sizeTier)`; kresli `drawImage` místo `fillText`.
- **Chunked static layer:** svět rozděl na chunky (např. 32×32 tiles); každý chunk renderuj jednou do offscreen canvasu a **cachuj**; přegeneruj chunk jen při editaci / změně LOD tieru. Na pan jen **blit** viditelných chunků s offsetem. Toto je klíč k 60 fps na velkých mapách.
- **Dynamická vrstva** (každý frame, jen viewport): lidé, nájezdníci, coin-markery, animace (kouř, vlajky, voda), požáry/stavby.
- **Den/noc & sezóna:** globální tint overlay + samostatný pass „svítící okna/luceny" nad nočním overlayem (§13).
- **LOD (tři tiery dle px velikosti buňky):**
  - **far:** budova = 1 barevný blok; čtvrti jako barevné regiony; lidé skrytí/tečky → čte se jako mapa.
  - **mid:** zjednodušený sprite (~3×3), málo lidí.
  - **near:** plný detailní sprite + animace + Braille detail; lidé s chůzí.
- **Pan:** drag + setrvačnost. **Zoom:** kolečko + pinch/trackpad; preferuj **celé násobky** (integer scaling) pro ostrost glyphů. DPR-aware, crisp text.
- **Minimapa:** přehled celého města + skok kamerou (nutné pro orientaci ve velkém městě a pro průzkum, §12).
- **Font:** zvaž bundling ostrého monospace (JetBrains Mono / Cascadia) nebo bitmap-style; konzistentní advance.

---

## 12. Provázání s vaultem & průzkum (klíčová hodnota)

Město je **prostorový prohlížeč vaultu**. Skrz město musí jít zkoumat strukturu a klikat na náhodné poznámky.

- **Hover** budovy/člověka → tooltip: titulek, cesta, počet linků, mtime.
- **Levý klik budova** → otevři poznámku (`workspace.getLeaf().openFile`). Víc rezidentů → malý popover se seznamem k výběru.
- **Klik člověk** → otevři jeho poznámku; volba **„Sledovat"** (kamera ho doprovází při toulání).
- **Toulání / „Překvap mě":** kamera pomalu driftuje městem; klikni na cokoli, co projde → znovuobjevování vlastních poznámek.
- **Skok na poznámku:** vyhledávání → zvýrazní budovu, kamera k ní doletí (mauve highlight).
- **Struktura = město:** popisky čtvrtí = názvy složek; cesty viditelně spojují linkované poznámky; minimapa = alternativa graph-view. Hráč doslova vidí tvar svého vaultu jako město.
- **Obousměrnost:** otevřená aktivní poznámka v Obsidianu → její budova ve městě pulzuje/zvýrazní (volitelné, hezké).

---

## 13. Den/noc & roční období

- **Reálný systémový čas** (toggle): plynulý cyklus den → soumrak → noc → svítání. Noc = ztmavení + **svítící okna a luceny**.
- **Roční období** z reálného data: jaro (květy/růžová), léto (zeleň), podzim (peach/oranžová), zima (sníh/bledá, holé stromy). Mění listí, trávu, cesty, případně střechy.
- Demo/ladění: volitelné zrychlení času a ruční přepínač sezóny.
- Vše barevně odvozeno z Mocha palety.

---

## 14. Nájezdníci (z chyb vaultu)

- **Rozbité linky** a **osiřelé/chybějící assety** → nájezdníci přicházejí od okraje mapy k pokladně/budovám a **kradou zlato**, dokud nezmizí.
- Počet/síla úměrné počtu problémů ve vaultu → hra jemně **motivuje k údržbě** vaultu, ale neztrestá (jen kosmeticky/ekonomicky drobně).
- Vizuál: výrazný sprite, červený akcent; klidná, ne-stresující frekvence (lze vypnout v nastavení).

---

## 15. Nastavení (settings tab)

`notesPerPerson` · `peoplePerHouse` · `goldRatePerBuilding` · `goldCapHours` (default 8) · `treasuryOverflow` on/off · ignorované složky/tagy · zdroj čtvrtí (folder/tag/single) · `autoPlaceNewHouses` vs. to-place tray · animace (hustota / off) · `enableRaiders` · `followSystemClock` (den/noc, sezóny) · `performanceMode` (méně NPC, jednodušší LOD) · seed / Regenerate.

---

## 16. Výkon & škálování

- Cíl: plynulé i při **5000+ poznámkách** (~1250 domů). Prototyp generuje 3000 poznámek do mapy ~195×195 za ~220 ms a síť je 100% souvislá — drž tuto laťku.
- Generace mimo hlavní vlákno kde to jde; inkrementální eventy O(1)–O(log n).
- Chunk cache + LOD + glyph cache (§11) jsou povinné, ne volitelné.
- NPC jen **vzorek** (cap, default ~50) + LOD; nikdy nerenderuj všechny občany jako postavy.
- Lazy: chunky a sprite tiery generuj na vyžádání viewportem.

---

## 17. Architektura kódu & fáze

**Tech stack:** TypeScript, Obsidian Plugin API, bundling esbuild. `main.ts` = `Plugin` subclass; vlastní `ItemView` (canvas leaf) pro město; `PluginSettingTab`. Render čistě canvas. Stav přes `loadData/saveData`.

**Doporučené moduly:**
```
src/
  main.ts                 // plugin lifecycle, view & settings registrace
  vault/VaultModel.ts     // čtení vaultu, link graf, eventy → diff
  city/CityGenerator.ts   // organický layout (z prototypu) + landmarky + hradby
  city/CityState.ts       // budovy/dekorace/overrides, persistence
  city/Events.ts          // vault eventy → animace/diff
  economy/Gold.ts         // pomalá akumulace, stropy, sběr
  render/Renderer.ts      // chunk cache, LOD, pan/zoom, glyph cache
  render/Sprites.ts       // data-driven katalog + autotiling
  render/DayNightSeason.ts
  interact/PlannerMode.ts // přesun, cesty, dekorace, undo/redo
  interact/Explore.ts     // hover/klik→poznámka, follow, jump, minimapa
  ui/Hud.ts, ui/Minimap.ts, ui/SettingsTab.ts
```

**Fáze (každá s vizuálním quality-gate):**
- **M0 — Render & sprity základ:** canvas, glyph cache, chunk cache, pan/zoom, LOD, paleta. Statické krásné město z mock dat. *Gate: vypadá to skvěle bez jakékoli logiky.*
- **M1 — Vault integrace:** VaultModel, generace ze skutečného vaultu, čtvrti/linky/landmarky, hover+klik→poznámka, minimapa. *Gate: město = můj vault, dá se v něm orientovat.*
- **M2 — Živé eventy:** create/delete/rename/modify → výstavba/požár/stěhování/upgrade; den/noc + sezóny; lidé chodí. *Gate: město „žije".*
- **M3 — Plánovač (jádro):** přesun budov, cesty, dekorace, zámky, undo/redo, to-place tray. *Gate: navrhování města je radost.*
- **M4 — Ekonomika & nájezdníci:** pomalé zlato + stropy + sběr, gold-sink dekorace, nájezdníci z rozbitých linků, settings. *Gate: pasivní smyčka sedí, nestrhává pozornost.*

Sprite katalog se rozšiřuje průběžně všemi fázemi — **vizuál je trvalá priorita, ne závěrečný lak.**

---

## 18. Otevřené body (k rozhodnutí)

1. **Treasury overflow** po stropu budovy ano/ne (§8) — doporučení: ano, malý.
2. **Nové domy:** auto-place vs. to-place tray jako default (§5/§6).
3. **Tagy:** přesný kosmetický vliv (korouhve cechů? barva čtvrti?) — jen flavor.
4. **Orientace spritů:** podporovat rotace budov (hezčí frontage) vs. jen vzpřímené (jednodušší)?
5. **Font:** bundlovat vlastní monospace/bitmap font, nebo spolehnout na systémový?

---

*Konec zadání. Severka platí nade vším: nejdřív a nejvíc — aby město bylo nádherné.*
