# Porównanie algorytmów difficulty: BitMonero vs Monero vs Arqma

## Reakcja na nagły spadek hashrate’u w sieci

Gdy nagle znika duża część hashrate’u (np. wyłączenie puli, atak 51%, odejście minerów), czas bloków rośnie. Szybkość dostosowania difficulty decyduje, jak długo sieć „stoi” na zbyt wysokim difficulty i jak szybko wraca do normalnego tempa bloków.

---

## 1. BitMonero i Monero – ten sam algorytm

**Źródła:**  
- BitMonero: `src/cryptonote_basic/difficulty.cpp`, `src/cryptonote_config.h`  
- Monero: identyczne stałe (DIFFICULTY_WINDOW 720, DIFFICULTY_CUT 60, DIFFICULTY_TARGET 120 s).

**Algorytm:** klasyczny Cryptonote (bez LWMA):

- Bierze ostatnie **720** bloków (z lagiem 735).
- Sortuje po timestampach, odcina **60** z początku i **60** z końca (odporność na outliery).
- Liczy `time_span` i `total_work` na **środkowych ~600 blokach**.
- Wzór: `next_difficulty = (total_work * TARGET) / time_span` — uśrednianie **bez wag**.

**Stałe:**

| Stała | Wartość | Znaczenie |
|-------|---------|-----------|
| DIFFICULTY_WINDOW | 720 | liczba bloków w oknie |
| DIFFICULTY_CUT | 60 | odcięte z każdej strony po sortowaniu |
| Efektywne okno | ~600 bloków | ~20 h przy TARGET=120 s |
| DIFFICULTY_TARGET_V2 | 120 s | docelowy czas bloku |

**Reakcja na nagły spadek hashrate’u (np. −50%):**

- Czas bloku rośnie (np. z 120 s do ~240 s).
- Difficulty jest uśredniane po **~600 blokach**; wszystkie bloki liczą się tak samo.
- Aby difficulty spadło o ok. 50%, w oknie musi przeważać „wolna” historia: potrzeba **setek** nowych wolnych bloków.
- Szacunkowo: **kilkanaście–kilkadziesiąt godzin** (nawet ok. 24 h), zanim difficulty zbliży się do nowego poziomu.
- W tym okresie bloki wychodzą **znacznie rzadziej** niż docelowe 120 s (np. 240 s lub więcej).

**Podsumowanie:** Bardzo wolna reakcja, duża stabilność wobec manipulacji timestampami i krótkotrwałych skoków hashrate’u.

---

## 2. Arqma – LWMA i warianty (szybka reakcja)

**Źródła:**  
- Arqma: `src/cryptonote_basic/difficulty.cpp` (klasyczny `next_difficulty` + `next_difficulty_lwma`, LWMA-3, LWMA-4, **next_difficulty_v16**), `src/cryptonote_config.h`.

**Algorytmy:**

1. **next_difficulty()** – ten sam klasyczny Cryptonote co Monero/BitMonero (okno 720, cut 60), używany na starszych wersjach.
2. **next_difficulty_lwma()** – LWMA Zawy’ego; wersja z hardforku: N=17 (v9) lub N=30 (v2), T=120 lub 240 s.
3. **next_difficulty_lwma_3()** – N=90, T=120 s.
4. **next_difficulty_lwma_4()** – N=90, T=120 s (z temperingiem długich solvetime’ów).
5. **next_difficulty_v16()** – aktualny algorytm na HF v16: **N=90**, **T=120 s**, z limitem spadku (3×T), regułą 10% jump i zaokrągleniem.

**Stałe Arqma (v16 / LWMA):**

| Stała | Wartość | Znaczenie |
|-------|---------|-----------|
| DIFFICULTY_WINDOW_V16 | 90 | okno N bloków w LWMA |
| DIFFICULTY_TARGET_V16 | 120 s | docelowy czas bloku |
| FTL (future time limit) | 360 s | ograniczenie manipulacji timestampami |

**Ideą LWMA:** Ostatnie bloki mają **większą wagę** (liniowo: blok i ma wagę i). Długie czasy rozwiązania ostatnich bloków szybko podnoszą LWMA(solvetimes), więc **next_difficulty** od razu spada.

**Reakcja na nagły spadek hashrate’u (np. −50%):**

- Czas bloku rośnie (np. do ~240 s).
- Dzięki wagom **już pierwsze kilkanaście–kilkadziesiąt bloków** z długim solvetime’em wyraźnie obniża wyznaczaną difficulty.
- Szacunkowo: **znacząca korekta w 1–3 h**, pełne dostosowanie w **ok. 4–6 h** (rząd wielkości N× nowy czas bloku).
- Ograniczenia (np. „max drop 3×T”, „10% jump”) nieco spowalniają ekstremalne spadki, ale i tak reakcja jest **znacznie szybsza** niż w Monero/BitMonero.

**Podsumowanie:** Szybka reakcja na spadek hashrate’u; przy N=90 i T=120 s sieć wraca do normalnego tempa w ciągu kilku godzin zamiast kilkudziesięciu.

---

## 3. Porównanie w skrócie

| Cecha | BitMonero / Monero | Arqma (LWMA v16) |
|-------|--------------------|------------------|
| Algorytm | Cryptonote (sort + cut, bez wag) | LWMA (liniowe wagi, ostatnie bloki ważniejsze) |
| Okno | 720 bloków (~600 efektywnie) | 90 bloków |
| Docelowy czas bloku | 120 s | 120 s (v16) |
| Reakcja na duży spadek hashrate’u | **Wolna** – wiele godzin do ~1 dnia | **Szybka** – 1–6 h do wyraźnego dostosowania |
| Stabilność / ochrona przed atakami | Bardzo duża (długie okno, cut) | Wymaga m.in. ograniczenia FTL (Arqma: 360 s) |

---

## 4. Wnioski dla BitMonero

- **BitMonero = Monero** pod względem difficulty: ten sam, wolno reagujący algorytm.
- **Arqma** przy nagłym zniknięciu dużej ilości hashrate’u dostosuje difficulty **znacznie szybciej** dzięki LWMA (okno 90 bloków, wagi liniowe).
- Aby BitMonero reagował szybciej na takie sytuacje, trzeba by:
  - wprowadzić algorytm w stylu LWMA (np. na wzór Arqma v16) i odpowiednie stałe (N, T, FTL), oraz
  - aktywować go od określonego hardforku (jak w Arqma).

Referencje:  
- Monero: `github.com/monero-project/monero` (config i difficulty jak w BitMonero).  
- Arqma: `github.com/arqma/arqma` (`difficulty.cpp` – LWMA, LWMA-3, LWMA-4, v16; `cryptonote_config.h` – DIFFICULTY_WINDOW_V*, FTL).
