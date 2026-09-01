# SDLC Agent Team — 7 ról + orkiestrator + pętle skilli

Zespół SDLC do użycia w Claude Code, Codex i Devin. Kanoniczne role oraz skille
są w `.claude/`, a `.codex/` i `.devin/` zawierają natywne profile lub lekkie
odnośniki. Każdy agent ma **własną pętlę skilla** — wewnętrzny cykl przebiegów z
samokrytyką i kryteriami wyjścia. Nad nimi działa orkiestrator z pętlą zwrotną.

```
analysis -> design -> implementation -> testing -> review -> done
    ^          ^            ^             |         |
    +----------+---- feedback ------------+---------+
```

## Struktura

```
.claude/
  agents/                      <- 7 kanonicznych definicji ról
    metrics-analyst.md
    parallel-tester.md
    sdlc-analyst.md
    sdlc-architect.md
    sdlc-developer.md
    sdlc-tester.md
    sdlc-reviewer.md
  skills/
    sdlc-protocol/             <- wspólny kontrakt (stan, findings, routing)
    sdlc-orchestrator/         <- manager, /sdlc-orchestrator
    sdlc-metrics-loop/         <- opcjonalne definicje metryk
    sdlc-adversarial-loop/     <- opcjonalny drugi tor testów
    sdlc-analyst-loop/            <- 4 przebiegi
    sdlc-architect-loop/          <- 5 przebiegów
    sdlc-developer-loop/          <- 6 przebiegów
    sdlc-tester-loop/             <- 6 przebiegów
    sdlc-reviewer-loop/           <- 5 przebiegów
      SKILL.md + evals.json       <- każdy skill ma własne evals
evals/activation.json          <- trigger-rate suite dla całej rodziny
.codex/
  config.toml                  <- multi-agent + jawna rejestracja skilli
  agents/*.toml                <- 7 projektowych custom agents
  skills/*/SKILL.md            <- odnośniki do kanonicznych skilli
.devin/                        <- profile i odnośniki dla Devin
AGENTS.md                      <- instrukcje orkiestracji dla Codex
```

## Evals

Każdy skill ma `evals.json`: 4-5 scenariuszy, każdy z `query`,
`expected_behavior` (asercje PASS/FAIL) i `baseline` (co model robi bez skilla).
Scenariusze celują w hard rules, nie w happy path — np. dla `sdlc-tester-loop`
*"just make the failing test pass"* ma skończyć się odmową edycji produkcji.
To ta część, która w ogóle uzasadnia istnienie skilla.

`evals/activation.json` — 20 zapytań (10 fire / 10 no-fire) mierzących
trigger-rate całej siódemki naraz. Mierzone razem, bo prawdziwe ryzyko to
przestrzeliwanie między rodzeństwem, nie odpalanie w izolacji. Cel: ≥90%
trafień i zero false-fire na sześciu zapytaniach o zwykłą robotę.

## Instalacja

**Per projekt** (zalecane — zespół widzi to w repo):

```bash
cp -r sdlc-agents/.claude/agents/*        <projekt>/.claude/agents/
cp -r sdlc-agents/.claude/skills/sdlc-*   <projekt>/.claude/skills/
cp -r sdlc-agents/.codex                   <projekt>/          # Codex
cp    sdlc-agents/AGENTS.md                <projekt>/          # Codex
```

**Globalnie** (wszystkie projekty): to samo do `~/.claude/agents/` i
`~/.claude/skills/`.

Po instalacji zdecyduj, co z `.sdlc/` w projekcie docelowym — orkiestrator
zakłada go przy pierwszym uruchomieniu. Wersjonować, jeśli chcesz ślad audytowy
(kto co zgłosił, w której iteracji, jak rozwiązane). Do `.gitignore`, jeśli
traktujesz to jako stan roboczy. Domyślnie proponuję **wersjonować** —
`findings.jsonl` jest append-only właśnie po to, i przy sporze o to, czy coś
było przetestowane, jest jedynym dowodem.

`evals/` zostaje w tym repo, nie kopiuj go do projektu docelowego — dotyczy
samych skilli, nie kodu, który nimi budujesz.

Weryfikacja w Codex: profile w `.codex/agents/` obejmują 7 ról, a `/skills`
pokazuje 9 skilli `sdlc-*`. Projekt musi być oznaczony jako zaufany, aby Codex
załadował projektowy `.codex/config.toml`.

## Uruchomienie

Claude Code:

```text
/sdlc-orchestrator zaimplementuj kalkulator RPN z jednym wejściem evaluate(expr)
```

Codex:

```text
$sdlc-orchestrator zaimplementuj kalkulator RPN z jednym wejściem evaluate(expr)
```

Orkiestrator zakłada `.sdlc/`, odpala fazy po kolei jako subagentów, zbiera
findings, przewija do fazy z przyczyną, powtarza do czysta albo do wyczerpania
budżetu (`max_iterations`, default 6).

Pojedynczą rolę też można wywołać osobno, np. sam przegląd:
`Use the sdlc-reviewer agent on the current diff`.

## Pętle skilli — po jednej na agenta

Każda pętla to przebiegi `draft -> samokrytyka -> rewizja` z twardym limitem
**3 rewizji** i checklistą wyjścia. Limit jest po to, żeby agent nie mielił
w kółko na koszt budżetu iteracji orkiestratora.

| Agent | Przebiegi | Sedno samokrytyki |
|---|---|---|
| analyst | harvest → draft → testability challenge → revise | czy tester napisze z tego asercję? |
| architect | reuse survey → draft → simplification → trace → revise | który `R<n>` umrze, jak to usunę? |
| developer | context → triage → implement → **run** → self-review → revise | przyczyna czy objaw? |
| tester | derive → boundary sweep → write → **execute** → classify → report | czyj to root cause? |
| reviewer | scope → 6-osiowy sweep → contract check → **verify** → verdict | czy umiem podać konkretny failure scenario? |

Pogrubione przebiegi są nieusuwalne: developer i tester **naprawdę wykonują**
kod, reviewer **weryfikuje** każdy kandydat na finding zanim go zgłosi.

## Pętla zwrotna — routing po przyczynie

Finding niesie `target_phase` = faza, do której należy **naprawa**, nie ta, która
zauważyła problem.

| Objaw | target_phase |
|---|---|
| zachowanie, którego nikt nie wyspecyfikował | `analysis` |
| spec jest, struktura go nie unosi | `design` |
| spec i design OK, kod zły | `implementation` |
| brak przypadku testowego | `testing` |

Jeden defekt może wymagać **dwóch** findings (brakujące wymaganie + brakujący
kod). Wysyłanie wszystkiego do developera zamienia pętlę zwrotną w pętlę retry —
brakujące wymaganie wraca w następnej iteracji.

Severity: `blocker` zatrzymuje pipeline, `major` zawraca, `minor` tylko loguje.

## Stan na dysku

```
.sdlc/
  state.json        faza, iteracja, budżet, status
  requirements.md   analyst
  design.md         architect
  test-report.md    tester (z prawdziwym outputem runnera)
  review.md         reviewer (GO / NO-GO)
  findings.jsonl    append-only, audit trail
  work-log.md       wpis na każdy przebieg agenta
```

Findings nigdy się nie kasuje — przepisuje na `"resolved": true` z polem
`"resolution"`. Kod źródłowy idzie do drzewa projektu, nie do `.sdlc/`.

## Bezpieczniki

- Developer nie tyka testów. Tester nie tyka produkcji. Reviewer nie tyka nic.
- Tester nie raportuje wyniku, którego nie zaobserwował.
- Ten sam kod findingu w 3 iteracjach z rzędu → `blocked`, stop. Pętla nie
  zbiega, root cause jest źle zaadresowany.
- Orkiestrator nie robi żadnej fazy sam — to niszczy niezależność ocen.

## Codex

Codex korzysta natywnie z custom agents w `.codex/agents/`. Fazy uruchamia
sekwencyjnie, z wyjątkiem dwóch niezależnych testerów uruchamianych równolegle.
Tryb bez subagentów pozostaje wyłącznie fallbackiem dla hostów, które faktycznie
nie udostępniają delegowania.

## Referencyjna implementacja

Ten sam pipeline istnieje też jako deterministyczny program w Pythonie (stdlib,
`python main.py`) — pokazuje routing findings na sucho, bez LLM. **Nie jest
częścią tego repo**; leży obok, w katalogu `sdlc-orchestrator/` tej samej
przestrzeni roboczej. Jeśli klonujesz sam bundle, tego kodu nie dostaniesz i
niczego nie tracisz — agenci są kompletni bez niego.
