# Minimal LAMP Demo (Ansible)

Ten projekt wdraża najprostsze możliwe środowisko LAMP (Linux + Apache + MySQL + PHP) na hoście z Ubuntu. Całość została rozbita na pięć ról Ansible, aby łatwo pokazać podstawy pracy z rolami.

## Struktura

- `playbook.yaml` – uruchamia role w kolejności `base → web → php → database → app`, równocześnie przekazując nazwy bazy/użytkownika/hasła w sekcji `vars`.
- `inventory.ini` – definiuje grupę `web_server`; docelowy host ma ustawione `ansible_user`, `ansible_port`, `ansible_password` i `ansible_become_password`, dlatego nie trzeba podawać `-k/-K` przy uruchamianiu playbooka.
- `requirements.yaml` – zawiera jedyną wymaganą kolekcję (`community.mysql`).
- `roles/base` – minimalne pakiety i MOTD (prosty przykład podstawowej konfiguracji systemu).
- `roles/web` – instalacja Apache, utworzenie DocumentRoot i wdrożenie prostego VirtualHosta.
- `roles/php` – instalacja PHP 8.3 + modułu mysql, prosty override `php.ini`.
- `roles/database` – instalacja MySQL, własny `mysqld.cnf`, utworzenie bazy/użytkownika/tabeli oraz przykładowej wiadomości.
- `roles/app` – pojedynczy `index.php` łączący się z bazą i wypisujący rekordy.

## Wymagania wstępne

1. Kontroler z Ansible 2.15+ (projekt testowany z ansible-core 2.17).
2. Na docelowym hoście użytkownik z prawami sudo (np. `devops`).
3. Połączenie SSH działające na niestandardowym porcie 7655 (jak w przykładzie).
4. Zainstalowana kolekcja z `requirements.yaml` (pobierana przez `ansible-galaxy`, bo moduły `community.mysql.*` nie są częścią ansible-core):

   ```bash
   ansible-galaxy collection install -r requirements.yaml
   ```

## Uruchomienie

1. Zaktualizuj `inventory.ini`, aby wskazywał poprawny adres/port oraz poświadczenia hosta docelowego.
2. (Opcjonalnie) zmień w `playbook.yaml` wartości `database_name`, `database_user`, `database_password`, jeśli potrzebujesz innych danych logowania.
3. Uruchom playbook:

   ```bash
   ansible-playbook -i inventory.ini playbook.yaml
   ```

   Hasła pobiorą się z inventory, więc nie trzeba dodawać `-k` / `-K`.
4. Po zakończeniu otwórz w przeglądarce `http://<IP_SERWERA>/` – zobaczysz prostą stronę PHP z listą rekordów z MySQL.

## Dostosowanie

- Zmiana portu HTTP: edytuj `web_listen_port` w `roles/web/defaults/main.yaml` i zaktualizuj szablon `vhost.conf.j2`.
- Inna ścieżka DocumentRoot lub inne poświadczenia bazy: wystarczy nadpisać zmienne w `group_vars/` albo w samym playbooku.
- Hasła można przenieść do Ansible Vault, jeśli repozytorium ma trafić do innych osób.

## Najczęstsze problemy

1. **Apache nie startuje** – sprawdź, czy port 80 nie jest zajęty (np. przez nginx) i ew. zatrzymaj konkurencyjną usługę.
2. **`dpkg returned an error code (1)`** – oznacza uszkodzone pakiety w systemie; należy najpierw uruchomić `sudo apt-get -f install` na hoście docelowym.
3. **`Host key verification failed`** – usuń stary wpis w `~/.ssh/known_hosts`: `ssh-keygen -R '[HOST]:PORT'`.

## Podgląd działania

![Podgląd aplikacji](web-page.png)

Po wykonaniu powyższych kroków playbook jest idempotentny i można uruchamiać go wielokrotnie, aby w prosty sposób pokazać działanie podstawowego środowiska LAMP oraz ról Ansible.
