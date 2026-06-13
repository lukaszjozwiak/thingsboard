---
change_id: refactor-opportunities
title: Rank and plan which documented tech-debt problems to refactor, in what shape and order
status: planned
created: 2026-06-13
updated: 2026-06-13
archived_at: null
---

## Notes

Mamy analizę tego repozytorium, która dokumentuje dług techniczny i ryzyka
strukturalne: context/changes/os-encji/research.md. Ta zmiana odpowiada na
pytanie, które tamta analiza celowo zostawiła otwarte: KTÓRE z tych problemów
warto naprawić, w jakim docelowym kształcie i w jakiej kolejności.

Eksplorujemy każdy zapisany problem w kodzie i historii, a potem porządkujemy je
jako refactor opportunities. Zmiana przebiega etapami: eksploracja → decyzja
i plan → implementacja. Na etapie eksploracji nie dzieje się żaden refaktor i nie
zapada żadna decyzja.

Wynik eksploracji: research.md tej zmiany, zakończony rankingiem opcji
z trade-offami. Najpierw przeczytam raport; decyzja, co realizujemy, zapada na
etapie planowania, a refaktor rusza dopiero według przyjętego planu.
