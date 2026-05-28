# tofu

Zestaw wrapperów dla OpenTofu używanych z backendem GitLab Terraform State.

Skrypty są przeznaczone do uruchamiania w katalogu projektu GitLab. Na podstawie
`remote.origin.url` wykrywają pełną ścieżkę projektu, pobierają jego ID z GitLab
API i konfigurują backend HTTP GitLaba dla stanu OpenTofu.

## Skrypty

| Skrypt | Opis |
|---|---|
| `tofu-init` | Czyści lokalną inicjalizację OpenTofu i uruchamia `tofu init` z backendem GitLab Terraform State. |
| `tofu-plan` | Ładuje `tofu-init`, a następnie uruchamia `tofu plan`. |
| `tofu-apply` | Ładuje `tofu-init`, a następnie uruchamia `tofu apply -auto-approve`. |

## Wymagania

- `bash`
- `git`
- `curl`
- `jq`
- `tofu`
- token GitLab z dostępem do projektu
- repozytorium z ustawionym `remote.origin.url` w formacie SSH

Przykład obsługiwanego adresu `origin`:

```text
git@gitlab.com:group/project.git
```

Adres HTTPS nie jest obecnie obsługiwany przez parser ścieżki projektu.

## Instalacja

Skrypty znajdują się w katalogu `bin`:

```bash
./bin/tofu-init
```

Opcjonalnie można dodać katalog `bin` do `PATH`:

```bash
export PATH="$PWD/bin:$PATH"
tofu-init
```

## Konfiguracja

Wymagany jest token GitLab ustawiony w jednej z poniższych zmiennych:

| Zmienna | Opis |
|---|---|
| `CI_JOB_TOKEN` | Token używany w GitLab CI. |
| `GITLAB_TOKEN` | Token używany lokalnie. Jest brany pod uwagę, gdy `CI_JOB_TOKEN` nie jest ustawiony. |

Zmienne opcjonalne:

| Zmienna | Domyślnie | Opis |
|---|---|---|
| `CI_SERVER_URL` | `https://gitlab.com` | Adres GitLaba używany do zbudowania adresu backendu state. |
| `TF_STATE_NAME` | `production` | Nazwa stanu Terraform/OpenTofu w GitLab. |

## Użycie

Inicjalizacja lokalna:

```bash
GITLAB_TOKEN="glpat-..." ./bin/tofu-init
```

Inicjalizacja z inną nazwą stanu:

```bash
GITLAB_TOKEN="glpat-..." \
TF_STATE_NAME="dev" \
./bin/tofu-init
```

W GitLab CI skrypt może użyć `CI_JOB_TOKEN`:

```bash
CI_JOB_TOKEN="$CI_JOB_TOKEN" ./bin/tofu-init
```

Plan:

```bash
GITLAB_TOKEN="glpat-..." ./bin/tofu-plan
```

Apply:

```bash
GITLAB_TOKEN="glpat-..." ./bin/tofu-apply
```

Uwaga: `tofu-plan` i `tofu-apply` wykonują `source ./tofu-init`, więc w obecnej
implementacji oczekują pliku `tofu-init` w bieżącym katalogu procesu. Jeżeli
skrypty są uruchamiane bezpośrednio z `bin`, najpierw upewnij się, że ścieżka
uruchomienia pasuje do tej zależności.

## Co robi `tofu-init`

1. Sprawdza, czy ustawiono `CI_JOB_TOKEN` albo `GITLAB_TOKEN`.
2. Odczytuje `remote.origin.url`.
3. Usuwa prefix `git@host:` oraz suffix `.git`.
4. Pobiera ID projektu z GitLab API.
5. Wypisuje wykryte informacje o projekcie i stanie.
6. Usuwa lokalne pliki inicjalizacji OpenTofu.
7. Uruchamia `tofu init` z backendem GitLab Terraform State.

Konfigurowany backend ma postać:

```text
<CI_SERVER_URL>/api/v4/projects/<CI_PROJECT_ID>/terraform/state/<TF_STATE_NAME>
```

Blokowanie stanu używa endpointu:

```text
<CI_SERVER_URL>/api/v4/projects/<CI_PROJECT_ID>/terraform/state/<TF_STATE_NAME>/lock
```

## Skutki uboczne

Przed `tofu init` skrypt usuwa z bieżącego katalogu:

```text
.terraform
.terraform.lock.hcl*
```

Każde uruchomienie wymusza więc ponowną inicjalizację OpenTofu w katalogu, z
którego uruchomiono skrypt.

## Ważne uwagi

- Skrypt wypisuje wartość tokena w logu (`Token: ...`). Nie uruchamiaj go w
  miejscu, w którym logi są publiczne.
- Pobranie ID projektu jest obecnie wykonywane z `https://gitlab.com/api/v4`,
  niezależnie od wartości `CI_SERVER_URL`.
- Nazwa użytkownika backendu jest ustawiona na stałe jako `mrachuna`.
- `tofu-apply` uruchamia `tofu apply -auto-approve`, więc nie wymaga
  interaktywnego potwierdzenia.

## Przykładowy wynik

```text
🔧 Initializing OpenTofu with GitLab backend...

   Project url:       git@gitlab.com:group/project.git
   Project full path: group/project
   Project:           https://gitlab.com/-/projects/123456
   State:             production
   Token:             glpat-...

✅ OpenTofu initialized.
```

## Rozwiązywanie problemów

| Problem | Przyczyna | Rozwiązanie |
|---|---|---|
| `Brak tokena: ustaw GITLAB_TOKEN lub CI_JOB_TOKEN` | Nie ustawiono żadnej zmiennej z tokenem. | Ustaw `GITLAB_TOKEN` lokalnie albo `CI_JOB_TOKEN` w CI. |
| `CI_PROJECT_ID` ma wartość `null` | GitLab API nie zwróciło ID projektu. | Sprawdź token, uprawnienia i `remote.origin.url`. |
| `tofu init` zwraca błąd autoryzacji | Token nie ma dostępu do GitLab Terraform State. | Użyj tokena z odpowiednimi uprawnieniami do projektu. |
| Backend wskazuje zły projekt | Źle wykryta ścieżka projektu z `remote.origin.url`. | Sprawdź format adresu `origin`. |
| `./tofu-init: No such file or directory` przy `tofu-plan` albo `tofu-apply` | Wrapper źródłuje `./tofu-init` względem bieżącego katalogu. | Uruchom wrapper z katalogu, w którym dostępny jest `tofu-init`, albo popraw ścieżkę źródłowania w skrypcie. |
