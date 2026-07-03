# PRD — Ivote: aplikacja głosowania DAO dla IPI

> **Status:** DRAFT (Fala 3)
> **Właściciel:** IPI DAO / dev crew
> **Powiązane zgłoszenia:** zamyka #2 (Fala 3), domyka #1 („undefined functionality"), roadmapa: `ipicoin/universal-independency-declaration#1`
> **Repozytorium:** `ipicoin/Ivote` (Next.js + TypeScript, bootstrap `create-cosmos-app`)

---

## 1. Wprowadzenie i cel

`Ivote` to webowa aplikacja (dApp) do zarządzania on-chain (governance) dla IPI DAO. Umożliwia posiadaczom tokena głosowanie nad propozycjami rządzącymi łańcuchem IPI oraz przeglądanie ich cyklu życia — od okresu depozytu, przez okres głosowania, aż do wyniku i egzekucji.

Aplikacja jest **interfejsem** do modułu `x/gov` Cosmos SDK. Nie wprowadza własnej logiki liczenia głosów — konsensus i tally są realizowane on-chain przez łańcuch IPI. Ivote odpowiada za: prezentację danych governance, zebranie decyzji użytkownika, zbudowanie i podpisanie transakcji oraz jej rozgłoszenie (broadcast) do sieci.

### Cel biznesowy
- Dać wspólnocie IPI DAO przejrzysty, samoobsługowy panel do sprawowania władzy nad protokołem.
- Zamknąć lukę „undefined functionality" (#1): jednoznacznie zdefiniować, **czym jest** Ivote i **jak** DAO głosuje.
- Zwiększyć partycypację (turnout) i obniżyć barierę wejścia dla nietechnicznych członków DAO.

## 2. Single Source of Truth (SSOT)

Parametry przyjęte jako wiążące dla całej Fali 3:

| Parametr | Wartość |
| --- | --- |
| Denominacja bazowa | `nipi` (nano-IPI; jednostka wyświetlana: `ipi`) |
| Warstwa governance | Cosmos SDK moduł `x/gov` (v1) |
| Waga głosu | staking (delegowane/zbondowane tokeny), zgodnie z tally `x/gov` |
| Wallet / podpis | `wallet-core.js` (Fala 2) — signer i zarządzanie kontem |
| RPC / broadcast + query | `ipi-rpc` (Fala 2) → endpoint `https://ipicoin.eu/rpc` |
| Konfiguracja łańcucha | `chainconfig` (chain-id, denom, prefix, endpointy) |

> Uwaga implementacyjna: dzisiejszy boilerplate używa `cosmos-kit` + `interchain-query` + `chain-registry`. Docelowo warstwa podpisu i RPC są odseparowane za adaptery `wallet-core.js` i `ipi-rpc`, aby aplikacja była niezależna od konkretnego dostawcy portfela. Migracja adapterów należy do implementacji (osobne zadanie po zatwierdzeniu PRD).

## 3. Użytkownicy (persony)

- **Członek DAO / delegator** — posiada zbondowany IPI, chce oddawać głosy i śledzić wyniki.
- **Walidator** — głosuje wagą własną i delegatorów; potrzebuje szybkiego podglądu aktywnych propozycji.
- **Obserwator (nie połączony)** — przegląda listę i wyniki propozycji bez portfela (read-only).
- **Autor propozycji** — tworzy propozycję i wnosi depozyt (funkcja `later`, patrz zakres).

## 4. Zakres

### 4.1 MVP (Fala 3)
1. Lista propozycji governance (aktywne na górze, potem pozostałe).
2. Szczegóły propozycji (tytuł, opis Markdown, typ, status, harmonogram, depozyt).
3. Oddanie głosu: `Yes` / `No` / `NoWithVeto` / `Abstain`.
4. Waga głosu = staking (prezentacja i zależność od tally on-chain).
5. Status i wyniki: pasek tally, kworum, progi, czas zakończenia.
6. Historia: oznaczenie „Voted" i pokazanie własnego wyboru dla propozycji, w których użytkownik głosował.
7. Read-only dla niepodłączonego portfela (lista + wyniki).

### 4.2 Later (poza Falą 3)
- Tworzenie propozycji z UI + wpłata depozytu (`MsgSubmitProposal`, `MsgDeposit`).
- Ważony głos rozdzielony (weighted vote, `MsgVoteWeighted`).
- Powiadomienia (koniec okresu głosowania, przekroczenie kworum).
- Delegacja/redelegacja stakingu z poziomu Ivote.
- Historia zmian parametrów governance i archiwum wykonanych propozycji.
- Wielojęzyczność UI.

### 4.3 Out of scope
- Własny mechanizm liczenia głosów lub konsensusu (to robi łańcuch przez `x/gov`).
- Custody środków / przechowywanie kluczy prywatnych (klucze zostają w `wallet-core.js`/portfelu).
- Operacje cross-chain / IBC governance.
- Zmiany w samym module `x/gov` łańcucha IPI (to warstwa chain, nie dApp).

## 5. Funkcje (szczegółowo)

### F1 — Lista propozycji
Pobranie propozycji przez `ipi-rpc` (query `cosmos.gov.v1.Proposals`). Sortowanie: najpierw `VOTING_PERIOD`, potem malejąco po `id`. Karta pokazuje: `#id`, tytuł, status (pending/passed/rejected), skrócone tally, `votingEndTime`, znacznik „Voted" jeśli użytkownik już głosował.

### F2 — Szczegóły propozycji
Modal/strona z: pełnym opisem (render Markdown), typem propozycji, statusem, `submitTime`/`votingStartTime`/`votingEndTime`, całkowitym depozytem, wynikiem tally i progami (kworum, próg, próg weta).

### F3 — Oddanie głosu
Dostępne tylko dla podłączonego portfela i propozycji w `VOTING_PERIOD`. Opcje mapowane na `x/gov` `VoteOption`:

| UI | VoteOption | wartość |
| --- | --- | --- |
| Yes | `VOTE_OPTION_YES` | 1 |
| Abstain | `VOTE_OPTION_ABSTAIN` | 2 |
| No | `VOTE_OPTION_NO` | 3 |
| NoWithVeto | `VOTE_OPTION_NO_WITH_VETO` | 4 |

Budowa `MsgVote { proposalId, voter, option }`, opłata w `nipi`, podpis przez `wallet-core.js`, broadcast przez `ipi-rpc`. Po sukcesie: toast + refetch danych.

### F4 — Waga głosu = staking
Aplikacja prezentuje, że siła głosu wynika ze zbondowanych tokenów (query `cosmos.staking.v1beta1.Pool` → `bondedTokens`). Faktyczne tally waży głosy stakiem po stronie łańcucha; brak stakingu ⇒ głos bez wagi (informacja w UI).

### F5 — Status i wyniki
`GovernanceVoteBreakdown` / pasek tally z rozbiciem Yes/No/NoWithVeto/Abstain, procent kworum względem `bondedTokens`, czas do końca. Kworum pobierane z `cosmos.gov.v1.Params(tallying)`.

### F6 — Historia / mój głos
Dla podłączonego adresu: query głosów użytkownika (`cosmos.gov.v1.Vote`) i oznaczenie propozycji, w których oddał głos, wraz z wybraną opcją.

## 6. Integracje

- **`x/gov` (Cosmos SDK v1):** źródło propozycji, tally, parametrów; typy komunikatów `MsgVote` (MVP), `MsgSubmitProposal`/`MsgDeposit`/`MsgVoteWeighted` (later).
- **`wallet-core.js` (Fala 2):** połączenie konta, pobranie adresu, podpis transakcji (signer). Odseparowany od UI adapterem.
- **`ipi-rpc` / RPC (`https://ipicoin.eu/rpc`):** zapytania query (proposals, votes, params, staking pool) oraz broadcast podpisanych transakcji.
- **`chainconfig`:** chain-id, `nipi`/`ipi` (denom + exponent), bech32 prefix, endpointy RPC/REST. Źródło prawdy dla parametrów łańcucha w aplikacji.

## 7. User stories + kryteria akceptacji

**US-1 — Przeglądanie propozycji (obserwator)**
Jako obserwator chcę widzieć listę propozycji bez łączenia portfela.
- AC: lista ładuje się z `ipi-rpc`; aktywne (`VOTING_PERIOD`) na górze; każda karta ma id, tytuł, status, tally, czas końca.

**US-2 — Szczegóły propozycji**
Jako członek DAO chcę zobaczyć pełny opis i parametry propozycji.
- AC: opis renderuje Markdown; widoczne status, harmonogram, kworum i progi.

**US-3 — Oddanie głosu**
Jako członek DAO chcę zagłosować Yes/No/NoWithVeto/Abstain.
- AC: głosowanie tylko przy podłączonym portfelu i statusie `VOTING_PERIOD`; wybór mapuje się na poprawny `VoteOption`; transakcja podpisana `wallet-core.js` i rozgłoszona `ipi-rpc`; po sukcesie toast + odświeżenie; przy błędzie czytelny komunikat.

**US-4 — Podgląd wagi głosu**
Jako delegator chcę rozumieć, że mój głos waży stakiem.
- AC: UI pokazuje zależność od stakingu; brak stakingu jest komunikowany.

**US-5 — Wyniki i kworum**
Jako członek DAO chcę widzieć bieżące tally i postęp kworum.
- AC: pasek Yes/No/NoWithVeto/Abstain; procent kworum liczony względem `bondedTokens`; czas do zakończenia.

**US-6 — Mój głos / historia**
Jako głosujący chcę widzieć, gdzie już oddałem głos i jaki.
- AC: propozycje z moim głosem mają znacznik „Voted" i pokazują wybraną opcję.

## 8. Ekrany (opis)

1. **Home / Lista propozycji** — nagłówek, przełącznik motywu, connect wallet, lista kart propozycji; stan „połącz portfel" dla read-only akcji głosowania; spinner podczas ładowania.
2. **Modal szczegółów propozycji** — tytuł `#id + nazwa`, opis Markdown, metadane, sekcja wyników (breakdown + kworum), grupa radio z 4 opcjami głosu + przycisk „Vote" (aktywny warunkowo).
3. **Stan portfela** — komponenty connect/user/warning (sieć, brak portfela, zły chain-id).
4. **Toasty** — sukces/błąd transakcji.

## 9. Model danych (logiczny)

- **Proposal:** `id`, `title`, `summary/description`, `type`, `status` (`UNSPECIFIED|DEPOSIT_PERIOD|VOTING_PERIOD|PASSED|REJECTED|FAILED`), `submitTime`, `votingStartTime`, `votingEndTime`, `totalDeposit[]`, `finalTallyResult`.
- **TallyResult:** `yesCount`, `noCount`, `noWithVetoCount`, `abstainCount` (wagi w `nipi`).
- **Vote:** `proposalId`, `voter`, `options[] { option, weight }`.
- **TallyParams:** `quorum`, `threshold`, `vetoThreshold`.
- **StakingPool:** `bondedTokens`, `notBondedTokens`.
- **VoteOption enum:** 0 UNSPECIFIED, 1 YES, 2 ABSTAIN, 3 NO, 4 NO_WITH_VETO.
- **ChainConfig:** `chainId`, `denom=nipi`, `displayDenom=ipi`, `exponent`, `bech32Prefix`, `rpc=https://ipicoin.eu/rpc`.

## 10. Wymagania niefunkcjonalne

- **Bezpieczeństwo:** klucze prywatne nigdy nie opuszczają portfela; aplikacja tylko buduje i wysyła podpisane transakcje.
- **Niezawodność:** obsługa błędów RPC/tx (timeout, out-of-gas, brak środków na fee w `nipi`) z czytelnym komunikatem.
- **Wydajność:** cache zapytań query (stale-time), unikanie zbędnych refetchów.
- **Dostępność / UX:** wsparcie trybu jasny/ciemny; stany loading/empty/error.
- **Zgodność:** typy z `interchain-query` / Cosmos SDK v1; TypeScript strict.

## 11. Zamyka #1 (undefined functionality)

Zgłoszenie #1 sygnalizuje, że repozytoria ekosystemu IPI (w tym `Ivote`) zostały utworzone bez opisu przeznaczenia. Niniejszy PRD domyka tę lukę dla `Ivote`:

- **Czym jest Ivote:** dApp — interfejs governance IPI DAO do przeglądania propozycji i głosowania on-chain przez moduł `x/gov`.
- **Jak DAO głosuje:** posiadacze zbondowanego IPI oddają głos `Yes/No/NoWithVeto/Abstain` na propozycje w okresie głosowania; waga = staking; tally, kworum i progi rozstrzyga łańcuch.
- **Jakie propozycje gov:** standardowe propozycje `x/gov` (m.in. tekstowe i zmiany parametrów), z pełnym cyklem: depozyt → głosowanie → wynik → egzekucja on-chain.
- **Cykl życia propozycji:** `DEPOSIT_PERIOD → VOTING_PERIOD → PASSED/REJECTED/FAILED`, z kworum i progami z `TallyParams`.
- **Rola w ekosystemie:** Ivote konsumuje `wallet-core` (podpis) i `ipi-rpc` (query/broadcast); jest warstwą prezentacji, nie zmienia logiki chain.

Po zatwierdzeniu tego PRD zgłoszenie #1 można zamknąć w części dotyczącej `Ivote` (przeznaczenie zdefiniowane), a implementacja realizuje pozostałe kryteria akceptacji Fali 3 (#2).

## 12. Otwarte kwestie

- Ostateczne wartości kworum/progów/vetoThreshold i długości okresów po stronie `chainconfig` łańcucha IPI.
- Harmonogram migracji z `cosmos-kit`/`interchain-query` na adaptery `wallet-core.js` + `ipi-rpc`.
- Czy tworzenie propozycji z UI wchodzi do kolejnej fali (Later) czy pozostaje CLI-only na start.
