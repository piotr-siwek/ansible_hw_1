# Minimal LAMP Demo (Ansible)

Ten projekt wdraża środowisko LAMP (Linux + Apache + MySQL + PHP) na hoście z Ubuntu, z możliwością przełączania między trzema środowiskami: **development**, **staging** i **production**. Każde środowisko ma własną konfigurację zabezpieczeń, debugowania i wydajności.

## Struktura projektu

```
praca_domowa_1/
├── playbook.yaml           # Główny playbook z logiką środowisk
├── inventory.ini           # Definicja hostów i grup
├── requirements.yaml       # Wymagane kolekcje Ansible
├── group_vars/             # Zmienne środowiskowe
│   ├── dev.yaml           # Konfiguracja development
│   ├── staging.yaml       # Konfiguracja staging
│   └── prod.yaml          # Konfiguracja production
└── roles/                  # Role Ansible
    ├── base/              # Podstawowa konfiguracja systemu
    ├── web/               # Serwer Apache
    ├── php/               # Interpreter PHP
    ├── database/          # Baza danych MySQL
    └── app/               # Aplikacja PHP
```

### Pliki konfiguracyjne

#### `playbook.yaml`
Główny playbook zawiera:
- **vars** – zmienne globalne (dane logowania do bazy)
- **pre_tasks** – zadania wykonywane PRZED rolami (walidacja środowiska, ładowanie zmiennych)
- **roles** – lista ról do wykonania w kolejności
- **post_tasks** – zadania wykonywane PO rolach (testy konfiguracji)

#### `inventory.ini`
Definiuje grupy hostów i parametry połączenia:
- `ansible_user` – użytkownik SSH
- `ansible_port` – port SSH (7655)
- `ansible_password` – hasło SSH
- `ansible_become_password` – hasło sudo

#### `requirements.yaml`
Wymaga kolekcji:
- `community.mysql` – moduły do zarządzania MySQL
- `community.general` – moduł `apache2_module` do włączania modułów Apache

#### `group_vars/`
Katalog ze zmiennymi środowiskowymi. Każdy plik (`dev.yaml`, `staging.yaml`, `prod.yaml`) definiuje:
- `env_name` – nazwa środowiska (wyświetlana w aplikacji)
- `app_debug` – czy włączyć tryb debugowania
- `web_security_headers` – lista nagłówków bezpieczeństwa HTTP
- `php_display_errors` – czy PHP ma wyświetlać błędy (`"On"` lub `"Off"`)
- `php_error_reporting` – poziom raportowania błędów PHP
- `database_bind_address` – adres nasłuchiwania MySQL (`0.0.0.0` dla dev, `127.0.0.1` dla prod)
- `assert_expected_port` – oczekiwany port HTTP (do testów)

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

### Krok 1: Konfiguracja inventory

Edytuj `inventory.ini` i ustaw poprawne dane hosta:

```ini
[web_server]
192.168.1.26 ansible_user=devops ansible_port=7655 ansible_password=devops ansible_become_password=devops
```

### Krok 2: Wybór środowiska

Playbook obsługuje trzy środowiska:

#### Development (domyślne)
```bash
ansible-playbook -i inventory.ini -e "env=dev" playbook.yaml
```

**Charakterystyka:**
- Debug włączony (`app_debug: true`)
- PHP wyświetla wszystkie błędy (`display_errors = On`)
- MySQL nasłuchuje na wszystkich interfejsach (`bind-address = 0.0.0.0`)
- Minimalne nagłówki bezpieczeństwa

#### Staging
```bash
ansible-playbook -i inventory.ini -e "env=staging" playbook.yaml
```

**Charakterystyka:**
- Debug wyłączony
- PHP ukrywa błędy (`display_errors = Off`)
- MySQL tylko localhost (`bind-address = 127.0.0.1`)
- Rozszerzone nagłówki bezpieczeństwa

#### Production
```bash
ansible-playbook -i inventory.ini -e "env=prod" playbook.yaml
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

### Pre-tasks (zadania wstępne)

Wykonywane **przed** rolami:

1. **Walidacja środowiska** – sprawdza, czy podana wartość `env` jest poprawna (dev/staging/prod)
2. **Ładowanie zmiennych** – wczytuje odpowiedni plik z `group_vars/` na podstawie parametru `env`

**Dlaczego to jest ważne?**
- Zapobiega uruchomieniu playbooka z błędną konfiguracją
- Pozwala na dynamiczne przełączanie między środowiskami bez edycji plików

### Tagi (tags)

Każda rola ma przypisany tag, co pozwala na selektywne uruchamianie:

```bash
# Tylko rola web
ansible-playbook -i inventory.ini -e "env=dev" playbook.yaml --tags web

# Tylko role web i php
ansible-playbook -i inventory.ini -e "env=dev" playbook.yaml --tags web,php

# Wszystko oprócz testów
ansible-playbook -i inventory.ini -e "env=dev" playbook.yaml --skip-tags tests
```

**Dostępne tagi:**
- `base` – rola base
- `web` – rola web
- `php` – rola php
- `database` – rola database
- `app` – rola app
- `tests` – testy w post_tasks
- `always` – walidacja środowiska (zawsze wykonywana)

**Dlaczego używamy tagów?**
- Przyspieszenie developmentu (nie trzeba uruchamiać całego playbooka)
- Możliwość naprawy pojedynczej roli bez wpływu na resztę
- Łatwiejsze testowanie zmian

### Post-tasks (zadania końcowe)

Wykonywane **po** wszystkich rolach:

1. **Test zgodności debug** – sprawdza, czy `app_debug` jest włączony tylko w dev
2. **Test PHP display_errors** – sprawdza, czy ustawienia PHP są zgodne z polityką środowiska
3. **Test portu HTTP** – sprawdza, czy Apache nasłuchuje na oczekiwanym porcie

**Dlaczego to jest ważne?**
- Automatyczna weryfikacja poprawności konfiguracji
- Wykrywa błędy przed wdrożeniem na produkcję
- Dokumentuje oczekiwane zachowanie systemu

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

1. Utwórz plik `group_vars/test.yaml`
2. Dodaj `test` do listy `allowed_envs` w `playbook.yaml`
3. Dodaj zadanie `include_vars` w `pre_tasks`

### Zabezpieczenie haseł (Ansible Vault)

```bash
# Zaszyfruj plik z hasłami
ansible-vault encrypt inventory.ini

# Uruchom playbook z hasłem do vault
ansible-playbook -i inventory.ini -e "env=prod" playbook.yaml --ask-vault-pass
```

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
ansible-playbook -i inventory.ini -e "env=dev" playbook.yaml
# PLAY RECAP: changed=15

# Drugie uruchomienie - brak zmian
ansible-playbook -i inventory.ini -e "env=dev" playbook.yaml
# PLAY RECAP: changed=0
```

## Podsumowanie

Ten projekt demonstruje:
- ✅ Organizację kodu Ansible w role
- ✅ Zarządzanie wieloma środowiskami (dev/staging/prod)
- ✅ Używanie zmiennych i szablonów Jinja2
- ✅ Handlery do restartowania usług
- ✅ Tagi do selektywnego uruchamiania zadań
- ✅ Automatyczne testy konfiguracji (assert)
- ✅ Idempotencję i bezpieczne wielokrotne uruchamianie

Jest to minimalna, ale kompletna implementacja stosu LAMP z obsługą różnych środowisk, gotowa do rozbudowy o dodatkowe funkcje.
