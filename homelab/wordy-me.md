# Ghid Tehnic: Instalare WordyMe pe Proxmox (LXC + Docker)

Acest document conține pașii exacți parcurși pentru instalarea și configurarea WordyMe întrun container LXC Ubuntu 24.04 pe Proxmox.

---

## 💡 Contextul Arhitecturii: Local vs. Server (LXC)
Este important de înțeles că WordyMe (ca multe alte aplicații de pe GitHub) este configurată implicit pentru a rula pe un **PC personal (localhost)**. 

### Problema de bază:
Când o aplicație rulează pe `localhost`, browserul și serverul sunt pe același aparat. Când o mutăm pe un **Server/LXC**, browserul tău (pe laptop) trebuie să comunice cu un aparat la distanță prin rețea. Fără modificări, aplicația va încerca să se caute pe ea însăși pe laptopul tău și va eșua.

### Ce am modificat pentru a funcționa pe LXC:
1.  **Maparea Porturilor:** Am deschis manual portul `3000` în Docker Compose. Implicit, backend-ul era izolat; acum este accesibil din rețea.
2.  **Identitatea Serverului (`BETTER_AUTH_URL`):** I-am spus backend-ului adresa sa reală de IP. Fără asta, el nu știe să valideze sesiunile de logare venite de la distanță.
3.  **Busola Browserului (`VITE_BACKEND_URL`):** Am instruit interfața web (care rulează în browserul tău) să caute datele la IP-ul serverului (`10.10.1.71`), nu pe `localhost`.
4.  **Reconstrucția (`--build`):** Am forțat Docker să re-asambleze aplicația pentru a „coace” aceste adrese de IP direct în codul sursă.

---

## Pasul 1: Pregătirea Proxmox LXC
Am creat un container LXC ultra-eficient pentru a găzdui aplicația.

**Configurație:**
- Template: `ubuntu-24.04-standard`
- Resurse: 1 CPU, 1GB RAM, 10GB Disk
- Rețea: IP Static `10.10.1.71`
- **Acțiune Critică:** Am activat `Options -> Features -> Nesting`. 
*De ce?* Permite Docker să gestioneze straturile de fișiere în interiorul containerului Proxmox.

---

## Pasul 2: Instalare Docker
Am folosit scriptul oficial Docker pentru o instalare rapidă și corectă pe Ubuntu.

**Comenzi:**
```bash
# Descarcare si rulare script oficial
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh ./get-docker.sh
```
*De ce?* Docker izolează aplicația (Frontend și Backend) de sistemul de operare principal, asigurând portabilitatea.

---

## Pasul 3: Descărcare WordyMe
Am adus codul sursă direct de pe GitHub.

**Comenzi:**
```bash
# Instalare git si clonare repo
apt install git -y
git clone https://github.com/TeamCoderz/WordyMe.git
cd WordyMe
```

---

## Pasul 4: Configurare Variabile de Mediu (.env)
Configurăm aplicația să recunoască adresa serverului Proxmox.

**Comenzi:**
```bash
# Creare fisier din modelul oferit
cp .env.example .env

# Generare Secret de Securitate
openssl rand -hex 32

# Editare manuala (nano .env)
# Setari obligatorii:
BETTER_AUTH_SECRET=[Codul de 64 caractere generat anterior]
CLIENT_URL=http://10.10.1.71:5173
VITE_BACKEND_URL=http://10.10.1.71:3000
```

---

## Pasul 5: Personalizare Docker Compose (Rezolvare Login Loop)
Am adaptat planul de lansare (`docker-compose.yml`) pentru a deschide accesul către API.

**Modificări în `docker-compose.yml` (secțiunea backend):**
```yaml
backend:
  ports:
    - "3000:3000"
  environment:
    - CLIENT_URL=http://10.10.1.71:5173
    - BETTER_AUTH_URL=http://10.10.1.71:3000
```

---

## Pasul 6: Lansare și Reconstrucție
Am pornit containerele și am forțat aplicarea noilor setări.

**Comandă:**
```bash
# Pornire cu reconstructie imagini
docker compose up -d --build

# Verificare status
docker ps
```

---

## Rezultat Final
- **Interfață Web:** `http://10.10.1.71:5173`
- **Backend API:** `http://10.10.1.71:3000`
- **Status:** Aplicația este complet funcțională și accesibilă din browserul local.

---

## 🚀 Alternativa: Instalare Rapidă pe PC (Quick Start)
Dacă vrei să testezi aplicația pe propriul laptop (Windows sau Mac), aceasta este **cea mai simplă metodă de a testa aproape orice aplicație de pe GitHub**. 

### 1. Cerințe Prealabile (Instalare Git & Docker)
Dacă nu ai deja uneltele instalate:

*   **Pe Windows:**
    *   **Docker:** Descarcă [Docker Desktop](https://www.docker.com/products/docker-desktop/).
    *   **Git:** Deschide terminalul (CMD/PowerShell) și scrie: `winget install --id Git.Git`.
*   **Pe Mac:**
    *   **Docker:** Descarcă [Docker Desktop](https://www.docker.com/products/docker-desktop/).
    *   **Git:** Deschide terminalul și scrie `git`. Dacă nu e instalat, Mac-ul te va întreba automat dacă vrei să instalezi "Xcode Command Line Tools". Spune "Yes".

### 2. Lansarea în 60 de secunde
După ce ai uneltele, rulează aceste comenzi în folderul unde vrei să stea proiectul:
```bash
git clone https://github.com/TeamCoderz/WordyMe.git
cd WordyMe
docker compose up -d
```
Accesează apoi: `http://localhost:5173`.

---

## 🧹 Cum cureți totul (Dezinstalare completă)
Dacă ai terminat testarea și vrei să ștergi totul de pe PC/Server:

1.  **Oprește și șterge containerele & datele:**
    În folderul proiectului (`WordyMe`), scrie:
    ```bash
    docker compose down -v
    ```
    *Notă: Parametrul `-v` (volumes) șterge și baza de date (notițele). Dacă vrei să le păstrezi, folosește doar `docker compose down`.*

2.  **Șterge folderul cu codul sursă:**
    ```bash
    cd ..
    rm -rf WordyMe
    ```

3.  **Curățare generală Docker (opțional):**
    Pentru a șterge imaginile descărcate și cache-ul rămas:
    ```bash
    docker system prune -a
    ```

4.  **Dezinstalare Docker (Dacă nu mai ai nevoie de el):**
    *   **Windows/Mac:** Uninstall "Docker Desktop" din meniul de Apps.
    *   **LXC (Linux):** `sudo apt-get purge docker-ce docker-ce-cli containerd.io` urmat de `sudo rm -rf /var/lib/docker`.

---

## ✅ Rezultat Final
Docker este **cel mai bun mod de a experimenta cu software-ul de pe GitHub** fără să îți „murdărești” sistemul principal cu instalări de baze de date sau limbaje de programare. Totul stă în „containere” pe care le poți șterge instantaneu, exact cum am făcut mai sus.

---


## Resurse suplimentare

- [Repo oficial WordyMe](https://github.com/teamcoderz/wordyme)
- [Documentație Docker](https://docs.docker.com)
- [Proxmox LXC — documentație](https://pve.proxmox.com/wiki/Linux_Container)

---

_Ultima actualizare: 2026-03-29_