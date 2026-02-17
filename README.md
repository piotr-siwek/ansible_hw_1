# Minimal LAMP Demo (Ansible)

Ten projekt wdraża środowisko LAMP (Linux + Apache + MySQL + PHP) na hoście z Ubuntu, z możliwością przełączania między trzema środowiskami: **development**, **staging** i **production**. Każde środowisko ma własną konfigurację zabezpieczeń, debugowania i wydajności.

## Kluczowe cechy projektu

- ✅ **Organizacja w role** – każdy komponent (Apache, PHP, MySQL) w osobnej roli
- ✅ **Wielkośrodowiskowość** – osobne inventory dla dev/staging/prod
- ✅ **Automatyczne ładowanie zmiennych** – Ansible sam wczytuje konfigurację środowiska
- ✅ **Idempotencja** – bezpieczne wielokrotne uruchamianie
- ✅ **Tagi** – selektywne uruchamianie ról
- ✅ **Automatyczne testy** – weryfikacja konfiguracji po wdrożeniu
- ✅ **Best practices** – zgodne ze standardami Ansible

## Struktura projektu

```
praca_domowa_1/
├── playbook.yaml           # Główny playbook (43 linie)
├── ansible.cfg             # Konfiguracja Ansible (cache, SSH, logi)
├── requirements.yaml       # Wymagane kolekcje z wersjami
├── .gitignore              # Pliki do ignorowania w Git
├── inventories/            # Osobne inventory dla każdego środowiska
│   ├── dev/
│   │   ├── hosts          # Definicja hostów dla development
│   │   └── group_vars/
│   │       └── web_server.yaml  # Zmienne środowiskowe dla dev
│   ├── staging/
│   │   ├── hosts          # Definicja hostów dla staging
│   │   └── group_vars/
│   │       └── web_server.yaml  # Zmienne środowiskowe dla staging
│   └── prod/
│       ├── hosts          # Definicja hostów dla production
│       └── group_vars/
│           └── web_server.yaml  # Zmienne środowiskowe dla prod
└── roles/                  # Role Ansible
    ├── base/              # Podstawowa konfiguracja systemu
    │   ├── defaults/main.yaml
    │   ├── tasks/main.yaml
    │   └── templates/motd.j2
    ├── web/               # Serwer Apache
    │   ├── defaults/main.yaml
    │   ├── tasks/main.yaml
    │   ├── handlers/main.yaml
    │   └── templates/vhost.conf.j2
    ├── php/               # Interpreter PHP
    │   ├── defaults/main.yaml
    │   ├── tasks/main.yaml
    │   ├── handlers/main.yaml
    │   └── templates/php.ini.j2
    ├── database/          # Baza danych MySQL
    │   ├── defaults/main.yaml
    │   ├── tasks/main.yaml
    │   ├── handlers/main.yaml
    │   └── templates/lamp.cnf.j2
    └── app/               # Aplikacja PHP
        ├── defaults/main.yaml
        ├── tasks/main.yaml
        ├── handlers/main.yaml
        └── templates/index.php.j2
```

### Pliki konfiguracyjne

#### `playbook.yaml`
Główny playbook (43 linie) zawiera:

**Sekcja `vars`:**
- `database_name` – nazwa bazy danych MySQL
- `database_user` – użytkownik bazy danych
- `database_password` – hasło do bazy (w produkcji powinno być w Vault)

**Sekcja `roles`:**
Lista ról wykonywanych w kolejności:
1. `base` – podstawowa konfiguracja systemu
2. `web` – instalacja i konfiguracja Apache
3. `php` – instalacja i konfiguracja PHP
4. `database` – instalacja i konfiguracja MySQL
5. `app` – wdrożenie aplikacji PHP

Każda rola ma przypisany tag (np. `tags: ["web"]`) do selektywnego uruchamiania.

**Sekcja `post_tasks`:**
Automatyczne testy sprawdzające:
- Czy tryb debug jest włączony tylko w dev
- Czy ustawienia PHP są zgodne z polityką środowiska
- Czy Apache nasłuchuje na oczekiwanym porcie

**Uwaga:** Playbook nie ma sekcji `become: true` na poziomie głównym - każdy task, który wymaga uprawnień root, ma własne `become: true`.

#### `inventories/`
Katalog z osobnymi inventory dla każdego środowiska.

Każde środowisko (`dev/`, `staging/`, `prod/`) ma:

**1. Plik `hosts`**
Definiuje hosty i parametry połączenia:
```ini
[web_server]
192.168.1.26 ansible_user=devops ansible_port=7655 ansible_password=devops ansible_become_password=devops
```

**Parametry połączenia:**
- `ansible_user` – użytkownik SSH (np. devops)
- `ansible_port` – port SSH (7655 zamiast standardowego 22)
- `ansible_password` – hasło SSH (w produkcji użyj Vault!)
- `ansible_become_password` – hasło sudo (w produkcji użyj Vault!)

**2. Katalog `group_vars/`**
Zawiera plik `web_server.yaml` ze zmiennymi specyficznymi dla środowiska.

**Jak to działa (mechanizm Ansible):**
1. Uruchamiasz: `ansible-playbook -i inventories/dev/hosts playbook.yaml`
2. Ansible czyta plik `inventories/dev/hosts`
3. Wykrywa grupę `[web_server]`
4. **Automatycznie** szuka i ładuje `inventories/dev/group_vars/web_server.yaml`
5. Wszystkie zmienne z tego pliku są dostępne w całym playbooku

**Korzyści tej struktury:**
- ✅ **Convention over configuration** – standardowa praktyka Ansible
- ✅ **Automatyczne ładowanie** – Ansible sam wczytuje zmienne bez dodatkowego kodu
- ✅ **Separacja środowisk** – każde środowisko ma własne hosty i zmienne
- ✅ **Prosty playbook** – tylko 43 linie
- ✅ **Bezpieczeństwo** – łatwiej zarządzać dostępem do różnych środowisk
- ✅ **Skalowalność** – łatwo dodać nowe środowisko (np. `test/`)

#### `requirements.yaml`
Wymaga kolekcji Ansible z określonymi wersjami:

```yaml
collections:
  - name: community.mysql
    version: ">=3.0.0"
  - name: community.general
    version: ">=8.0.0"
```

**Co zawierają:**
- `community.mysql` – moduły do zarządzania MySQL:
  - `mysql_db` – tworzenie baz danych
  - `mysql_user` – tworzenie użytkowników
  - `mysql_query` – wykonywanie zapytań SQL
- `community.general` – ogólne moduły:
  - `apache2_module` – włączanie modułów Apache (np. headers)

**Dlaczego wersje są ważne?**
- Zapewniają kompatybilność z używanymi modułami
- Unikają problemów z breaking changes
- Dokumentują minimalne wymagania projektu

**Instalacja:**
```bash
ansible-galaxy collection install -r requirements.yaml
```

#### `ansible.cfg`
Konfiguracja Ansible dla projektu.

**Sekcja `[defaults]`:**
- `roles_path = ./roles` – ścieżka do ról
- `host_key_checking = False` – wyłącza weryfikację kluczy SSH (wygoda w dev)
- `deprecation_warnings = False` – ukrywa ostrzeżenia o przestarzałych funkcjach
- `interpreter_python = auto_silent` – automatyczny wybór interpretera Python
- `force_color = True` – kolorowe wyjście
- `nocows = 1` – wyłącza ASCII art krowy ;)

**Sekcja `[defaults]` - wydajność:**
- `gathering = smart` – zbiera fakty tylko gdy potrzebne
- `fact_caching = jsonfile` – cache'uje fakty w plikach JSON
- `fact_caching_connection = /tmp/ansible_facts` – lokalizacja cache
- `fact_caching_timeout = 3600` – cache ważny przez 1h
- `forks = 5` – równoległe wykonywanie na 5 hostach
- `log_path = ./ansible.log` – ścieżka do logów

**Sekcja `[privilege_escalation]`:**
- `become = True` – domyślnie używaj sudo
- `become_method = sudo` – metoda eskalacji uprawnień
- `become_user = root` – eskaluj do użytkownika root

**Sekcja `[ssh_connection]`:**
- `pipelining = True` – przyspiesza wykonywanie (mniej połączeń SSH)
- `ssh_args` – optymalizacja połączeń SSH (ControlMaster)

**Korzyści:**
- ⚡ **Szybsze wykonywanie** – cache faktów + pipelining
- 📝 **Logi** – wszystko zapisywane w `ansible.log`
- 🔧 **Wygoda** – brak weryfikacji kluczy SSH w dev
- 📊 **Monitoring** – łatwe debugowanie dzięki logom

#### `.gitignore`
Zapobiega commitowaniu wrażliwych i tymczasowych plików do Git.

**Pliki Ansible:**
```
*.retry          # Pliki retry po nieudanym uruchomieniu
.vault_pass      # Hasło do Ansible Vault (KRYTYCZNE!)
*.log            # Logi z ansible.cfg
```

**Pliki Python:**
```
*.pyc            # Skompilowane pliki Python
__pycache__/     # Katalog cache Python
*.pyo, *.pyd     # Inne pliki Python
.Python
*.so             # Biblioteki współdzielone
```

**Pliki IDE:**
```
.vscode/         # Visual Studio Code
.idea/           # PyCharm/IntelliJ
*.swp, *.swo     # Vim
*~               # Pliki backup
```

**Pliki systemowe:**
```
.DS_Store        # macOS
Thumbs.db        # Windows
```

**Pliki tymczasowe:**
```
*.tmp
*.bak
```

**Dlaczego to jest ważne?**
- 🔒 **Bezpieczeństwo** – `.vault_pass` nie trafi do repozytorium
- 🧹 **Czystość** – brak śmieci w Git
- 👥 **Współpraca** – każdy może używać swojego IDE

#### Zmienne środowiskowe (`inventories/*/group_vars/web_server.yaml`)

Każdy plik definiuje zmienne specyficzne dla środowiska:

**Zmienne ogólne:**
- `env_name` – nazwa środowiska ("dev", "staging", "prod")
  - Wyświetlana na stronie aplikacji
  - Używana w testach post_tasks

**Zmienne aplikacji:**
- `app_debug: true/false` – tryb debugowania
  - `true` w dev – szczegółowe logi, stack traces
  - `false` w staging/prod – brak wrاżliwych informacji

**Zmienne serwera WWW:**
- `web_security_headers` – lista nagłówków HTTP
  - **dev:** tylko `X-Content-Type-Options`
  - **staging:** + `X-Frame-Options`, `X-XSS-Protection`
  - **prod:** + `Strict-Transport-Security` (HSTS)

**Zmienne PHP:**
- `php_display_errors: "On"/"Off"` – czy wyświetlać błędy
  - `"On"` w dev – widoczne błędy dla developerów
  - `"Off"` w staging/prod – ukryte błędy przed użytkownikami
  - **Uwaga:** wartości w cudzysłowach, żeby YAML nie konwertował na boolean
- `php_error_reporting` – poziom raportowania
  - `E_ALL` w dev – wszystkie błędy
  - `E_ALL & ~E_DEPRECATED & ~E_STRICT` w staging
  - `E_ALL & ~E_NOTICE & ~E_DEPRECATED & ~E_STRICT` w prod

**Zmienne bazy danych:**
- `database_bind_address` – adres nasłuchiwania MySQL
  - `0.0.0.0` w dev – dostęp zdalny (wygoda developmentu)
  - `127.0.0.1` w staging/prod – tylko localhost (bezpieczeństwo)

**Zmienne testów:**
- `assert_expected_port: 80` – oczekiwany port HTTP
  - Używane w post_tasks do weryfikacji konfiguracji

### Role Ansible

Każda rola ma strukturę:
```
rola/
├── defaults/main.yaml    # Domyślne wartości zmiennych
├── tasks/main.yaml       # Lista zadań do wykonania
├── handlers/main.yaml    # Handlery (np. restart usług)
└── templates/            # Szablony Jinja2 (.j2)
```

#### `roles/base`
**Cel:** Podstawowa konfiguracja systemu operacyjnego.
- Aktualizuje cache pakietów APT
- Instaluje podstawowe narzędzia (vim, curl, git, htop)
- Wdraża niestandardowy MOTD (Message of the Day)

#### `roles/web`
**Cel:** Instalacja i konfiguracja serwera Apache.
- Instaluje pakiet `apache2`
- Tworzy katalog DocumentRoot
- Wdraża VirtualHost z szablonu `vhost.conf.j2`
- Włącza moduł `headers` (do nagłówków bezpieczeństwa)
- Uruchamia i włącza usługę Apache

**Zmienne środowiskowe:**
- `web_security_headers` – lista nagłówków HTTP (więcej w prod niż w dev)

#### `roles/php`
**Cel:** Instalacja i konfiguracja interpretera PHP.
- Instaluje PHP 8.3 + moduł `php-mysql`
- Wdraża niestandardowy plik `php.ini` z ustawieniami środowiskowymi
- Restartuje Apache po zmianie konfiguracji

**Zmienne środowiskowe:**
- `php_display_errors` – `"On"` w dev, `"Off"` w staging/prod
- `php_error_reporting` – poziom raportowania błędów

#### `roles/database`
**Cel:** Instalacja i konfiguracja bazy danych MySQL.
- Instaluje `mysql-server` i `python3-pymysql` (wymagane przez moduły Ansible)
- Wdraża minimalny plik konfiguracyjny `lamp.cnf`
- Tworzy bazę danych, użytkownika i tabelę
- Wstawia przykładowe dane

**Zmienne środowiskowe:**
- `database_bind_address` – `0.0.0.0` w dev (dostęp zdalny), `127.0.0.1` w prod (tylko localhost)

#### `roles/app`
**Cel:** Wdrożenie aplikacji PHP.
- Tworzy katalog aplikacji
- Wdraża plik `index.php` z szablonu
- Aplikacja łączy się z MySQL i wyświetla dane z tabeli
- Pokazuje nazwę środowiska na stronie

**Zmienne środowiskowe:**
- `app_environment_label` – nazwa środowiska wyświetlana w UI

## Wymagania wstępne

1. **Kontroler Ansible:** wersja 2.15+ (projekt testowany z ansible-core 2.17)
2. **Host docelowy:** Ubuntu 22.04+ z użytkownikiem posiadającym uprawnienia sudo
3. **Połączenie SSH:** działające na porcie 7655 (lub innym zdefiniowanym w `inventory.ini`)
4. **Kolekcje Ansible:** zainstaluj wymagane kolekcje:

   ```bash
   ansible-galaxy collection install -r requirements.yaml
   ```

   **Dlaczego to jest potrzebne?**
   - Moduły `community.mysql.*` (do zarządzania bazą danych) nie są częścią ansible-core
   - Moduł `community.general.apache2_module` (do włączania modułów Apache) również wymaga tej kolekcji

## Uruchomienie

### Krok 1: Konfiguracja hostów

Edytuj pliki `inventories/*/hosts` i ustaw poprawne dane hostów dla każdego środowiska.

Przykład dla dev (`inventories/dev/hosts`):
```ini
[web_server]
192.168.1.26 ansible_user=devops ansible_port=7655 ansible_password=devops ansible_become_password=devops
```

**Uwaga:** W środowisku produkcyjnym powinieneś użyć innych hostów i zaszyfrować hasła Ansible Vault.

### Krok 2: Wybór środowiska

Playbook obsługuje trzy środowiska:

#### Development
```bash
ansible-playbook -i inventories/dev/hosts playbook.yaml
```

**Charakterystyka:**
- Debug włączony (`app_debug: true`)
- PHP wyświetla wszystkie błędy (`display_errors = On`)
- MySQL nasłuchuje na wszystkich interfejsach (`bind-address = 0.0.0.0`)
- Minimalne nagłówki bezpieczeństwa

#### Staging
```bash
ansible-playbook -i inventories/staging/hosts playbook.yaml
```

**Charakterystyka:**
- Debug wyłączony
- PHP ukrywa błędy (`display_errors = Off`)
- MySQL tylko localhost (`bind-address = 127.0.0.1`)
- Rozszerzone nagłówki bezpieczeństwa

#### Production
```bash
ansible-playbook -i inventories/prod/hosts playbook.yaml
```

**Charakterystyka:**
- Debug wyłączony
- PHP ukrywa błędy i ostrzeżenia
- MySQL tylko localhost
- Pełny zestaw nagłówków bezpieczeństwa (HSTS, X-Frame-Options: DENY, itp.)

### Krok 3: Weryfikacja

Po zakończeniu playbooka:

1. Otwórz przeglądarkę: `http://<IP_SERWERA>/`
2. Zobaczysz stronę z:
   - Nazwą środowiska (np. "Środowisko: **dev**")
   - Listą wiadomości z bazy danych MySQL

## Jak działa playbook?

### Automatyczne ładowanie zmiennych

Ansible automatycznie ładuje zmienne ze struktury inventory bez żadnego dodatkowego kodu:

```bash
ansible-playbook -i inventories/dev/hosts playbook.yaml
```

**Jak to działa krok po kroku:**

1. **Uruchamiasz playbook** z parametrem `-i inventories/dev/hosts`

2. **Ansible czyta plik hosts:**
   ```ini
   [web_server]
   192.168.1.26 ansible_user=devops ...
   ```

3. **Ansible wykrywa grupę `web_server`**

4. **Ansible automatycznie szuka i ładuje:**
   - `inventories/dev/group_vars/web_server.yaml`
   - `inventories/dev/group_vars/all.yaml` (jeśli istnieje)

5. **Ansible ładuje zmienne w kolejności priorytetów:**
   - Zmienne z `inventories/dev/group_vars/all.yaml` (jeśli istnieje)
   - Zmienne z `inventories/dev/group_vars/web_server.yaml`
   - Zmienne z `playbook.yaml` (sekcja `vars`) - najwyższy priorytet

6. **Wszystkie zmienne są dostępne** w rolach, taskach, templateach

**Dlaczego to działa tak dobrze?**
- ✅ **Convention over configuration** – standardowa praktyka Ansible
- ✅ **Zero dodatkowego kodu** – nie trzeba ręcznie ładować zmiennych
- ✅ **Prosty playbook** – tylko 43 linie
- ✅ **Niezawodność** – Ansible sam dba o kolejność ładowania
- ✅ **Bezpieczeństwo** – separacja inventory = separacja dostępu
- ✅ **Skalowalność** – łatwo dodać nowe środowisko

### Tagi (tags)

Każda rola ma przypisany tag, co pozwala na selektywne uruchamianie:

```bash
# Tylko rola web
ansible-playbook -i inventories/dev/hosts playbook.yaml --tags web

# Tylko role web i php
ansible-playbook -i inventories/dev/hosts playbook.yaml --tags web,php

# Wszystko oprócz testów
ansible-playbook -i inventories/dev/hosts playbook.yaml --skip-tags tests
```

**Dostępne tagi:**
- `base` – rola base (aktualizacja apt, instalacja pakietów, MOTD)
- `web` – rola web (instalacja Apache, vhost, moduły)
- `php` – rola php (instalacja PHP, konfiguracja php.ini)
- `database` – rola database (instalacja MySQL, tworzenie bazy/użytkownika/tabeli)
- `app` – rola app (wdrożenie aplikacji PHP)
- `tests` – testy w post_tasks (weryfikacja konfiguracji)

**Dlaczego używamy tagów?**
- Przyspieszenie developmentu (nie trzeba uruchamiać całego playbooka)
- Możliwość naprawy pojedynczej roli bez wpływu na resztę
- Łatwiejsze testowanie zmian

### Post-tasks (zadania końcowe) - Automatyczne testy

Wykonywane **po** wszystkich rolach, używają modułu `ansible.builtin.assert`.

**Test 1: Zgodność trybu debug ze środowiskiem**
```yaml
- (env_name == 'dev' and app_debug | bool) or
  (env_name != 'dev' and not (app_debug | bool))
```
Sprawdza:
- ✅ W dev: `app_debug` musi być `true`
- ✅ W staging/prod: `app_debug` musi być `false`

Cel: Zapobiega przypadkowemu włączeniu debugowania w produkcji.

**Test 2: Zgodność PHP display_errors z trybem debug**
```yaml
- ((app_debug | bool) and php_display_errors == 'On') or
  (not (app_debug | bool) and php_display_errors == 'Off')
```
Sprawdza:
- ✅ Jeśli debug włączony → PHP wyświetla błędy (`On`)
- ✅ Jeśli debug wyłączony → PHP ukrywa błędy (`Off`)

Cel: Zapobiega wyciekom wrażliwych informacji w produkcji.

**Test 3: Weryfikacja portu HTTP**
```yaml
- web_listen_port == assert_expected_port
```
Sprawdza:
- ✅ Apache nasłuchuje na oczekiwanym porcie (domyślnie 80)

Cel: Wykrywa konflikty portów lub błędy konfiguracji.

**Co się dzieje gdy test nie przechodzi?**
```
fatal: [192.168.1.26]: FAILED! => {
    "msg": "Tryb debug nie zgadza się z oczekiwaniami dla środowiska prod."
}
```
Playbook zatrzymuje się i wyświetla komunikat błędu.

**Dlaczego to jest ważne?**
- 🔒 **Bezpieczeństwo** – wykrywa niebezpieczne konfiguracje przed wdrożeniem
- ✅ **Jakość** – automatyczna weryfikacja zgodności z polityką
- 📝 **Dokumentacja** – testy opisują oczekiwane zachowanie systemu
- ⚡ **Szybkość** – błędy wykrywane od razu, nie po wdrożeniu

## Dostosowanie konfiguracji

### Zmiana portu HTTP

1. Edytuj `roles/web/defaults/main.yaml`:
   ```yaml
   web_listen_port: 8080
   ```
2. Zaktualizuj `group_vars/*/yaml`:
   ```yaml
   assert_expected_port: 8080
   ```

### Zmiana danych logowania do bazy

Edytuj `playbook.yaml`:
```yaml
vars:
  database_name: moja_baza
  database_user: moj_user
  database_password: bezpieczne_haslo
```

### Dodanie nowego środowiska

1. Utwórz katalog `inventories/test/`
2. Utwórz plik `inventories/test/hosts` z definicją hostów
3. Utwórz plik `inventories/test/group_vars/web_server.yaml` ze zmiennymi
4. Uruchom: `ansible-playbook -i inventories/test/hosts playbook.yaml`

### Zabezpieczenie haseł (Ansible Vault)

**⚠️ WAŻNE:** Obecna konfiguracja zawiera hasła w plaintext, co jest **niebezpieczne dla środowisk produkcyjnych**.

#### Krok 1: Utwórz zaszyfrowany plik z hasłami

```bash
# Utwórz plik z hasłami
cat > group_vars/all/vault.yaml << EOF
---
vault_ansible_password: devops
vault_ansible_become_password: devops
vault_database_password: lamp_password
EOF

# Zaszyfruj plik
ansible-vault encrypt group_vars/all/vault.yaml
```

#### Krok 2: Zaktualizuj inventory.ini

Zamień hasła na zmienne:
```ini
[web_server]
192.168.1.26 ansible_user=devops ansible_port=7655 ansible_password={{ vault_ansible_password }} ansible_become_password={{ vault_ansible_become_password }}
```

#### Krok 3: Zaktualizuj playbook.yaml

Zamień hasło bazy:
```yaml
vars:
  database_password: "{{ vault_database_password }}"
```

#### Krok 4: Uruchom playbook z Vault

```bash
# Z promptem o hasło
ansible-playbook -i inventories/prod/hosts playbook.yaml --ask-vault-pass

# Lub z plikiem hasła (bezpieczniej)
echo "twoje_haslo_vault" > .vault_pass
chmod 600 .vault_pass
ansible-playbook -i inventories/prod/hosts playbook.yaml --vault-password-file .vault_pass
```

**Uwaga:** Plik `.vault_pass` jest już w `.gitignore` i nie zostanie commitowany do repozytorium.

## Najczęstsze problemy

### 1. Apache nie startuje

**Objaw:** `Unable to restart service apache2`

**Przyczyna:** Port 80 zajęty przez inną usługę (np. nginx)

**Rozwiązanie:**
```bash
# Sprawdź, co zajmuje port 80
sudo netstat -tulpn | grep :80

# Zatrzymaj konkurencyjną usługę
sudo systemctl stop nginx
sudo systemctl disable nginx
```

### 2. Błąd składni Apache

**Objaw:** `Syntax error on line X of /etc/apache2/...`

**Przyczyna:** Błąd w szablonie VirtualHost lub brak włączonego modułu

**Rozwiązanie:**
```bash
# Sprawdź konfigurację Apache
sudo apache2ctl configtest

# Włącz wymagany moduł
sudo a2enmod headers
sudo systemctl restart apache2
```

### 3. Uszkodzone pakiety

**Objaw:** `dpkg returned an error code (1)`

**Przyczyna:** Przerwana wcześniejsza instalacja pakietów

**Rozwiązanie:**
```bash
sudo apt -f install
sudo dpkg --configure -a
```

### 4. Weryfikacja klucza SSH

**Objaw:** `Host key verification failed`

**Przyczyna:** Zmieniony klucz hosta w `~/.ssh/known_hosts`

**Rozwiązanie:**
```bash
ssh-keygen -R '[192.168.1.26]:7655'
```

### 5. Test assert nie przechodzi

**Objaw:** `Ustawienia PHP display_errors są niespójne ze środowiskiem`

**Przyczyna:** Niezgodność między zmiennymi środowiskowymi a faktyczną konfiguracją

**Rozwiązanie:**
- Sprawdź, czy plik `group_vars/<env>.yaml` ma poprawne wartości
- Upewnij się, że wartości `On`/`Off` są w cudzysłowach: `"On"`, `"Off"`
- Uruchom playbook ponownie, aby zaktualizować konfigurację

## Podgląd działania

![Podgląd aplikacji](web-page.png)

## Idempotencja

Playbook jest **idempotentny** – można go uruchamiać wielokrotnie bez skutków ubocznych:
- Pakiety są instalowane tylko jeśli ich brakuje
- Pliki konfiguracyjne są aktualizowane tylko jeśli się zmieniły
- Usługi są restartowane tylko jeśli konfiguracja się zmieniła (dzięki handlerom)
- Baza danych i użytkownik są tworzone tylko jeśli nie istnieją

**Przykład:**
```bash
# Pierwsze uruchomienie - wiele zmian
ansible-playbook -i inventories/dev/hosts playbook.yaml
# PLAY RECAP: changed=15

# Drugie uruchomienie - brak zmian
ansible-playbook -i inventories/dev/hosts playbook.yaml
# PLAY RECAP: changed=0
```

## Podsumowanie

Ten projekt demonstruje **best practices Ansible** dla środowisk wielośrodowiskowych:

### Funkcjonalności
- ✅ **Organizacja w role** – każdy komponent (Apache, PHP, MySQL) w osobnej roli
- ✅ **Wielośrodowiskowość** – osobne inventory dla dev/staging/prod
- ✅ **Automatyczne ładowanie zmiennych** – convention over configuration
- ✅ **Szablony Jinja2** – dynamiczna konfiguracja (vhost, php.ini, lamp.cnf)
- ✅ **Handlery** – inteligentne restartowanie usług tylko gdy potrzebne
- ✅ **Tagi** – selektywne uruchamianie ról
- ✅ **Automatyczne testy** – weryfikacja konfiguracji (assert)
- ✅ **Idempotencja** – bezpieczne wielokrotne uruchamianie
- ✅ **Bezpieczeństwo** – .gitignore, wersjonowanie kolekcji, przygotowanie pod Vault

### Architektura
- **Playbook:** 43 linie – prosty i czytelny
- **5 ról:** base, web, php, database, app
- **3 środowiska:** dev, staging, prod z osobnymi inventory
- **Automatyczne ładowanie:** Ansible sam wczytuje zmienne środowiskowe
- **Testy:** 3 asserty sprawdzające poprawność konfiguracji

### Różnice między środowiskami

| Cecha | Development | Staging | Production |
|-------|------------|---------|------------|
| Debug | ✅ Włączony | ❌ Wyłączony | ❌ Wyłączony |
| PHP errors | 👁️ Widoczne | 🙈 Ukryte | 🙈 Ukryte |
| MySQL bind | 🌍 0.0.0.0 | 🏠 127.0.0.1 | 🏠 127.0.0.1 |
| Security headers | 1 nagłówek | 3 nagłówki | 4 nagłówki + HSTS |
| Error reporting | E_ALL | E_ALL & ~E_DEPRECATED | E_ALL & ~E_NOTICE |

### Następne kroki (opcjonalne usprawnienia)

1. **Ansible Vault** – zaszyfruj hasła w inventory
2. **CI/CD** – automatyczne wdrażanie przez GitLab/GitHub Actions
3. **Monitoring** – dodaj rolę dla Prometheus/Grafana
4. **Backup** – automatyczne backupy bazy danych
5. **SSL/TLS** – certyfikaty Let's Encrypt przez Certbot
6. **Firewall** – rola dla ufw/iptables
7. **Logrotate** – rotacja logów Apache/MySQL
8. **Fail2ban** – ochrona przed brute-force
