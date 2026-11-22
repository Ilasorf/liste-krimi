[🇬🇧 English](README.md) · **🇸🇪 Svenska**

# Matlistor

**Automatisk rensning, etiketter och portionering av specialkoster — för köket i Kriminalvården.**

En enda statisk HTML-fil. All bearbetning sker i webbläsaren på den lokala datorn.
Inget backend, ingen databas, ingen serverdel.

`100 % klientsida` · `noll nätverk (offline-bygge)` · `inga personuppgifter` · `en enda HTML-fil`

---

## Vad den gör

Från dagens matbeställningslista (PDF) skapar den:

- **Rensad lista** — specialkostlistan städad enligt fasta regler
- **Etiketter** — en etikett per person (avdelning, kost, nummer)
- **Summa per kost** — antal av varje kosttyp
- **Fläsk-räkning** — per avdelning
- **Portionering** — hur mycket som ska lagas per avdelning

Den läser **inga namn** — endast inskrivningsnummer, avdelning och kost.

---

## Säkerhet & integritet

- Allt körs i webbläsaren — **uppgifterna lämnar aldrig datorn**.
- **Inga namn** läses eller sparas.
- **Ingen databas, inga cookies, ingen loggning, ingen trafik ut.**
- Bearbetningen finns bara i flyktigt minne; när fliken stängs töms den.

```
PDF-lista (lokal)
   -> Webbläsarens minne: pdf.js läser bara X-koordinater + regex
   -> Namnkolumnen slängs
   -> jsPDF skapar utskrifts-PDF:erna
   -> Stäng fliken = minnet töms
```

---

## Drift

| Alternativ | Noter |
|---|---|
| **GitHub Pages** (nuvarande) | Gratis, men externt. |
| **In-house** | Lägg den enda HTML-filen på er egen webbserver, eller öppna den lokalt från disk. Ingen databas, inget att drifta. |
| **Offline-bygge** | Alla bibliotek inbäddade i HTML:en (offline-versionen av `index.html`): **noll externa anrop**, fungerar i ett helt isolerat nät bakom brandvägg. Rekommenderas för intern drift. |

### Bibliotek

| Bibliotek | Version | Syfte | Standardbygget laddar från |
|---|---|---|---|
| pdf.js | 3.11.174 | läser PDF:en | cdnjs (eller inbäddat i offline-bygget) |
| jsPDF | 2.5.1 | skapar utskrifts-PDF:erna | cdnjs (eller inbäddat) |
| jspdf-autotable | 3.8.2 | tabeller i utskriften | cdnjs (eller inbäddat) |
| pdf.worker | 3.11.174 | pdf.js workertråd | cdnjs (eller inbäddat som Blob) |

> **CSP-not (intern server):** offline-bygget behöver `worker-src blob:`
> (pdf.js-workern körs från en Blob-URL) och inline-skript (`script-src` för de
> inbäddade biblioteken), eller så serveras filen utan restriktiv CSP.
> Öppnad direkt från disk (`file://`) gäller ingen CSP.

---

## Så läses uppgifterna

| Indata | Logik | Utdata |
|---|---|---|
| PDF-textdelar med X-koordinater | dela kolumner vid Specialkost-gränsen | vänster = insk.nr + avd; höger = kost; **namn slängs** |
| avd-token delad av pdf.js (mellanslag) | regex som tål mellanslag, ta sedan bort dem | normaliserad avd, t.ex. `12C2/12C2-11:1` |
| rå kost-text | `cleanSpec()` regler för borttag / behåll / flagga | rensad kost, eller rad flaggad för kontroll |
| kost men ingen avd (efter att avd börjat) | säkerhetskoll mot inledande blocket | rad flaggad `AVD SAKNAS` (röd) |
| justeringssträng för portioner | global regex: `=` fast, `+`/`-` delta | portionsantal per avd (fast får inga extra) |

---

## Nyckelkod

### Inläsning per position
Listan läses efter X-koordinat, så namnkolumnen (mellan nummer+avd och kost-kolumnen)
rörs aldrig. pdf.js kan dela upp avdelningen i bitar med mellanslag, så matchningen
tillåter mellanslag och tar bort dem efteråt.

```js
const leftText = sorted.filter(o => o.x < boundary)
                       .map(o => o.str).join(" ");
const im = leftText.match(/(\d{4}\/\d+)/);               // insk.nr
const am = leftText.match(                               // avd (tål mellanslag)
  /([A-Za-zÅÄÖåäö0-9]+\s*\/\s*[A-Za-zÅÄÖåäö0-9\s\-]*?:\s*\d+)/);
cur = {
  insk: im ? im[1] : "",
  avd:  am ? am[1].replace(/\s+/g, "") : "",             // ta bort mellanslag
  spec: rightText                                        // NAMNET läses aldrig
};
```

### Automatisk etikett-storlek
Avdelningsnamn med bokstäver (72RO, 12B2, 12C2 …) börjar två punkter mindre och
krymps tills de får plats; rena sifferavdelningar behåller full storlek.

```js
const special = /[A-Za-zÅÄÖåäö]/.test(it.ward || "");
let ws = special ? WARD - 2 : WARD;
while (ws > 8 && doc.getStringUnitWidth(it.ward || "") * ws * PT > WARD_MAXW) {
  ws -= 1;                       // krymp tills namnet får plats
}
```

### Säkerhetskontroll: avdelning saknas
En person med kost men utan avdelning — som dyker upp efter att avdelningarna börjat —
flaggas röd. Det inledande blocket utan avdelning (längst upp) lämnas som det är.

```js
const firstAvd = cleaned.findIndex(p => p.avd);
if (firstAvd >= 0) {
  cleaned.forEach((p, i) => {
    if (i > firstAvd && !p.avd && p.text !== "" && p.flag !== "review")
      p.flag = "noavd";          // kost men ingen avd -> flagga röd
  });
}
```

### Portionering: justeringar
Hittar **alla** justeringar på en rad oavsett avskiljare (mellanslag, komma, radbrytning).
`=` sätter ett fast antal (inga extra); `+`/`-` ändrar antalet och extra läggs till.

```js
const RX = /([Hh]?\s*\d+[A-Za-zÅÄÖåäö]*\d*)\s*(=|\+|-)\s*(\d+)/g;
let m;
while ((m = RX.exec(line))) {       // alla justeringar, inte bara första
  const k = normW(m[1]), op = m[2], n = +m[3];
  if (op === "=") map[k] = { type: "fixed", val: n };          // fast: inga extra
  else            map[k] = { type: "delta", val: (op === "-" ? -1 : 1) * n };
}
```

---

## Rensningsregler

- Tas alltid bort: `Ej Fläsk`, `Ramadan`, `Ungdom`.
- Tas bort endast när de står ensamma: `Nötter`, `Mandel`, `Laktos`, `Ej Blodmat`.
- Allergener och korta "Annat"-livsmedel får ett `-` framför.
- Instruktioner (t.ex. Diabetes, Hjärtproblem) förblir text.
- `Nötter`/`Mandel` blandat med annat → raden flaggas `KONTROLLERA`, orörd.
- **Rader raderas aldrig** — i tvivel behålls raden.

---

## Version & kontakt

Aktuell version: **v1.28**. Visas längst ner i verktyget och i portioneringens PDF.

Byggt och underhålls av **Simone Frosali**, Kök, Kriminalvården Västervik.
